# Progress — Smart Meeting Room Booking

> ไฟล์ติดตามความคืบหน้า (living document) — อัปเดตทุกครั้งที่ปิดงาน
> ในการสอน Vibe Coding ให้นักเรียน **กาเช็กตามจริง** ระหว่างสร้าง และให้ AI อัปเดตไฟล์นี้หลังแต่ละ milestone
> สถานะต้นฉบับ: ✅ **สร้างเสร็จครบทุก milestone และทดสอบแล้ว**

อัปเดตล่าสุด: 2026-06-22

---

## 1. สถานะ Milestone

| Milestone | ขอบเขต | สถานะ |
|---|---|---|
| **M0** Setup | โครงโปรเจกต์ + data layer + design tokens + primitives | ✅ เสร็จ |
| **M1** Auth | login + session(HMAC) + role guard ฝั่ง server | ✅ เสร็จ |
| **M2** Rooms & Gallery | room list/detail + gallery + lightbox | ✅ เสร็จ |
| **M3** Booking Core ⭐ | booking form + overlap check ใน transaction | ✅ เสร็จ |
| **M4** Calendar & My Bookings | timeline ทุกห้อง + ยกเลิกการจอง | ✅ เสร็จ |
| **M5** Admin Room Mgmt | จัดการห้อง + gallery (อัปโหลด/ลบ/ตั้งปก) | ✅ เสร็จ |
| **M6** User Management | จัดการผู้ใช้ครบ + safety guards | ✅ เสร็จ |
| **M7** Polish | Clean UI + responsive + a11y + รัศมี 8px | ✅ เสร็จ |

---

## 2. เช็กลิสต์ฟีเจอร์ (F1–F9)

- [x] **F1** Authentication & Role — login/logout, session, guard, block บัญชีปิด
- [x] **F2** Room List & Detail — การ์ดห้อง + หน้ารายละเอียด
- [x] **F3** Room Gallery — หลายภาพ + lightbox + จัดการภาพ (admin)
- [x] **F4** Booking — เลือก slot ว่าง + ยืนยัน + จัดการ 409
- [x] **F5** Conflict Prevention — overlap ใน SQL transaction (atomic)
- [x] **F6** Calendar / Timeline — ทุกห้องในจอเดียว, ของฉันสีต่าง
- [x] **F7** My Bookings — กำลังจะถึง/ผ่านมาแล้ว + ยกเลิก
- [x] **F8** Admin Room & Gallery Mgmt — เพิ่ม/แก้/ปิดห้อง + ImageManager
- [x] **F9** Admin User Mgmt — create/edit/role/reset/active/delete + guards

---

## 3. ผลการทดสอบ (Verification Log)

| รายการ | วิธีทดสอบ | ผล |
|---|---|---|
| Concurrency (กันชน) | ยิง 8 POST /api/bookings พร้อมกัน slot เดียว | ✅ 201×1, 409×7 |
| Overlap logic | overlap / adjacent / นอกเวลา | ✅ 409 / 201 / 400 |
| Role guard | user เรียก admin API | ✅ 403 |
| Auth | login ถูก/ผิด, บัญชีปิด | ✅ 200 / 401 / 403 |
| ยกเลิกการจอง | DELETE owner/คนอื่น | ✅ 200 / 403 |
| User mgmt guards | username ซ้ำ, admin คนสุดท้าย, ลบตัวเอง, ลบผู้ใช้มีจอง | ✅ 409 / 400 / 400 / 400 |
| Image mgmt | upload / set cover / delete / ลบภาพปก | ✅ 201 / 200 / 200 / ปกเลื่อนอัตโนมัติ |
| Build & typecheck | `npm run build`, `tsc --noEmit` | ✅ ผ่าน 15 routes |

---

## 4. ข้อมูลโปรเจกต์ (สรุปตัวเลข)

- **8 หน้า** (login, rooms, rooms/[id], calendar, my-bookings, admin/rooms, admin/users, + root redirect)
- **11 API routes** (auth ×2, rooms ×4, bookings ×2, users ×3)
- **14 components** (7 ui primitives + 7 feature/shell)
- **14 lib modules** (db ×6, auth ×3, booking ×2, + constants/types/validation)
- **4 ตาราง** SQLite + 2 indexes
- **Dependencies (runtime):** next, react, react-dom, better-sqlite3, bcryptjs, zod

---

## 5. ข้อจำกัดที่รู้แล้ว (Known Limitations)

- รันได้บน **single Node process** เท่านั้น (SQLite write-lock + in-memory connection)
- ต้องมี **persistent disk** เขียน `data/app.db` และ `public/uploads` — ไม่เหมาะ serverless multi-instance / read-only FS
- ไม่มี Email/Notification, ไม่มี Approval workflow, ไม่มีหน้าสมัครเอง (อยู่ใน Out-of-scope)
- session อายุ 7 วันใน cookie; ใช้ secret จาก `SESSION_SECRET` (ต้องเปลี่ยนใน production)
- ภาพอัปโหลดไม่บีบอัด/ปรับขนาด (จำกัดแค่ ≤ 5MB)

---

## 6. บทเรียน / กับดักที่เจอระหว่างสร้าง (สำหรับสอน)

1. **`.next` cache เพี้ยน** — รัน `npm run build` ขณะ `npm run dev` ทำให้เกิด `Cannot find module './xxx.js'` → แก้: ปิด dev, `rm -rf .next`, dev ใหม่ → **อย่า build ขณะ dev รัน**
2. **กันชนต้องอยู่ใน transaction** — ตรวจ overlap แล้ว INSERT แยกกันไม่ atomic; ต้องห่อใน `db.transaction()`
3. **Additive migration** — เพิ่มคอลัมน์ `is_active` ภายหลังโดยไม่ลบข้อมูลเดิม (ALTER TABLE หลังเช็ค PRAGMA)
4. **Guard ต้อง verify กับ DB** — เชื่อ cookie อย่างเดียวทำให้ปิดบัญชี/เปลี่ยน role ไม่มีผลทันที
5. **Consistency ของ UI** — เกิดจากใช้ token + primitive ชุดเดียว; การ hardcode สีคือบ่อเกิดความไม่สอดคล้อง

---

## 7. ขั้นถัดไป (ถ้าจะต่อยอด)

- [ ] ย้าย SQLite → Postgres (แก้แค่ `lib/db/`) เพื่อรองรับ multi-instance
- [ ] เพิ่ม Email/Notification ตอนจอง/ยกเลิก
- [ ] Recurring booking + reschedule
- [ ] Dashboard สถิติการใช้ห้อง
- [ ] บีบอัด/ปรับขนาดภาพตอนอัปโหลด (sharp) + ย้ายไป object storage
- [ ] ปุ่มสลับ Dark mode (tokens รองรับอยู่แล้ว)

---

## 8. วิธีรัน (Quick Start)

```bash
npm install
npm run db:migrate && npm run db:seed
npm run dev    # http://localhost:3000 (หรือ 3001 ถ้าพอร์ตไม่ว่าง)
```
เข้าสู่ระบบ: `jira / password123` (admin) · `staff / password123` (user)
