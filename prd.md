# PRD — Smart Meeting Room Booking

> ระบบจองห้องประชุมภายในคณะ — เน้นกันการจองชนกัน 100%, UI แบบ Clean & Modern, และจัดเก็บข้อมูลด้วย SQLite (embedded, ไฟล์เดียว)
>
> 📚 **เอกสารชุดนี้ใช้สำหรับสอน Vibe Coding** — สร้างระบบนี้ขึ้นใหม่ตั้งแต่ต้นด้วย AI โดยอ่านคู่กับ [AGENTS.md](AGENTS.md), [implementation-plan.md](implementation-plan.md), [architecture.md](architecture.md), [schema.md](schema.md), [progress.md](progress.md)

| ฟิลด์ | รายละเอียด |
|---|---|
| **ชื่อโปรเจกต์** | Smart Meeting Room Booking |
| **เวอร์ชันเอกสาร** | 2.0 (สะท้อนระบบที่สร้างเสร็จจริง) |
| **สถานะ** | ✅ MVP สร้างเสร็จและทดสอบแล้ว |
| **Tech Stack** | Next.js 14 (App Router) + TypeScript + Tailwind CSS |
| **Data Layer** | SQLite (embedded) ผ่าน `better-sqlite3` |
| **กลุ่มผู้ใช้** | บุคลากร / อาจารย์ / เจ้าหน้าที่ภายในคณะ |

---

## 1. วัตถุประสงค์ (Objectives)

ปัจจุบันการจองห้องประชุมภายในคณะมักทำผ่านช่องทางกระจัดกระจาย (โทร, แชท, สมุดจองหน้าห้อง) ทำให้เกิด **การจองชนกัน (Double Booking)**, ไม่เห็นภาพรวมตารางการใช้งาน, และขาดข้อมูลห้องที่ชัดเจน

ระบบนี้แก้ปัญหาโดย:

1. **รวมศูนย์การจอง** ให้อยู่ที่เดียว เห็นสถานะห้องทุกห้อง
2. **กันการจองชนกันอัตโนมัติ** ด้วยกติกา First-Come-First-Served ที่ทำงานระดับฐานข้อมูล (atomic transaction)
3. **ช่วยตัดสินใจ** ด้วย Gallery ภาพห้อง + ข้อมูลสิ่งอำนวยความสะดวก
4. **มอบประสบการณ์ใช้งานที่สะอาด ทันสมัย** (Clean & Modern) ใช้ง่ายทั้ง Desktop และ Mobile

---

## 2. เป้าหมายและตัวชี้วัด (Goals & Success Metrics)

| เป้าหมาย | ตัวชี้วัด | เกณฑ์ความสำเร็จ | ผลที่ได้ |
|---|---|---|---|
| ลดการจองชนกัน | เคส Double Booking ต่อเดือน | 0 เคส | ✅ ทดสอบ concurrency 8 request พร้อมกัน → สำเร็จ 1, ปฏิเสธ 7 |
| จองสะดวก | เวลาเฉลี่ยต่อการจอง | < 60 วินาที | ✅ flow 3 ขั้น เลือกห้อง→เวลา→ยืนยัน |
| ใช้งานได้จริง | แทนช่องทางเดิม | ≥ 80% | — (วัดหลัง deploy) |
| คุณภาพ UX | ความพึงพอใจ | ≥ 4.0/5.0 | — (วัดหลัง deploy) |

---

## 3. ผู้ใช้และสิทธิ์ (Users & Roles)

ระบบมี Authentication และแยกสิทธิ์ 2 ระดับ

### 3.1 User (ผู้ใช้ทั่วไป)
- ดูรายการห้อง + Gallery ภาพ
- ดูปฏิทินการจองของทุกห้อง
- สร้าง / ยกเลิก **การจองของตนเอง**
- ดูประวัติการจองของตนเอง

### 3.2 Admin (ผู้ดูแลระบบ)
- ทำทุกอย่างที่ User ทำได้
- จัดการห้อง: เพิ่ม / แก้ไข / ปิดใช้งาน
- จัดการ Gallery: อัปโหลด / ลบ / ตั้งภาพปก
- **จัดการผู้ใช้: เพิ่ม / แก้ไข / เปลี่ยน role / รีเซ็ตรหัสผ่าน / เปิด-ปิดบัญชี / ลบ**

> **กติกาการกันชน:** First-Come-First-Served — ใครยืนยันก่อนได้สิทธิ์ก่อน ระบบล็อก slot นั้นทันทีระดับ DB transaction และผู้ใช้รายอื่นจะมองไม่เห็น slot นั้นเป็นตัวเลือกที่ว่าง

---

## 4. ขอบเขต (Scope)

### 4.1 In-Scope — สร้างเสร็จแล้ว ✅
- ระบบ Login + แยก Role (Admin / User) — session-based
- หน้ารายการห้อง (Room List) + ภาพหน้าปก
- Gallery ภาพห้อง (หลายภาพ + Lightbox)
- หน้าจองห้อง (Booking Form) เลือกวัน–เวลาแบบ slot 30 นาที
- ปฏิทินแสดงตารางการจองทุกห้อง (Timeline View)
- กลไกกันการจองชนกันอัตโนมัติ (SQL transaction)
- ดู / ยกเลิกการจองของตนเอง
- Admin: จัดการห้อง + Gallery (อัปโหลด/ลบ/ตั้งปก)
- **Admin: จัดการผู้ใช้แบบครบวงจร** (เพิ่มขึ้นจากแผนเดิม)
- จัดเก็บข้อมูลทั้งหมดใน SQLite

### 4.2 Out-of-Scope (รอบถัดไป) ⏭️
- การยืนยัน/แจ้งเตือนผ่าน Email
- ระบบอนุมัติการจอง (Approval Workflow)
- หน้าสมัครสมาชิกเอง (Self-register) — รอบนี้ Admin สร้างให้
- การจองแบบเกิดซ้ำ (Recurring Booking)
- รายงาน / Dashboard สถิติ
- เชื่อมต่อปฏิทินภายนอก (Google / Outlook)
- Check-in / Check-out หน้างาน
- Database server แยก (Postgres/MySQL) — รอบนี้ใช้ SQLite แบบ embedded

---

## 5. ฟีเจอร์หลัก (Core Features)

> แต่ละฟีเจอร์มี **Acceptance Criteria** ที่ตรวจสอบได้ — ใช้เป็นเช็กลิสต์ตอนสร้างใหม่

### F1 — Authentication & Role
- ต้อง Login ก่อนเข้าถึงทุกฟีเจอร์ (กันระดับ layout ฝั่ง server)
- ระบบจดจำ session + role; เมนู Admin ซ่อนจาก User
- รหัสผ่าน hash ด้วย bcrypt; session เป็น signed cookie (HMAC)
- มีปุ่ม Logout; บัญชีที่ถูกปิดใช้งานเข้าระบบไม่ได้

### F2 — Room List & Detail
- แสดง: ชื่อห้อง, ความจุ, ที่ตั้ง, สิ่งอำนวยความสะดวก, ภาพหน้าปก, สถานะ
- คลิกห้อง → หน้ารายละเอียด + Gallery + ตารางการจองวันนี้
- Admin เห็นห้องที่ปิดใช้งานด้วย; User เห็นเฉพาะห้องเปิด

### F3 — Room Gallery
- หลายภาพต่อห้อง, แสดงเป็น grid + เปิดภาพใหญ่ (Lightbox)
- Admin: อัปโหลด / ลบ / ตั้งเป็นภาพปก
- ไฟล์เก็บใน `/public/uploads/rooms/<roomId>/`, path เก็บในตาราง `room_images`

### F4 — Booking (จองห้อง)
- เลือกห้อง → วันที่ → ช่วงเวลา (slot 30 นาที) → หัวข้อ → ยืนยัน
- แสดงเฉพาะ slot ที่ว่าง (slot ที่ถูกจอง disable)
- ตรวจ overlap ฝั่ง server ก่อนบันทึก; ถ้าชน (race) → ปฏิเสธพร้อมข้อความ + รีเฟรช slot

### F5 — Booking Conflict Prevention (กันชนอัตโนมัติ)
- การจอง = ช่วง `[start, end)` ผูกกับ `roomId` + วันที่
- เงื่อนไข overlap: `existing.start < new.end AND new.start < existing.end`
- การ "ตรวจ → เขียน" อยู่ใน **SQL transaction เดียว** (`BEGIN IMMEDIATE`) → atomic, กัน race condition โดยไม่ต้องเขียน mutex เอง
- รายละเอียดเชิงลึกดู [architecture.md](architecture.md) §Booking Flow

### F6 — Calendar / Timeline View
- มุมมองรายวันของทุกห้องในจอเดียว (timeline 08:00–18:00)
- แต่ละ block แสดงหัวข้อ + ผู้จอง + ช่วงเวลา (tooltip)
- การจองของตนเองมีสีต่างจากของผู้อื่น; เลือกวันได้

### F7 — My Bookings
- ดูการจองของตนเอง แยก "กำลังจะถึง / ผ่านมาแล้ว"
- ยกเลิกได้เฉพาะการจองที่ยังไม่เริ่ม → slot ว่างทันที

### F8 — Admin: Room & Gallery Management
- เพิ่ม / แก้ไข / ปิดใช้งานห้อง (ชื่อ, ความจุ, ที่ตั้ง, สิ่งอำนวยความสะดวก)
- จัดการ Gallery ของแต่ละห้อง (อัปโหลด/ลบ/ตั้งปก) ผ่าน popup

### F9 — Admin: User Management
- เพิ่มผู้ใช้ใหม่ (username, ชื่อแสดง, รหัสผ่าน, role)
- แก้ไขข้อมูล + เปลี่ยน role (admin/user)
- รีเซ็ตรหัสผ่าน
- เปิด/ปิดใช้งานบัญชี + ลบผู้ใช้
- **Safety guards:** ต้องมี admin ที่ใช้งานได้ ≥ 1 คนเสมอ, ลบบัญชีตัวเองไม่ได้, ลบผู้ใช้ที่มีประวัติจองไม่ได้ (ปิดใช้งานแทน)

---

## 6. โครงสร้างข้อมูล (Data Model)

> รายละเอียดเต็มดู [schema.md](schema.md) — สรุปย่อ:

- **users** — id, username (unique), display_name, password_hash (bcrypt), role (admin/user), is_active, created_at
- **rooms** — id, name, capacity, location, amenities (JSON string), cover_image, is_active, created_at
- **room_images** — id, room_id (FK), path, sort_order
- **bookings** — id, room_id (FK), user_id (FK), title, date, start_at, end_at, created_at

ฐานข้อมูลเดียวที่ไฟล์ `data/app.db` · เปิด WAL + foreign_keys · ทุกการเขียนผ่าน API route ฝั่ง server

---

## 7. สถาปัตยกรรมและ Tech Stack

| ชั้น | เทคโนโลยี | บทบาท |
|---|---|---|
| Framework | Next.js 14 (App Router) | Server Components + API Routes |
| Language | TypeScript (strict) | Type safety ทั้งระบบ |
| Styling | Tailwind CSS | Utility-first + Design tokens |
| Auth | Signed cookie session (HMAC) + bcryptjs | Login + Role guard |
| Persistence | SQLite (`better-sqlite3`) | users / rooms / bookings |
| Validation | Zod | ตรวจ input client & server |

> รายละเอียดชั้นและ flow ดู [architecture.md](architecture.md)

---

## 8. ทิศทางการออกแบบ UI/UX (Clean & Modern)

เป้าหมาย: สะอาด ทันสมัย แบบ SaaS — ใช้ง่าย อ่านชัด สม่ำเสมอ

| องค์ประกอบ | แนวทาง |
|---|---|
| **Layout** | Sidebar ซ้าย (เมนู + ผู้ใช้ + logout); มือถือยุบเป็น top bar + drawer |
| **โทนสี** | พื้นเทาอ่อนมาก `#f7f7f8` + การ์ดขาว + **accent ส้ม `#ea580c`** ใช้เน้นจุดสำคัญ |
| **ตัวอักษร** | Sans-serif ทั้งระบบ (Inter + Noto Sans Thai), หัวเรื่องน้ำหนัก 600 |
| **Semantic colors** | success (เขียว) / danger (แดง) / warning (เหลือง) สำหรับสถานะ |
| **Radius** | การ์ด / ปุ่ม / popup = **8px** เท่ากันทั้งระบบ; badge เป็นแคปซูล |
| **Components** | ใช้ primitive ซ้ำ: Button, Badge, Card, Input/Field, Alert, Modal, PageHeader |
| **Popup** | อยู่กึ่งกลางจอ + scroll ในตัวเมื่อเนื้อหายาว |

> หลักการ: **consistency มาก่อน** — ทุก element เกิดจาก primitive กลางชุดเดียว ไม่เขียน class มือซ้ำ ๆ

---

## 9. Non-Functional Requirements

- **Performance:** หน้าโหลดเร็ว; ภาพ Gallery lazy load
- **Responsive:** ใช้งานได้ดีทั้ง Desktop และ Mobile
- **Accessibility:** คอนทราสต์ผ่าน WCAG AA, focus ring ชัด, ใช้คีย์บอร์ดได้
- **Security:** password hash (bcrypt), session signed (HMAC), ป้องกัน route ตาม role **ฝั่ง server**, validate input ทุกจุด (Zod), prepared statement กัน SQL injection
- **Maintainability:** แยก data access layer (`lib/db/*`) ชัดเจน เปลี่ยนไป DB จริงได้โดยไม่กระทบ UI
- **Data Integrity:** การจองทำใน SQL transaction + constraint ระดับ schema (CHECK, UNIQUE, FK)

---

## 10. แผนการพัฒนา (สรุป)

> ลำดับการสร้างทีละขั้นดู [implementation-plan.md](implementation-plan.md) (M0–M7)

M0 Setup → M1 Auth → M2 Rooms & Gallery → M3 Booking Core (กันชน) → M4 Calendar & My Bookings → M5 Admin Room Mgmt → M6 User Management → M7 Polish (Clean UI, responsive, a11y)

---

## 11. การตัดสินใจที่ใช้ (Locked Decisions)

| ประเด็น | ค่าที่ใช้ |
|---|---|
| Time slot | ทีละ 30 นาที (เลือกอิสระแต่ลงตัว :00 / :30) |
| เวลาทำการ | 08:00–18:00 |
| จองล่วงหน้า | สูงสุด 30 วัน; ยกเลิกได้ก่อนเวลาเริ่ม |
| Duration | ขั้นต่ำ 30 นาที, สูงสุด 4 ชั่วโมง |
| Timezone | ตรึงทั้งระบบเป็น Asia/Bangkok (UTC+7) |
| User account | Admin สร้างให้ (ไม่มีหน้าสมัครเอง) |

---

## 12. ความเสี่ยง / ข้อจำกัด

| ประเด็น | แนวทางรับมือ |
|---|---|
| SQLite + in-memory session | ต้องรันเป็น **single Node process** (เครื่อง/VM เดียว) — ไม่เหมาะ serverless multi-instance |
| เก็บภาพใน `/public/uploads` | ต้องมี persistent disk เขียนไฟล์ได้; จำกัดขนาด ≤ 5MB |
| `better-sqlite3` เป็น native module | ปกติมี prebuilt binary; ถ้า build ไม่ผ่านต้องมี build tools (Xcode CLT บน macOS) |
| ข้อมูลโตขึ้น | มี index `(room_id, date)`; ย้าย Postgres ได้ผ่าน data layer |

---

*เอกสารนี้สะท้อนระบบที่สร้างเสร็จจริง — ใช้คู่กับเอกสารอีก 5 ไฟล์เพื่อสร้างขึ้นใหม่*
