# AGENTS.md — คู่มือสำหรับ AI Agent (Vibe Coding)

> ไฟล์นี้คือ "กติกาการทำงาน" ที่ AI ต้องอ่านก่อนแก้โค้ดในโปรเจกต์นี้
> ใช้สอน Vibe Coding: ให้นักเรียนป้อนเอกสารชุดนี้ให้ AI แล้วสร้างระบบขึ้นใหม่ทีละขั้น
> อ่านคู่กับ [prd.md](prd.md) (ทำอะไร), [architecture.md](architecture.md) (โครงสร้าง), [implementation-plan.md](implementation-plan.md) (ลำดับสร้าง), [schema.md](schema.md) (ฐานข้อมูล), [progress.md](progress.md) (ความคืบหน้า)

---

## 1. ภาพรวมโปรเจกต์ (อ่านก่อนเสมอ)

ระบบจองห้องประชุม **Next.js 14 (App Router) + TypeScript + Tailwind + SQLite**
- หัวใจคือ **กันการจองชนกัน (double booking) ระดับ DB transaction**
- เก็บข้อมูลใน SQLite ไฟล์เดียว `data/app.db` ผ่าน `better-sqlite3` (synchronous)
- Auth เป็น signed-cookie session เอง (HMAC) ไม่ใช้ library ภายนอก
- UI สไตล์ Clean & Modern: sidebar + พื้นขาว + accent ส้ม, ใช้ primitive กลางทั้งหมด

---

## 2. คำสั่งที่ใช้บ่อย (Commands)

```bash
npm install          # ติดตั้ง dependencies
npm run db:migrate   # สร้าง/อัปเดต schema → data/app.db (idempotent)
npm run db:seed      # ใส่ข้อมูลตัวอย่าง (ล้างของเดิมก่อน)
npm run db:reset     # ลบ db แล้ว migrate + seed ใหม่
npm run dev          # dev server (http://localhost:3000)
npm run build        # production build
npm run lint         # ESLint
npx tsc --noEmit     # typecheck อย่างเดียว
```

**บัญชีทดลอง (จาก seed):** `jira / password123` (admin) · `staff / password123` (user)

---

## 3. กฎเหล็ก (Critical Rules) — ห้ามพลาด

1. **❌ ห้ามรัน `npm run build` ขณะ `npm run dev` กำลังรัน** — ทั้งคู่เขียนทับโฟลเดอร์ `.next` เดียวกัน จะทำให้ dev พังด้วย error `Cannot find module './xxx.js'`
   - ถ้าเจอ error นี้: ปิด dev → `rm -rf .next` → `npm run dev` ใหม่
2. **✅ ทุกการตรวจสิทธิ์ต้องทำฝั่ง server** — ห้ามเชื่อ client; ใช้ `requirePageUser()` / `requirePageAdmin()` ใน page/layout และเช็ค `getApiSession()` + role ใน API route
3. **✅ การจองต้องผ่าน `createBookingTx`** เท่านั้น — ห้ามเขียน INSERT booking ตรง ๆ เพราะจะข้ามการกันชน
4. **✅ เขียน DB ผ่าน data layer (`lib/db/*`) เท่านั้น** — ห้าม query SQL จาก client หรือจาก component โดยตรง
5. **✅ Validate ทุก input ด้วย Zod** ทั้ง client และ server ก่อนแตะ DB
6. **✅ ใช้ prepared statement / parameter binding เสมอ** — ห้าม string-concat ค่าเข้า SQL
7. **✅ SQLite ต้องเป็น single process** — อย่าออกแบบให้รันหลาย instance; in-memory session ก็เช่นกัน

---

## 4. โครงสร้างไฟล์ & ที่อยู่ของสิ่งต่าง ๆ

```
app/
  (auth)/login/         # หน้า login (ไม่ต้อง auth)
  (app)/                # หน้าที่ต้อง login — layout เรียก requirePageUser()
    rooms/ rooms/[id]/ calendar/ my-bookings/
  (admin)/              # หน้าเฉพาะ admin — layout เรียก requirePageAdmin()
    admin/rooms/ admin/users/
  api/                  # API routes (auth, rooms, bookings, users, images)
lib/
  db/                   # data access layer — connection, migrate, seed, repositories
  auth/                 # session (HMAC cookie), password (bcrypt), guard
  booking/              # overlap logic, slot generation/validation
  validation/schemas.ts # Zod schemas (ใช้ร่วม client+server)
  constants.ts          # ค่าคงที่ระบบ (timezone, slot, เวลาทำการ ฯลฯ)
  types.ts              # TypeScript types กลาง
components/
  ui/                   # primitives: Button, Badge, Card, Input, Alert, Modal, PageHeader
  app-shell/            # Sidebar + AppShell (layout)
  room/ booking/ calendar/   # feature components
data/app.db             # SQLite (สร้างจาก migrate; อยู่ใน .gitignore)
public/uploads/rooms/   # ไฟล์ภาพ gallery
```

**กฎการวางไฟล์:**
- Logic ฐานข้อมูล → `lib/db/<entity>.ts` (1 ไฟล์ต่อ 1 ตาราง)
- Business rule (overlap, slot) → `lib/booking/`
- UI ที่ใช้ซ้ำ → `components/ui/` (อย่าสร้าง variant ใหม่ถ้า primitive เดิมพอ)
- Component เฉพาะหน้า → วางข้างไฟล์ `page.tsx` ของหน้านั้น (เช่น `AdminUsers.tsx`)

---

## 5. ขนบการเขียนโค้ด (Conventions)

### TypeScript / React
- `strict: true`; หลีกเลี่ยง `any`
- **Server Component เป็นค่าตั้งต้น**; ใส่ `'use client'` เฉพาะเมื่อต้องใช้ state/event (ฟอร์ม, modal, interactive)
- หน้าใดอ่าน DB ให้ตั้ง `export const dynamic = 'force-dynamic'`
- Data fetching ในหน้า server → เรียก repository ตรง (เช่น `listRooms()`); ใน client → `fetch('/api/...')`

### Naming
- ตาราง/คอลัมน์ DB เป็น `snake_case`; โค้ด TS เป็น `camelCase` — แปลงในชั้น repository (mapper)
- id ใช้ prefix: `u_` (user), `r_` (room), `b_` (booking), `img_`

### UI
- ใช้ **design tokens** ผ่าน Tailwind เท่านั้น: `bg-card`, `text-fg`, `text-fg-muted`, `border-border`, `bg-brand`, `text-brand-fg`, `bg-surface` + semantic `text-success/danger/warning`
- **ห้าม hardcode สี** (เช่น `text-red-500`) — ใช้ semantic token; ข้อความ error/success ใช้ `<Alert tone="danger|success">`
- การ์ด/ปุ่ม/popup รัศมี **8px** (`rounded-lg`); badge เป็น `rounded-full`
- สถานะทุกชนิดใช้ `<Badge tone="...">`; หัวข้อหน้าใช้ `<PageHeader>`

### API route แพตเทิร์นมาตรฐาน
```ts
export async function POST(req: Request) {
  const session = getApiSession();
  if (!session) return NextResponse.json({ error: 'unauthorized' }, { status: 401 });
  // ถ้าเป็น admin-only: if (session.role !== 'admin') → 403
  const body = await req.json().catch(() => null);
  const parsed = someSchema.safeParse(body);
  if (!parsed.success) return NextResponse.json({ error: parsed.error.issues[0]?.message }, { status: 400 });
  // ...เรียก repository, จับ error เฉพาะ (เช่น ConflictError → 409)
  return NextResponse.json({ ok: true }, { status: 201 });
}
```

### HTTP status ที่ใช้
`200` ok · `201` created · `400` validation · `401` ไม่ได้ login · `403` ไม่มีสิทธิ์ · `404` ไม่พบ · `409` conflict (จองชน/username ซ้ำ)

---

## 6. แพตเทิร์นสำคัญที่ต้องเลียนแบบ

### การกันจองชนกัน (อย่าทำผิด!)
```ts
// lib/db/bookings.ts — ห่อ "ตรวจ overlap → INSERT" ใน transaction เดียว
export const createBookingTx = db.transaction((b) => {
  const conflict = db.prepare(
    `SELECT COUNT(*) AS n FROM bookings
      WHERE room_id = @roomId AND start_at < @endAt AND @startAt < end_at`
  ).get(b);
  if (conflict.n > 0) throw new ConflictError();   // → API คืน 409
  db.prepare(`INSERT INTO bookings (...) VALUES (...)`).run(b);
  return { ...b };
});
```
`db.transaction()` ของ better-sqlite3 ใช้ `BEGIN IMMEDIATE` (write-lock ทันที) → atomic, serialize การจองพร้อมกัน

### Auth guard
- `getVerifiedSession()` อ่าน cookie **แล้ว verify กับ DB** (user ยังมี + active) และคืน role ล่าสุด → บัญชีที่ถูกปิดถูกตัดทันที
- Page/layout: `requirePageUser()` / `requirePageAdmin()` (redirect ถ้าไม่ผ่าน)
- API: `getApiSession()` (คืน null → route จัดการ 401/403 เอง)

### Repository mapper
อ่าน row จาก DB (snake_case, boolean เป็น 0/1, amenities เป็น JSON string) แล้ว map เป็น TS object (camelCase, boolean จริง, array) ก่อนส่งออกเสมอ

---

## 7. เวลาทำงานต้องทำ (Workflow)

1. อ่าน [implementation-plan.md](implementation-plan.md) หา milestone/ขั้นที่กำลังทำ
2. เขียนโค้ดให้ตรง convention ข้างบน
3. **ตรวจงานจริง** — อย่าเดาว่าใช้ได้:
   - `npx tsc --noEmit` (typecheck ต้องผ่าน)
   - ถ้าแตะ booking/auth/api → ทดสอบด้วย `curl` ยิง endpoint จริง (ดูตัวอย่างใน implementation-plan)
   - ถ้าแตะ schema → `npm run db:migrate` แล้วตรวจ `PRAGMA table_info(...)`
4. อัปเดต [progress.md](progress.md) เมื่อปิดงานแต่ละชิ้น
5. **อย่า build ขณะ dev รัน** (กฎข้อ 3.1)

---

## 8. กับดักที่เจอบ่อย (Gotchas)

| อาการ | สาเหตุ | แก้ |
|---|---|---|
| `Cannot find module './276.js'` | `.next` cache เพี้ยน (มัก build ทับ dev) | ปิด dev, `rm -rf .next`, dev ใหม่ |
| อัปโหลดภาพแล้วไม่เห็นเปลี่ยน | ลืม `router.refresh()` หลัง mutate | เรียก refresh + reload data |
| จองชนกันหลุด | เขียน INSERT นอก transaction | ใช้ `createBookingTx` เท่านั้น |
| Port 3000 ไม่ว่าง | มี process อื่นใช้ | Next ขยับไป 3001 อัตโนมัติ (ปกติ) |
| `better-sqlite3` build fail | ไม่มี build tools | ติดตั้ง Xcode CLT (mac) / build-essential (linux) |
| สีไม่สม่ำเสมอ | hardcode สีแทน token | ใช้ semantic token + `<Alert>` / `<Badge>` |

---

## 9. สิ่งที่ห้ามทำ (Do NOT)

- ❌ เขียน SQL หรืออ่านไฟล์ JSON/DB จากฝั่ง client
- ❌ เชื่อ role/identity ที่ส่งมาจาก client โดยไม่ verify ฝั่ง server
- ❌ ข้าม Zod validation
- ❌ สร้างฐานข้อมูล/ตารางใหม่นอก `lib/db/migrate.ts`
- ❌ hardcode ค่าคงที่ (เวลาทำการ, slot) — ใช้ `lib/constants.ts`
- ❌ เพิ่ม dependency ใหม่โดยไม่จำเป็น (โปรเจกต์นี้ตั้งใจ dependency น้อย: next, react, better-sqlite3, bcryptjs, zod, tailwind)
