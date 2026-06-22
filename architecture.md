# Architecture — Smart Meeting Room Booking

> สถาปัตยกรรมระบบเชิงลึก สำหรับสอน Vibe Coding — อธิบายว่าแต่ละชั้นทำงานร่วมกันอย่างไร, ข้อมูลไหลอย่างไร, และทำไมจึงออกแบบแบบนี้
> อ่านคู่กับ [prd.md](prd.md), [schema.md](schema.md), [AGENTS.md](AGENTS.md)

---

## 1. ภาพรวม (High-Level)

```
┌─────────────────────────────────────────────────────────────┐
│  Browser (Client)                                           │
│  - Server Components (render ฝั่ง server, ไม่มี JS)          │
│  - Client Components ('use client') : ฟอร์ม, modal, ปฏิทิน  │
└───────────────┬─────────────────────────────────────────────┘
                │ HTTP (fetch / form submit / navigation)
┌───────────────▼─────────────────────────────────────────────┐
│  Next.js 14 App Router (Single Node Process)                │
│                                                             │
│  ┌─────────────┐   ┌──────────────┐   ┌──────────────────┐  │
│  │ Pages       │   │ API Routes   │   │ Auth Guard       │  │
│  │ (RSC)       │   │ /api/*       │   │ (server-side)    │  │
│  └──────┬──────┘   └──────┬───────┘   └────────┬─────────┘  │
│         │                 │                    │            │
│  ┌──────▼─────────────────▼────────────────────▼─────────┐  │
│  │  Business / Data Access Layer (lib/)                  │  │
│  │  db/* (repositories) · booking/* · auth/* · validation│  │
│  └──────────────────────────┬────────────────────────────┘  │
└─────────────────────────────┼───────────────────────────────┘
                              │ better-sqlite3 (synchronous)
                  ┌───────────▼────────────┐   ┌──────────────┐
                  │  SQLite  data/app.db    │   │ public/      │
                  │  (WAL, FK on)           │   │ uploads/     │
                  └─────────────────────────┘   └──────────────┘
```

**หลักการแยกชั้น:** UI ไม่รู้จัก SQL เลย — คุยกับ data layer (`lib/db/*`) ผ่านฟังก์ชันเท่านั้น ทำให้เปลี่ยน SQLite → Postgres ได้โดยแก้แค่ชั้น db

---

## 2. ชั้นต่าง ๆ (Layers)

### 2.1 Presentation (app/, components/)
- **Server Components (ค่าตั้งต้น)** — page ส่วนใหญ่อ่าน DB ตรงผ่าน repository แล้ว render เป็น HTML (ไม่มี JS ไป client) เช่น `rooms/page.tsx` เรียก `listRooms()`
- **Client Components (`'use client'`)** — เฉพาะที่ต้อง interactive: `BookingForm`, `CalendarView`, `AdminUsers`, `AdminRooms`, `ImageManager`, `Modal`, `Sidebar`, `LoginForm` — คุย backend ผ่าน `fetch('/api/...')`
- **Design system** — `components/ui/*` เป็น primitive ชุดเดียวที่ทุกหน้าใช้ (ดู §6)

### 2.2 Routing & Guards (app/ route groups)
ใช้ **route groups** แยกระดับสิทธิ์ โดยให้ layout เป็นด่านตรวจ:
- `(auth)/` — ไม่ต้อง login (หน้า login)
- `(app)/layout.tsx` — เรียก `requirePageUser()` → ทุกหน้าใต้นี้ต้อง login
- `(admin)/layout.tsx` — เรียก `requirePageAdmin()` → เฉพาะ admin

> guard อยู่ที่ layout = เขียนครั้งเดียว คุมทุกหน้าใต้กลุ่ม ไม่ต้องเช็คซ้ำในแต่ละ page

### 2.3 API Layer (app/api/)
REST endpoints สำหรับ mutation และ data ที่ client ต้องการ:
- `auth/login`, `auth/logout`
- `rooms` (GET/POST), `rooms/[id]` (GET/PUT), `rooms/[id]/images` (POST), `rooms/[id]/images/[imageId]` (DELETE/PATCH)
- `bookings` (GET/POST), `bookings/[id]` (DELETE)
- `users` (GET/POST), `users/[id]` (PUT/DELETE), `users/[id]/password` (POST)

ทุก route: ตรวจ session → validate (Zod) → เรียก repository → คืน JSON + status code

### 2.4 Business Logic (lib/booking/, lib/auth/)
- `booking/overlap.ts` — สูตร overlap
- `booking/slots.ts` — สร้าง slot 30 นาที, ตรวจเวลาทำการ/window/duration, แปลงเวลา ISO
- `auth/session.ts` — เซ็น/ถอด signed cookie (HMAC-SHA256)
- `auth/password.ts` — bcrypt hash/verify
- `auth/guard.ts` — verify session กับ DB + helper สำหรับ page/api

### 2.5 Data Access (lib/db/)
- `connection.ts` — เปิด SQLite (singleton กัน hot-reload สร้างซ้ำ), ตั้ง PRAGMA (WAL, foreign_keys, busy_timeout)
- `migrate.ts` — สร้าง schema (idempotent) + additive migration
- `seed.ts` — ข้อมูลตัวอย่าง + ภาพ placeholder
- `users.ts` / `rooms.ts` / `bookings.ts` — repository (CRUD + business query) พร้อม mapper snake_case↔camelCase

### 2.6 Persistence
- **SQLite** `data/app.db` — WAL mode (อ่าน-เขียนพร้อมกันดีขึ้น), foreign_keys on
- **ไฟล์ภาพ** `public/uploads/rooms/<roomId>/` — Next serve เป็น static; DB เก็บแค่ path

---

## 3. Flow สำคัญ (Sequence)

### 3.1 Login
```
LoginForm (client) ──POST /api/auth/login──▶ route
  route: validate(Zod) ▶ findByUsername ▶ verifyPassword(bcrypt)
       ▶ ถ้า isActive=false → 403
       ▶ setSession(): เซ็น {userId, role, displayName} เป็น cookie (HMAC)
  ◀── 200 {ok, role}
LoginForm ▶ router.push('/rooms')
```

### 3.2 ป้องกันการเข้าถึง (ทุก request หน้า (app)/(admin))
```
เปิดหน้า ▶ layout เรียก requirePageUser/Admin
  ▶ getVerifiedSession(): อ่าน cookie ▶ ตรวจลายเซ็น ▶ findById(userId)
       ▶ ถ้า user ไม่มี/ถูกปิด → return null → redirect('/login')
       ▶ คืน role ล่าสุดจาก DB (กัน role ค้างใน cookie)
```

### 3.3 การจอง + กันชน (หัวใจของระบบ)
```
BookingForm (client)
  - โหลด bookings ของห้อง+วัน ▶ คำนวณ slot ที่ว่าง ▶ disable slot ที่ถูกจอง (กันชนชั้น UX)
  - submit ──POST /api/bookings──▶ route
      route: getApiSession (401 ถ้าไม่ login)
           ▶ validate(Zod)
           ▶ validateBookingTime(): เวลาทำการ/window 30 วัน/duration 30นาที–4ชม.
           ▶ createBookingTx(...)  ◀── หัวใจ:
              BEGIN IMMEDIATE
                SELECT COUNT(*) WHERE room_id=? AND start_at<@end AND @start<end_at
                ถ้า > 0 → throw ConflictError (rollback)
                ELSE INSERT
              COMMIT
           ▶ สำเร็จ → 201 | ConflictError → 409
  - ถ้า 409 (race) ▶ แจ้งผู้ใช้ + reload slot ที่ว่างใหม่
```
> การกันชน **ชั้นจริงอยู่ที่ DB transaction** ส่วน UI disable slot เป็นแค่ UX — สอง request ที่ส่งพร้อมกันถูก serialize โดย SQLite write-lock จึงมีแค่ 1 ที่ผ่าน

### 3.4 อัปโหลด/จัดการภาพ
```
ImageManager (client) ──POST /api/rooms/[id]/images (multipart)──▶ route
  route: admin guard ▶ ตรวจชนิด/ขนาด (≤5MB) ▶ เขียนไฟล์ public/uploads ▶ addImage(path)
DELETE .../images/[imageId] ▶ ลบ row + unlink ไฟล์ + ถ้าเป็นปก เลื่อนปกไปภาพถัดไป
PATCH  .../images/[imageId] ▶ setCover(roomId, path)
```

---

## 4. Auth & Session Model

- **ไม่มี DB ตาราง session** — state ทั้งหมดอยู่ใน cookie ที่เซ็นด้วย `SESSION_SECRET`
- cookie ชื่อ `mr_session` = `base64url(payload) . HMAC-SHA256(payload)`
- `httpOnly`, `sameSite=lax`, `secure` (prod), อายุ 7 วัน
- ตอนใช้งานจริงทุกครั้ง **verify กับ DB** (`getVerifiedSession`) เพื่อรองรับการปิดบัญชี/เปลี่ยน role ทันที
- เลือกวิธีนี้เพราะ: dependency น้อย, เข้าใจง่ายสำหรับสอน, พอเพียงสำหรับ single-process MVP
- **ข้อแลกเปลี่ยน:** ใช้ได้ดีกับ process เดียว; ถ้าจะ scale หลาย instance ควรเปลี่ยนเป็น session store/JWT ที่ออกแบบมาเพื่อการนั้น

---

## 5. Data Access Pattern

ทุก repository ทำหน้าที่ 3 อย่าง:
1. **ซ่อน SQL** — UI/route เรียกฟังก์ชัน เช่น `listRooms()`, `createBookingTx()`
2. **Map ชนิดข้อมูล** — DB เก็บ boolean เป็น 0/1, amenities เป็น JSON string, เวลาเป็น ISO text → repository แปลงเป็น TS object ที่สะอาด
3. **บรรจุ business query** — เช่น overlap check, last-admin count, user-has-bookings

ผลคือ: **เปลี่ยน DB engine ได้โดยแก้แค่ `lib/db/`** — UI/API ไม่ต้องแตะ

---

## 6. Design System (UI Architecture)

```
Tokens (CSS variables @ globals.css)
  --bg --card --surface --border --fg(.muted .subtle)
  --brand(.hover .subtle .fg)  --success/-bg --danger/-bg --warning/-bg
        │  map ผ่าน tailwind.config.ts
        ▼
Primitives (components/ui/)
  Button · Badge · Card · Input/Select/Field · Alert · Modal · PageHeader
        │  ประกอบเป็น
        ▼
Feature components (room/ booking/ calendar/ app-shell/)
        │
        ▼
Pages (app/)
```

**กฎ:** ทุกสี/รัศมี/สถานะมาจาก token + primitive ชุดเดียว → consistency โดยอัตโนมัติ
- รัศมีการ์ด/ปุ่ม/popup = 8px (`rounded-lg`)
- error/success ใช้ `<Alert>` เท่านั้น; สถานะใช้ `<Badge tone>`; หัวข้อใช้ `<PageHeader>`
- Layout: `AppShell` = `Sidebar` (เดสก์ท็อป fixed / มือถือ drawer) + พื้นที่เนื้อหา

---

## 7. ข้อจำกัดการ Deploy (สำคัญ)

ระบบนี้ออกแบบให้รันเป็น **single Node process บนเครื่องที่มี persistent disk**:
- SQLite write-lock + in-memory connection singleton → ต้องเป็น process เดียว
- session อยู่ใน cookie (stateless) แต่ guard อ่าน DB ทุก request
- ไฟล์ภาพเขียนลง `public/uploads` → ต้องมี disk เขียนได้

**ไม่เหมาะ** serverless แบบ multi-instance / read-only FS (เช่น Vercel default) — ควร deploy บน VM/คอนเทนเนอร์เดียว (เช่น Render, Railway, VPS)

---

## 8. การตัดสินใจเชิงสถาปัตยกรรม & เหตุผล

| การตัดสินใจ | เหตุผล |
|---|---|
| SQLite แทน JSON ไฟล์ | ได้ transaction/constraint มาฟรี → กันชนง่ายและถูกต้องกว่า |
| `better-sqlite3` (sync) | API ตรงไปตรงมา, ใช้ใน RSC ได้ไม่ต้อง await, transaction เขียนง่าย |
| Signed cookie session เอง | dependency น้อย, โปร่งใส เหมาะสอน; เพียงพอสำหรับ single-process |
| Route group guards | คุมสิทธิ์ที่ layout เดียว ครอบทุกหน้าใต้กลุ่ม |
| Data layer แยกชัด | เปลี่ยน DB ภายหลังได้โดยไม่กระทบ UI |
| Design tokens + primitives | กัน UI ไม่สอดคล้องตั้งแต่ราก |
| Overlap check ใน transaction | atomic จริง ไม่ต้องพึ่ง mutex ที่เปราะ |
