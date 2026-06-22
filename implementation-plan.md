# Implementation Plan — Smart Meeting Room Booking

> ลำดับการสร้างระบบทีละขั้น สำหรับสอน Vibe Coding — ทำตามจาก M0 → M7 จะได้ระบบเหมือนต้นฉบับ
> แต่ละ milestone มี: งานที่ต้องทำ, ไฟล์ที่สร้าง, และวิธี **ตรวจงานจริง** ก่อนไปต่อ
> อ่านคู่กับ [AGENTS.md](AGENTS.md) (กติกา), [architecture.md](architecture.md), [schema.md](schema.md)

---

## วิธีใช้เอกสารนี้ (สำหรับ Vibe Coding)

1. ป้อน [prd.md](prd.md) + [AGENTS.md](AGENTS.md) + ไฟล์นี้ให้ AI เป็น context
2. สั่งทำ **ทีละ milestone** (อย่ารวบหลายขั้น) — เช่น "ทำ M1 ตาม implementation-plan"
3. หลังแต่ละขั้น ให้ AI **ตรวจงานจริง** ตามช่อง "✅ ตรวจ" ก่อนไปต่อ
4. อัปเดต [progress.md](progress.md) ทุกครั้งที่ปิดงาน

> **กฎทอง:** typecheck ต้องผ่าน + ทดสอบ endpoint ที่แตะจริงด้วย `curl` ก่อนบอกว่าเสร็จ

---

## ค่าคงที่ระบบ (ตั้งไว้ตั้งแต่ M0 — `lib/constants.ts`)

```
TIMEZONE = Asia/Bangkok, TZ_OFFSET = +07:00
SLOT_MINUTES = 30 · OPEN_HOUR = 8 · CLOSE_HOUR = 18
BOOKING_WINDOW_DAYS = 30 · MIN_DURATION = 30 นาที · MAX_DURATION = 240 นาที
AMENITY_OPTIONS = projector, tv, whiteboard, video_conference, sound_system, wifi
```

---

## M0 — Setup (โครง + Data Layer + Design Tokens)

**เป้าหมาย:** โปรเจกต์รันได้, DB อ่าน-เขียนได้, design tokens พร้อม

**งาน:**
- `create-next-app` (App Router, TS, Tailwind) หรือสร้าง config มือ: `package.json`, `tsconfig.json`, `next.config.mjs` (ใส่ `serverComponentsExternalPackages: ['better-sqlite3','bcryptjs']`), `tailwind.config.ts`, `postcss.config.mjs`
- ติดตั้ง: `better-sqlite3 zod bcryptjs` + dev `tsx @types/better-sqlite3 @types/bcryptjs`
- `app/globals.css` — CSS variables (tokens สี Clean & Modern) + `tailwind.config.ts` map tokens
- `app/layout.tsx` — ฟอนต์ Inter + Noto Sans Thai
- `lib/constants.ts`, `lib/types.ts`
- **Data layer:** `lib/db/connection.ts` (singleton + PRAGMA), `lib/db/migrate.ts` (4 ตาราง + index ตาม [schema.md](schema.md))
- `lib/db/users.ts`, `rooms.ts`, `bookings.ts` (repository เบื้องต้น)
- `lib/db/seed.ts` (2 users + 4 rooms + ภาพ placeholder)
- npm scripts: `db:migrate`, `db:seed`, `db:reset`
- **UI primitives:** `components/ui/` → `Button`, `Card`, `Input/Field`, `Badge`, `Alert`, `Modal`, `PageHeader`

**✅ ตรวจ:** `npm run db:migrate && npm run db:seed` สำเร็จ; `PRAGMA table_info` ครบ 4 ตาราง; `npm run dev` ขึ้นได้

---

## M1 — Auth & Role Guard

**เป้าหมาย:** เข้าระบบ + แยกสิทธิ์ฝั่ง server (F1)

**งาน:**
- `lib/auth/password.ts` (bcrypt hash/verify)
- `lib/auth/session.ts` (signed cookie HMAC: set/clear/get)
- `lib/auth/guard.ts` (`getVerifiedSession`, `requirePageUser`, `requirePageAdmin`, `getApiSession`)
- `lib/validation/schemas.ts` → `loginSchema`
- API: `api/auth/login` (validate → verify → block ถ้า inactive → setSession), `api/auth/logout`
- `app/(auth)/login/page.tsx` + `LoginForm.tsx`
- `app/page.tsx` (redirect ตาม session)
- `components/app-shell/Sidebar.tsx` + `AppShell.tsx` (เมนูตาม role + logout)
- `app/(app)/layout.tsx` (requirePageUser) + `app/(admin)/layout.tsx` (requirePageAdmin)

**✅ ตรวจ:** login ถูก/ผิด (200/401), บัญชีปิด → 403, เปิด `/rooms` โดยไม่ login → redirect 307

---

## M2 — Rooms & Gallery (+ Admin Room Mgmt เบื้องต้น)

**เป้าหมาย:** ดูห้อง/ภาพได้ครบ (F2, F3)

**งาน:**
- repository `rooms.ts`: `listRooms`, `getRoom` (รวม images), `createRoom`, `updateRoom`, `addImage`
- `schemas.ts` → `roomSchema`
- API: `rooms` (GET/POST), `rooms/[id]` (GET/PUT), `rooms/[id]/images` (POST upload)
- หน้า: `(app)/rooms/page.tsx` + `RoomCard.tsx`; `(app)/rooms/[id]/page.tsx` + `Gallery.tsx` (grid + Lightbox)

**✅ ตรวจ:** หน้า /rooms แสดงการ์ด; คลิกเข้า detail เห็น gallery; admin POST /api/rooms ได้, user โดน 403

---

## M3 — Booking Core (กันชนอัตโนมัติ) ⭐ สำคัญสุด

**เป้าหมาย:** จองได้ + กันชนจริงระดับ DB (F4, F5)

**งาน:**
- `lib/booking/overlap.ts` (`isOverlap`, `hasConflict`)
- `lib/booking/slots.ts` (`slotTimes`, `toISO`, `todayISODate`, `maxBookingDate`, `validateBookingTime`, `timeOf`)
- `lib/db/bookings.ts`: `createBookingTx` (transaction!), `ConflictError`, `listByRoomDate`, `listByDate`, `listByUser`, `getBooking`, `deleteBooking`
- `schemas.ts` → `createBookingSchema`
- API `bookings` (GET list + POST create ตามแพตเทิร์น [architecture.md](architecture.md) §3.3)
- `components/booking/BookingForm.tsx` (เลือกวัน/slot ว่าง/หัวข้อ; จัดการ 409)

**✅ ตรวจ (บังคับ):**
- overlap → 409, adjacent (15:00→16:00 ต่อ 13:00–15:00) → 201, นอกเวลา → 400
- **Concurrency:** ยิง 8 request พร้อมกัน slot เดียวกัน → สำเร็จ 1, ปฏิเสธ 7
```bash
for i in $(seq 1 8); do curl -s -b cookie -X POST .../api/bookings -d '{...same slot...}' & done; wait
```

---

## M4 — Calendar & My Bookings

**เป้าหมาย:** เห็นภาพรวม + จัดการการจองตนเอง (F6, F7)

**งาน:**
- API: `bookings/[id]` (DELETE — owner/admin, ห้ามยกเลิกที่เริ่มแล้ว)
- `components/calendar/CalendarView.tsx` (timeline ทุกห้อง, ของฉันสีต่าง) + `(app)/calendar/page.tsx`
- `components/booking/MyBookingsList.tsx` (แยกกำลังจะถึง/ผ่านมาแล้ว + ยกเลิก) + `(app)/my-bookings/page.tsx`

**✅ ตรวจ:** ปฏิทินแสดง block ถูกตำแหน่งเวลา; ยกเลิกการจอง → slot ว่างทันที; ยกเลิกของคนอื่น → 403

---

## M5 — Admin: Room & Gallery Management (เต็มรูปแบบ)

**เป้าหมาย:** จัดการห้อง + ภาพครบ (F8)

**งาน:**
- repository: `getImage`, `deleteImage` (เลื่อนปกถ้าลบภาพปก), `setCover`
- API: `rooms/[id]/images/[imageId]` (DELETE + PATCH ตั้งปก)
- `(admin)/admin/rooms/page.tsx` + `AdminRooms.tsx` (เพิ่ม/แก้/ปิดห้อง) + `ImageManager.tsx` (popup อัปโหลด/ลบ/ตั้งปก + feedback)

**✅ ตรวจ:** อัปโหลด → เห็นในกริด; ตั้งปก (PATCH) → cover เปลี่ยน; ลบภาพปก → ปกเลื่อนอัตโนมัติ

---

## M6 — Admin: User Management

**เป้าหมาย:** จัดการผู้ใช้ครบวงจร (F9)

**งาน:**
- migration เพิ่ม `is_active` (additive — ดู [schema.md](schema.md) §6)
- repository `users.ts`: `listUsers`, `usernameTaken`, `countActiveAdmins`, `updateUser`, `setPassword`, `deleteUser`, `userHasBookings`
- `schemas.ts` → `createUserSchema`, `updateUserSchema`, `resetPasswordSchema`
- API: `users` (GET/POST), `users/[id]` (PUT/DELETE), `users/[id]/password` (POST)
- guard `getVerifiedSession` ให้ตรวจ `is_active` + คืน role ล่าสุด; login route block inactive
- `(admin)/admin/users/page.tsx` + `AdminUsers.tsx` (create/edit/role/reset/active/delete)

**✅ ตรวจ:** username ซ้ำ → 409, รหัสสั้น → 400, ลด/ลบ admin คนสุดท้าย → 400, ลบตัวเอง → 400, ปิดบัญชี → login 403, รีเซ็ตรหัส → login ด้วยรหัสใหม่ได้

---

## M7 — Polish (Clean UI, Responsive, A11y)

**เป้าหมาย:** ขัดเกลาให้พร้อมใช้ (PRD §8, §9)

**งาน:**
- ตรวจ design tokens สม่ำเสมอทั้งระบบ (สี/รัศมี 8px/spacing)
- ทุกหน้าใช้ `PageHeader`; error/success ใช้ `Alert`; สถานะใช้ `Badge`
- Sidebar responsive (เดสก์ท็อป fixed / มือถือ drawer); popup กึ่งกลาง + scroll
- empty/loading states; ตรวจ contrast (WCAG AA), focus ring, คีย์บอร์ด
- `npm run build` ผ่าน (รันตอน dev ปิดอยู่!)

**✅ ตรวจ:** build ผ่านทุก route; เปิดทุกหน้าได้ 200; ทดสอบมือถือ (responsive)

---

## เส้นทางวิกฤต (Critical Path)

```
M0 ─▶ M1 ─▶ M2 ─▶ M3(⭐กันชน) ─▶ M4 ─▶ M5 ─▶ M6 ─▶ M7
```
- M3 คือหัวใจและ metric อันดับ 1 (double booking = 0) — อย่าผ่านถ้า concurrency test ไม่ผ่าน
- M5/M6 ต้องการ M2 + M1; M4 ต้องการ M2 + M3

---

## Definition of Done (ทั้งระบบ)

- [ ] double booking = 0 (concurrency test ผ่าน)
- [ ] ทุก route ป้องกันด้วย role ฝั่ง server
- [ ] การจองทำใน transaction + constraint ระดับ schema
- [ ] validate ทุก input (Zod) + prepared statement
- [ ] ครบ F1–F9 ตาม Acceptance Criteria ใน [prd.md](prd.md)
- [ ] UI สม่ำเสมอ (tokens + primitives), responsive, WCAG AA
- [ ] `npx tsc --noEmit` และ `npm run build` ผ่าน
