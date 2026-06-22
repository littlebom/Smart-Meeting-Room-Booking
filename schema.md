# Database Schema — Smart Meeting Room Booking

> สคีมาฐานข้อมูล SQLite ฉบับเต็ม สำหรับสอน Vibe Coding — ใช้สร้าง `lib/db/migrate.ts` ขึ้นใหม่ให้ตรงต้นฉบับ
> Engine: **SQLite** (`better-sqlite3`) · ไฟล์: `data/app.db` · PRAGMA: `journal_mode=WAL`, `foreign_keys=ON`, `busy_timeout=5000`

---

## 1. ER Diagram (ความสัมพันธ์)

```
users ──1───────< bookings >───────1── rooms
  id              user_id   room_id      id
                                          │
                                          1
                                          │
                                          <
                                     room_images
                                       room_id
```

- `users 1—N bookings` (ผู้ใช้หนึ่งคนมีได้หลายการจอง)
- `rooms 1—N bookings` (ห้องหนึ่งมีได้หลายการจอง)
- `rooms 1—N room_images` (ห้องหนึ่งมีได้หลายภาพ; ลบห้อง → ลบภาพ cascade)

---

## 2. ตาราง (Tables)

### 2.1 `users`
```sql
CREATE TABLE users (
  id            TEXT PRIMARY KEY,            -- 'u_xxxxxxxx'
  username      TEXT NOT NULL UNIQUE,
  display_name  TEXT NOT NULL,
  password_hash TEXT NOT NULL,               -- bcrypt
  role          TEXT NOT NULL DEFAULT 'user'
                CHECK (role IN ('admin','user')),
  is_active     INTEGER NOT NULL DEFAULT 1,  -- 0/1 (boolean)
  created_at    TEXT NOT NULL DEFAULT (datetime('now'))
);
```
| คอลัมน์ | ชนิด | หมายเหตุ |
|---|---|---|
| id | TEXT PK | prefix `u_` |
| username | TEXT UNIQUE | ใช้ login; กันซ้ำระดับ DB |
| display_name | TEXT | ชื่อที่แสดง |
| password_hash | TEXT | bcrypt (cost 10) — ห้ามเก็บ plaintext |
| role | TEXT | `admin` \| `user` (CHECK) |
| is_active | INTEGER | 1=เปิด, 0=ปิด → ปิดแล้ว login ไม่ได้ |
| created_at | TEXT | ISO datetime |

### 2.2 `rooms`
```sql
CREATE TABLE rooms (
  id          TEXT PRIMARY KEY,             -- 'r_xxxxxxxx'
  name        TEXT NOT NULL,
  capacity    INTEGER NOT NULL,
  location    TEXT,
  amenities   TEXT NOT NULL DEFAULT '[]',   -- JSON string: ["projector","tv",...]
  cover_image TEXT,                          -- path ภาพปก (อ้างใน room_images)
  is_active   INTEGER NOT NULL DEFAULT 1,
  created_at  TEXT NOT NULL DEFAULT (datetime('now'))
);
```
| คอลัมน์ | ชนิด | หมายเหตุ |
|---|---|---|
| amenities | TEXT (JSON) | เก็บ array เป็น JSON string; repository `JSON.parse/stringify` |
| cover_image | TEXT NULL | path ภาพปก; ตั้งอัตโนมัติเมื่ออัปโหลดภาพแรก |
| is_active | INTEGER | ห้องปิด → User มองไม่เห็น/จองไม่ได้ |

> **amenity keys:** `projector, tv, whiteboard, video_conference, sound_system, wifi` (นิยามใน `lib/constants.ts`)

### 2.3 `room_images`
```sql
CREATE TABLE room_images (
  id         TEXT PRIMARY KEY,
  room_id    TEXT NOT NULL REFERENCES rooms(id) ON DELETE CASCADE,
  path       TEXT NOT NULL,                  -- '/uploads/rooms/r_001/01.svg'
  sort_order INTEGER NOT NULL DEFAULT 0
);
```
- `ON DELETE CASCADE` — ลบห้องแล้วภาพหายตาม
- `path` เป็น public path (Next serve static); ไฟล์จริงอยู่ `public/uploads/...`

### 2.4 `bookings`
```sql
CREATE TABLE bookings (
  id         TEXT PRIMARY KEY,              -- 'b_xxxxxxxx'
  room_id    TEXT NOT NULL REFERENCES rooms(id),
  user_id    TEXT NOT NULL REFERENCES users(id),
  title      TEXT NOT NULL,
  date       TEXT NOT NULL,                 -- 'YYYY-MM-DD'
  start_at   TEXT NOT NULL,                 -- ISO8601 +07:00
  end_at     TEXT NOT NULL,                 -- ISO8601 +07:00
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  CHECK (start_at < end_at)
);
```
| คอลัมน์ | ชนิด | หมายเหตุ |
|---|---|---|
| date | TEXT | `YYYY-MM-DD` (เขต Asia/Bangkok) — ใช้กรองเร็ว |
| start_at / end_at | TEXT (ISO) | เก็บพร้อม offset `+07:00` รูปแบบเดียวกันทั้งระบบ |
| CHECK | — | กันเวลาเริ่ม ≥ สิ้นสุดระดับ DB |

> **เหตุผลที่ start_at/end_at เป็น TEXT ISO:** ทุกค่า offset เดียวกัน (+07:00) รูปแบบคงที่ → เปรียบเทียบแบบ lexicographic ได้ถูกต้อง (`<`, `>`) ใช้กับ overlap query ตรง ๆ

---

## 3. Indexes
```sql
CREATE INDEX idx_bookings_room_date ON bookings(room_id, date);
CREATE INDEX idx_room_images_room   ON room_images(room_id, sort_order);
```
- `idx_bookings_room_date` — เร่ง query หา overlap/ตารางของห้องตามวัน (query ที่ใช้บ่อยสุด)
- `idx_room_images_room` — เร่งการดึงภาพของห้องตามลำดับ

---

## 4. Query หัวใจ: ตรวจ Overlap

```sql
SELECT COUNT(*) AS n FROM bookings
 WHERE room_id = @roomId
   AND start_at < @endAt      -- existing.start < new.end
   AND @startAt < end_at;     -- new.start < existing.end
```
- เงื่อนไข overlap มาตรฐาน: `existing.start < new.end AND new.start < existing.end`
- ถ้า `n > 0` → ชน → โยน `ConflictError`
- **ต้องอยู่ใน transaction เดียวกับ INSERT** (ดู §5)

---

## 5. Transaction การจอง (Atomicity)

```ts
// db.transaction() ของ better-sqlite3 → ใช้ BEGIN IMMEDIATE (write-lock ทันที)
export const createBookingTx = db.transaction((b) => {
  const { n } = db.prepare(`SELECT COUNT(*) AS n FROM bookings
    WHERE room_id=@roomId AND start_at<@endAt AND @startAt<end_at`).get(b);
  if (n > 0) throw new ConflictError();
  db.prepare(`INSERT INTO bookings (id,room_id,user_id,title,date,start_at,end_at)
    VALUES (@id,@roomId,@userId,@title,@date,@startAt,@endAt)`).run(b);
  return { ...b };
});
```
สอง request พร้อมกันถูก serialize → มีแค่ 1 ที่ INSERT สำเร็จ อีกอันเจอ overlap แล้ว rollback → **double booking = 0**

---

## 6. Migration & Seed

### Migration (idempotent)
- ทุก `CREATE TABLE` / `CREATE INDEX` ใช้ `IF NOT EXISTS` → รันซ้ำได้ปลอดภัย
- **Additive migration** สำหรับ DB เดิม: ตรวจ `PRAGMA table_info(users)` ถ้าไม่มี `is_active` ค่อย `ALTER TABLE users ADD COLUMN is_active INTEGER NOT NULL DEFAULT 1`
  > บทเรียน: เพิ่มคอลัมน์โดยไม่ลบข้อมูลเดิม — รูปแบบ migration จริงที่ใช้บ่อย

### Seed data (`npm run db:seed`)
- ล้างข้อมูลเดิมก่อน (DELETE ทุกตาราง) แล้วใส่ใหม่
- **2 users:** `jira` (admin), `staff` (user) — รหัส `password123` (bcrypt)
- **4 rooms:** ห้องประชุมสภาคณะ, Auditorium, กลุ่มย่อย A, Workshop — แต่ละห้อง 3 ภาพ placeholder (SVG generate ลง `public/uploads`)
- 0 bookings (เริ่มสะอาด)

---

## 7. การแปลงชนิดข้อมูล (Mapper)

DB ↔ TypeScript ไม่ตรงกัน → repository แปลงเสมอ:

| ในฐานข้อมูล | ใน TypeScript |
|---|---|
| `snake_case` (display_name) | `camelCase` (displayName) |
| `is_active` INTEGER 0/1 | `isActive` boolean |
| `amenities` TEXT '["a","b"]' | `amenities` string[] |
| `start_at` TEXT ISO | `startAt` string |

> ดูตัวอย่าง mapper ใน `lib/db/users.ts`, `rooms.ts`, `bookings.ts`

---

## 8. Type สรุป (lib/types.ts)
```ts
type Role = 'admin' | 'user';
interface User   { id; username; displayName; role; isActive }       // ไม่มี passwordHash
interface UserRow extends User { passwordHash }                       // ใช้ภายใน auth เท่านั้น
interface Room   { id; name; capacity; location; amenities[]; coverImage; isActive; images? }
interface RoomImage { id; roomId; path; sortOrder }
interface Booking { id; roomId; userId; title; date; startAt; endAt; createdAt }
interface BookingWithMeta extends Booking { userName; roomName }      // join สำหรับแสดงผล
```
