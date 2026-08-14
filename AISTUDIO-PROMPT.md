# AISTUDIO-PROMPT.md — Prompt สำหรับสร้างระบบด้วย Google AI Studio

> แปลงเอกสารชุด Vibe Coding (prd / architecture / schema / implementation-plan) ให้เป็น Prompt
> สำหรับ **Google AI Studio → Build Apps** (https://aistudio.google.com/apps)
>
> ⚠️ อ่าน §1 ก่อน — สถาปัตยกรรมต้นฉบับ (Next.js + SQLite) รันบน AI Studio ไม่ได้ ต้องแปลง 5 จุด

---

## 1. ข้อจำกัดของ AI Studio Apps และการแปลงสถาปัตยกรรม

AI Studio Apps สร้าง **Single-Page App (React + TypeScript) ที่รันในเบราว์เซอร์ล้วน**
ไม่มี Node process / ไม่มี API route / ไม่มีไฟล์ระบบ → 5 จุดนี้ต้องเปลี่ยน

| # | ต้นฉบับ (Antigravity / Next.js) | บน AI Studio | ผลกระทบ |
|---|---|---|---|
| 1 | SQLite `data/app.db` (`better-sqlite3`) | **localStorage** ผ่าน data layer หน้าตาเดิม (`lib/db/*`) | โครง repository เหมือนเดิม เปลี่ยนแค่ engine |
| 2 | `BEGIN IMMEDIATE` transaction กันชน | **read → check overlap → write ในฟังก์ชัน sync เดียว** + version guard (optimistic lock) | JS เป็น single-thread → ในแท็บเดียวปลอดภัย 100%; ข้ามแท็บใช้ version guard |
| 3 | bcrypt + signed cookie (HMAC) ฝั่ง server | **SHA-256 + salt (Web Crypto)** + session ใน `sessionStorage` | ⚠️ **เป็น demo auth เท่านั้น** — ไม่ใช่ security จริง ต้องเขียนกำกับใน UI |
| 4 | Guard ฝั่ง server ที่ `layout.tsx` | **Route guard ฝั่ง client** ใน `<App>` | ⚠️ กันได้แค่ระดับ UX — ต้องระบุว่าของจริงต้องมี backend |
| 5 | อัปโหลดภาพลง `public/uploads/` | **base64 data URL** เก็บใน localStorage (≤ 300KB/ภาพ) หรือ URL ภายนอก | จำกัดโควตา localStorage ~5MB |

> **สรุป:** ได้ระบบที่ **ฟีเจอร์ครบ F1–F9 และ demo ได้จริง** เหมาะสำหรับสอน/นำเสนอ
> แต่ **ไม่ใช่ระบบ production** — ต้องย้ายไป backend จริงก่อนใช้งานจริง (บอกนักเรียนให้ชัด)

---

## 2. PROMPT A — One-shot (คัดลอกทั้งบล็อกไปวางใน AI Studio) 🚀

> ใช้อันนี้เป็นหลัก · วางในช่อง prompt ของ **Build Apps** แล้วกด Build
> ถ้าผลลัพธ์ยังไม่ครบ ให้ตามด้วย PROMPT B (ทีละเฟส) เพื่อเติม

````
สร้างเว็บแอป "Smart Meeting Room Booking" (ระบบจองห้องประชุมภายในคณะ) เป็น
React + TypeScript + Tailwind CSS แบบ frontend ล้วน รันในเบราว์เซอร์ ไม่มี backend
UI ภาษาไทยทั้งหมด

=== ข้อจำกัดสภาพแวดล้อม (สำคัญ ห้ามฝ่าฝืน) ===
- ห้ามใช้ Node API, ห้ามใช้ไฟล์ระบบ, ห้ามเรียก server/database ภายนอก
- เก็บข้อมูลทั้งหมดใน localStorage คีย์เดียวชื่อ "smrb.db.v1"
- โครงข้อมูลใน localStorage เลียนแบบตารางฐานข้อมูล 4 ตาราง:
  users, rooms, roomImages, bookings

=== สถาปัตยกรรมที่ต้องการ (แยกชั้นให้ชัด) ===
src/
  lib/constants.ts     ค่าคงที่ระบบทั้งหมด
  lib/types.ts         TypeScript types
  lib/db/store.ts      อ่าน/เขียน localStorage + seed + version guard
  lib/db/users.ts      repository ผู้ใช้
  lib/db/rooms.ts      repository ห้อง + ภาพ
  lib/db/bookings.ts   repository การจอง + createBooking (หัวใจกันชน)
  lib/booking/slots.ts สร้าง slot 30 นาที + validate เวลา
  lib/booking/overlap.ts สูตรตรวจ overlap
  lib/auth/*           hash รหัสผ่าน + session
  components/ui/       primitive: Button, Card, Input, Select, Field, Badge, Alert, Modal, PageHeader
  components/          feature components
  App.tsx              router + guard

กฎเหล็ก:
1) ทุกการอ่าน/เขียนข้อมูลต้องผ่าน lib/db/* เท่านั้น — component ห้ามแตะ localStorage ตรง ๆ
2) ทุกสี/รัศมี ใช้ design token จาก Tailwind config หรือ CSS variable — ห้าม hardcode สีในคอมโพเนนต์
3) การจองต้องเรียก createBooking() ตัวเดียวเท่านั้น (ตรวจชน + เขียน อยู่ในฟังก์ชันเดียว)
4) validate input ทุกจุดก่อนบันทึก และแสดง error เป็นภาษาไทย

=== ค่าคงที่ระบบ (lib/constants.ts) ===
TIMEZONE = 'Asia/Bangkok'  (ตรึงทั้งระบบ ห้ามใช้ UTC)
SLOT_MINUTES = 30 · OPEN_HOUR = 8 · CLOSE_HOUR = 18
BOOKING_WINDOW_DAYS = 30   (จองล่วงหน้าได้ไม่เกิน 30 วัน)
MIN_DURATION = 30 นาที · MAX_DURATION = 240 นาที
AMENITY_OPTIONS = projector(โปรเจกเตอร์), tv(ทีวี), whiteboard(ไวท์บอร์ด),
                  video_conference(ระบบประชุมทางไกล), sound_system(เครื่องเสียง), wifi(ไวไฟ)

=== โครงสร้างข้อมูล ===
User    { id:'u_xxx', username, displayName, passwordHash, salt, role:'admin'|'user', isActive:boolean, createdAt }
Room    { id:'r_xxx', name, capacity:number, location, amenities:string[], coverImage:string|null, isActive:boolean, createdAt }
RoomImage { id:'i_xxx', roomId, dataUrl:string, sortOrder:number }
Booking { id:'b_xxx', roomId, userId, title, date:'YYYY-MM-DD',
          startAt:'YYYY-MM-DDTHH:mm:00+07:00', endAt:same, createdAt }

เก็บ startAt/endAt เป็น string ISO ที่มี offset +07:00 เสมอ (รูปแบบเดียวกันทั้งระบบ)
เพื่อให้เปรียบเทียบด้วย < > แบบ string ได้ถูกต้อง

=== หัวใจของระบบ: กันการจองชนกัน (สำคัญที่สุด) ===
สูตร overlap มาตรฐาน:
  ชนกันเมื่อ  existing.startAt < new.endAt  AND  new.startAt < existing.endAt
  (ติดกันพอดี เช่น 13:00–15:00 กับ 15:00–16:00 = ไม่ชน ต้องจองได้)

createBooking(input) ต้องทำตามลำดับนี้ในฟังก์ชัน synchronous เดียว ห้ามมี await คั่นกลาง:
  1. อ่าน db ล่าสุดจาก localStorage (อ่านใหม่เสมอ ห้ามใช้ค่าที่ cache ไว้ใน state)
  2. ตรวจ validate เวลา: อยู่ในเวลาทำการ 08:00–18:00, ลงตัว :00/:30,
     ไม่ย้อนอดีต, ไม่เกิน 30 วันข้างหน้า, duration 30–240 นาที, ห้องต้อง isActive
  3. กรอง bookings ที่ roomId + date เดียวกัน แล้วเช็ค overlap ทุกตัว
     ถ้าเจอชน → throw ConflictError('ช่วงเวลานี้ถูกจองแล้ว กรุณาเลือกเวลาอื่น')
  4. เขียนกลับ localStorage พร้อมเพิ่ม db.version += 1
     ถ้า db.version ที่อ่านมาตอนแรก ≠ version ปัจจุบันใน localStorage → อ่านใหม่แล้วตรวจซ้ำ
     (optimistic lock กันกรณีเปิดหลายแท็บพร้อมกัน)

ที่หน้าฟอร์ม ให้ disable slot ที่ถูกจองไปแล้วด้วย (เป็นแค่ชั้น UX)
แต่การกันชนจริงต้องอยู่ใน createBooking เท่านั้น
ถ้า createBooking โยน ConflictError → แสดง Alert สีแดง + โหลดรายการ slot ว่างใหม่ทันที

=== ฟีเจอร์ที่ต้องมีครบ (F1–F9) ===
F1 Auth & Role
  - หน้า Login (username + password) ต้องเข้าระบบก่อนใช้ทุกหน้า
  - รหัสผ่าน hash ด้วย Web Crypto SHA-256 + salt สุ่มต่อผู้ใช้ (ห้ามเก็บ plaintext)
  - session เก็บใน sessionStorage; ทุกครั้งที่ render ต้องตรวจซ้ำกับ users ใน db
    ว่ายังมีตัวตนและ isActive อยู่ (รองรับกรณีถูกปิดบัญชีระหว่างใช้งาน)
  - เมนู Admin ต้องซ่อนจาก role user และกันเข้าถึงหน้า admin ด้วย
  - มีปุ่มออกจากระบบ; บัญชีที่ถูกปิดใช้งาน login ไม่ได้ (แจ้ง "บัญชีถูกระงับการใช้งาน")
  - แสดงแบนเนอร์เตือนที่หน้า Login ว่า "ระบบสาธิต — ข้อมูลเก็บในเบราว์เซอร์เครื่องนี้เท่านั้น
    ระบบยืนยันตัวตนไม่ปลอดภัยระดับใช้งานจริง"

F2 Room List & Detail
  - การ์ดห้อง: ภาพปก, ชื่อ, ความจุ, ที่ตั้ง, ไอคอนสิ่งอำนวยความสะดวก, badge สถานะ
  - คลิก → หน้ารายละเอียด + Gallery + ตารางการจองของวันนี้
  - user เห็นเฉพาะห้องที่เปิด, admin เห็นห้องปิดด้วย (มี badge "ปิดใช้งาน")

F3 Room Gallery
  - หลายภาพต่อห้อง แสดงเป็น grid, คลิกเปิด Lightbox (ปิดด้วย Esc / คลิกพื้นหลัง)
  - admin: อัปโหลด (อ่านไฟล์เป็น base64 data URL, จำกัด ≤ 300KB ต่อภาพ,
    ถ้าเกินให้ย่อด้วย canvas ก่อน), ลบ, ตั้งเป็นภาพปก
  - ลบภาพที่เป็นปก → เลื่อนปกไปภาพถัดไปอัตโนมัติ

F4 Booking
  - ขั้นตอน: เลือกห้อง → เลือกวันที่ → เลือกช่วงเวลา (slot 30 นาที) → ใส่หัวข้อ → ยืนยัน
  - แสดง slot 08:00–18:00 เป็นปุ่ม; slot ที่ถูกจองแล้ว disable + สีเทา
  - เลือกเวลาเริ่มและเวลาสิ้นสุด, แสดงระยะเวลาที่เลือก
  - จองสำเร็จ → Alert เขียว + เคลียร์ฟอร์ม + รีเฟรชตาราง

F5 Conflict Prevention — ตามหัวข้อ "หัวใจของระบบ" ข้างบน

F6 Calendar / Timeline
  - มุมมองรายวัน: แกนตั้ง = ห้องทั้งหมด, แกนนอน = เวลา 08:00–18:00
  - block การจองวางตำแหน่ง/ความกว้างตามเวลาจริง, แสดงหัวข้อ + ผู้จอง
  - hover เห็น tooltip (หัวข้อ, ผู้จอง, ช่วงเวลา)
  - การจองของฉันใช้สี accent ส้ม, ของคนอื่นใช้สีเทาอ่อน
  - เลื่อนวันก่อนหน้า/ถัดไป + ปุ่ม "วันนี้"

F7 My Bookings
  - แยก 2 กลุ่ม: "กำลังจะถึง" และ "ผ่านมาแล้ว"
  - ยกเลิกได้เฉพาะรายการที่ยังไม่ถึงเวลาเริ่ม → ยืนยันใน Modal → slot ว่างทันที

F8 Admin: จัดการห้อง + Gallery
  - เพิ่ม / แก้ไข / เปิด-ปิดใช้งานห้อง (ชื่อ, ความจุ, ที่ตั้ง, สิ่งอำนวยความสะดวกแบบ checkbox)
  - จัดการภาพของแต่ละห้องใน Modal (อัปโหลด / ลบ / ตั้งปก)

F9 Admin: จัดการผู้ใช้
  - เพิ่มผู้ใช้ (username, ชื่อแสดง, รหัสผ่าน ≥ 8 ตัว, role)
  - แก้ไขข้อมูล / เปลี่ยน role / รีเซ็ตรหัสผ่าน / เปิด-ปิดบัญชี / ลบ
  - Safety guards ที่ต้องมีครบ:
    * username ซ้ำไม่ได้ → "ชื่อผู้ใช้นี้ถูกใช้แล้ว"
    * ต้องเหลือ admin ที่ใช้งานได้อย่างน้อย 1 คนเสมอ (ห้ามลด role / ปิด / ลบ admin คนสุดท้าย)
    * ลบบัญชีตัวเองไม่ได้
    * ลบผู้ใช้ที่มีประวัติการจองไม่ได้ → แนะนำให้ปิดใช้งานแทน

=== ข้อมูลตัวอย่าง (seed อัตโนมัติเมื่อ localStorage ว่าง) ===
ผู้ใช้ 2 คน (รหัสผ่านทั้งคู่: password123)
  - jira  / ชื่อแสดง "จิรา (ผู้ดูแลระบบ)" / role admin
  - staff / ชื่อแสดง "สมชาย เจ้าหน้าที่"   / role user
ห้อง 4 ห้อง (แต่ละห้องมีภาพ placeholder 3 ภาพ สร้างเป็น inline SVG data URL
สีต่างกัน มีชื่อห้องบนภาพ — ห้ามลิงก์ภาพจากอินเทอร์เน็ต)
  1. ห้องประชุมสภาคณะ — 40 ที่นั่ง — ชั้น 3 อาคารอำนวยการ — projector, sound_system, video_conference, wifi
  2. ห้องประชุมใหญ่ (Auditorium) — 120 ที่นั่ง — ชั้น 1 อาคารเรียนรวม — projector, sound_system, wifi
  3. ห้องประชุมกลุ่มย่อย A — 12 ที่นั่ง — ชั้น 2 อาคารอำนวยการ — tv, whiteboard, wifi
  4. ห้อง Workshop — 24 ที่นั่ง — ชั้น 4 อาคารปฏิบัติการ — projector, whiteboard, wifi
การจองเริ่มต้น: สร้างตัวอย่าง 3 รายการในวันนี้และพรุ่งนี้ เพื่อให้เห็นปฏิทินมีข้อมูล
มีปุ่ม "รีเซ็ตข้อมูลตัวอย่าง" ในหน้า Admin (ยืนยันก่อนล้าง)

=== ดีไซน์ (Clean & Modern แบบ SaaS) ===
- Layout: Sidebar ซ้ายคงที่บนเดสก์ท็อป (โลโก้ + เมนู + ชื่อผู้ใช้ + ปุ่มออกจากระบบ);
  บนมือถือยุบเป็น top bar + drawer เลื่อนออกจากซ้าย
- เมนู: ห้องประชุม / ปฏิทิน / การจองของฉัน  + (เฉพาะ admin) จัดการห้อง / จัดการผู้ใช้
- สี: พื้นหลัง #f7f7f8 · การ์ดขาว #ffffff · เส้นขอบ #e5e5e7 ·
      ตัวอักษรหลัก #18181b, รอง #71717a · accent ส้ม #ea580c (hover #c2410c)
      success #16a34a · danger #dc2626 · warning #ca8a04 (พร้อมเวอร์ชันพื้นหลังอ่อนของแต่ละสี)
- ฟอนต์: Inter + Noto Sans Thai (โหลดจาก Google Fonts), หัวเรื่อง font-weight 600
- รัศมี 8px เท่ากันทั้งระบบ (การ์ด / ปุ่ม / input / modal); badge เป็นแคปซูล
- Modal อยู่กึ่งกลางจอ มี backdrop จาง เนื้อหายาวให้ scroll ในตัว
- ทุกหน้าใช้ PageHeader (หัวข้อ + คำอธิบาย + ปุ่ม action ขวา)
- error/success ใช้คอมโพเนนต์ Alert เท่านั้น; สถานะใช้ Badge เท่านั้น
- มี empty state ทุกรายการที่อาจว่าง (ไอคอน + ข้อความ + ปุ่มถัดไป)
- Responsive: ใช้ได้ดีทั้งเดสก์ท็อปและมือถือ
- Accessibility: คอนทราสต์ผ่าน WCAG AA, มี focus ring ชัดเจน, ใช้คีย์บอร์ดได้,
  Modal ปิดด้วย Esc, ปุ่มไอคอนมี aria-label

=== เกณฑ์ตรวจรับ (ต้องทำงานได้จริงทุกข้อ) ===
1. login ด้วย jira/password123 → เห็นเมนู admin ครบ; staff/password123 → ไม่เห็นเมนู admin
2. จอง 13:00–15:00 สำเร็จ → จองซ้ำ 14:00–16:00 ห้องเดิมวันเดิม ต้องถูกปฏิเสธ
3. จอง 15:00–16:00 ต่อจากข้อ 2 (ติดกันพอดี) ต้องสำเร็จ
4. จอง 07:00 หรือ 18:30 ต้องถูกปฏิเสธ (นอกเวลาทำการ); จองเกิน 4 ชั่วโมงต้องถูกปฏิเสธ
5. ยกเลิกการจองของตัวเอง → slot กลับมาว่างทันทีในฟอร์มและปฏิทิน
6. ปิดบัญชี staff แล้ว login ด้วย staff → ต้องขึ้น "บัญชีถูกระงับการใช้งาน"
7. ลอง ลบ/ปิด/ลดสิทธิ์ admin คนสุดท้าย → ต้องถูกปฏิเสธพร้อมข้อความอธิบาย
8. รีเฟรชหน้าเว็บ → ข้อมูลทั้งหมดยังอยู่ครบ (localStorage)
9. ย่อจอเป็นขนาดมือถือ → sidebar ยุบเป็น drawer และใช้งานได้ครบ

สร้างให้ครบทุกฟีเจอร์ในรอบเดียว เขียน TypeScript แบบ strict (ไม่มี any)
และแยกไฟล์ตามโครงสร้างที่กำหนดไว้ข้างบน
````

---

## 3. PROMPT B — แบบทีละเฟส (สำหรับการสอน)

> ถ้าอยากให้นักเรียนเห็นทีละชั้นแบบ M0–M7 เดิม ให้ยุบเป็น 5 เฟสตามนี้
> วาง PROMPT A เป็นข้อความแรกก่อน แล้วสั่งทีละเฟสในช่องแชทของ AI Studio

**เฟส 1 — โครง + ข้อมูล + ดีไซน์**
```
เริ่มจากเฟส 1 เท่านั้น ยังไม่ต้องทำฟีเจอร์อื่น:
- lib/constants.ts, lib/types.ts
- lib/db/store.ts (อ่าน/เขียน localStorage คีย์ smrb.db.v1 + seed ข้อมูลตัวอย่าง + version)
- repository: users.ts, rooms.ts, bookings.ts (ยังไม่ต้องมี createBooking)
- design token + UI primitives ครบชุด: Button, Card, Input, Select, Field, Badge, Alert,
  Modal, PageHeader
- หน้าเดโมชั่วคราวที่แสดง primitive ทุกตัว เพื่อให้ฉันตรวจดีไซน์ก่อน
```

**เฟส 2 — Login + Layout + Guard (F1)**
```
ทำเฟส 2: หน้า Login + AppShell (Sidebar/drawer) + route guard ตาม role
เน้น: hash รหัสผ่านด้วย Web Crypto SHA-256 + salt, ตรวจ isActive ซ้ำกับ db ทุกครั้งที่ render,
เมนู admin ต้องซ่อนจาก user
ตรวจให้ฉันดู: login ถูก, login ผิด, บัญชีที่ถูกปิด — ทั้ง 3 เคสขึ้นข้อความถูกต้อง
```

**เฟส 3 — ห้อง + Gallery (F2, F3)**
```
ทำเฟส 3: หน้ารายการห้อง (การ์ด) + หน้ารายละเอียดห้อง + Gallery แบบ grid และ Lightbox
user เห็นเฉพาะห้องที่เปิดใช้งาน, admin เห็นทั้งหมด
```

**เฟส 4 — จองห้อง + กันชน (F4, F5) ⭐ สำคัญที่สุด**
```
ทำเฟส 4: ฟอร์มจองห้อง + createBooking

บังคับ: createBooking ต้องอ่าน localStorage ใหม่ → ตรวจ overlap → เขียน
ในฟังก์ชัน synchronous เดียว ห้ามมี await คั่นกลาง และห้ามใช้ข้อมูลจาก React state
มาตัดสินว่าชนหรือไม่ (state อาจเก่า)

พิสูจน์ให้ฉันดูด้วยการทดสอบจริงในแอป แล้วบอกผลทีละเคส:
- จอง 13:00–15:00 → สำเร็จ
- จอง 14:00–16:00 ห้องเดิมวันเดิม → ต้องถูกปฏิเสธ
- จอง 15:00–16:00 (ติดกันพอดี) → ต้องสำเร็จ
- จอง 07:00 / 18:30 → ปฏิเสธ (นอกเวลาทำการ)
- จองยาว 5 ชั่วโมง → ปฏิเสธ (เกิน 4 ชั่วโมง)
ถ้าเคสไหนไม่ผ่าน ให้แก้จนผ่านก่อน อย่าไปเฟสถัดไป
```

**เฟส 5 — ปฏิทิน + การจองของฉัน + Admin (F6–F9)**
```
ทำเฟส 5 ให้ครบ:
- ปฏิทิน Timeline รายวันทุกห้อง (การจองของฉันสีส้ม, ของคนอื่นสีเทา)
- หน้าการจองของฉัน (แยกกำลังจะถึง/ผ่านมาแล้ว + ยกเลิก)
- Admin จัดการห้อง + Gallery
- Admin จัดการผู้ใช้ พร้อม safety guards ครบ 4 ข้อ
  (username ซ้ำ, ต้องเหลือ admin ≥ 1, ลบตัวเองไม่ได้, ลบคนที่มีประวัติจองไม่ได้)
แล้วไล่ตรวจ "เกณฑ์ตรวจรับ" ทั้ง 9 ข้อให้ฉันดูทีละข้อ
```

---

## 4. PROMPT C — ฟีเจอร์เสริมด้วย Gemini (ทำได้เฉพาะบน AI Studio)

> จุดแข็งของ AI Studio คือมี Gemini API พร้อมใช้ — ใช้สร้างฟีเจอร์ที่ต้นฉบับไม่มี
> เพิ่มหลังจากระบบหลักเสร็จแล้วเท่านั้น

```
เพิ่มฟีเจอร์ "จองด้วยภาษาธรรมชาติ" โดยใช้ Gemini (@google/genai, โมเดล gemini-2.5-flash)

- ช่องพิมพ์บนหน้าจองห้อง เช่น "ขอจองห้องกลุ่มย่อย A พรุ่งนี้บ่ายสองถึงบ่ายสาม ประชุมทีมวิจัย"
- ส่ง prompt พร้อมรายชื่อห้อง (id + ชื่อ) และวันที่วันนี้ตามเวลาไทย
- บังคับให้ตอบเป็น JSON ด้วย responseSchema:
  { roomId: string, date: 'YYYY-MM-DD', startTime: 'HH:mm', endTime: 'HH:mm', title: string }
- นำผลลัพธ์มา "เติมลงฟอร์ม" เท่านั้น — ห้ามบันทึกอัตโนมัติ
  ผู้ใช้ต้องกดยืนยันเอง และยังต้องผ่าน createBooking ตามปกติ (กติกากันชนห้ามข้าม)
- ถ้า Gemini ตอบไม่ครบหรือ parse ไม่ได้ → แสดง Alert ว่า "ไม่เข้าใจคำสั่ง กรุณากรอกฟอร์มเอง"
- เก็บ API key ผ่าน environment variable ของ AI Studio ห้าม hardcode ในโค้ด
```

ไอเดียอื่นที่ต่อยอดได้: สรุปการใช้งานห้องรายสัปดาห์เป็นภาษาไทย, แนะนำห้องที่เหมาะกับจำนวนคน + อุปกรณ์ที่ต้องการ

---

## 5. Prompt แก้ปัญหาที่เจอบ่อยบน AI Studio

| อาการ | Prompt ที่ใช้แก้ |
|---|---|
| ทำไม่ครบ / ตกฟีเจอร์ | `ตรวจว่าฟีเจอร์ F1–F9 ในข้อกำหนดครบหรือยัง ทำรายการที่ขาดให้ครบ อย่าแก้ส่วนที่ทำงานดีอยู่แล้ว` |
| จองชนกันได้ (บั๊กร้ายแรง) | `createBooking ยังกันชนไม่ได้ ตรวจ 3 จุด: (1) อ่าน localStorage ใหม่ทุกครั้ง ห้ามใช้ React state (2) สูตรต้องเป็น existing.startAt < new.endAt && new.startAt < existing.endAt (3) ตรวจและเขียนต้องอยู่ในฟังก์ชัน sync เดียว ไม่มี await คั่น` |
| จองติดกันแล้วถูกปฏิเสธ | `13:00–15:00 กับ 15:00–16:00 ไม่ถือว่าชน ต้องจองได้ ตรวจว่าใช้ < ไม่ใช่ <=` |
| เวลาเพี้ยนไป 7 ชั่วโมง | `เวลาทั้งระบบต้องเป็น Asia/Bangkok ห้ามใช้ new Date().toISOString() (แปลงเป็น UTC) ให้ประกอบ string 'YYYY-MM-DDTHH:mm:00+07:00' เองและเทียบแบบ string` |
| localStorage เต็ม | `ภาพที่อัปโหลดต้องย่อด้วย canvas ให้กว้างไม่เกิน 1200px และบีบเป็น JPEG คุณภาพ 0.7 ก่อนเก็บ ถ้ายังเกิน 300KB ให้ปฏิเสธพร้อมข้อความบอกขนาด` |
| UI ไม่สม่ำเสมอ | `ตรวจว่าทุกสีมาจาก design token ไม่มี hardcode, รัศมี 8px เท่ากันทุกที่, error/success ใช้ Alert เท่านั้น, สถานะใช้ Badge เท่านั้น แก้จุดที่หลุด` |
| ข้อมูลหายหลังรีเฟรช | `ข้อมูลต้องอ่าน/เขียน localStorage คีย์ smrb.db.v1 ตลอด ห้ามเก็บไว้แค่ใน useState ตรวจว่า seed ทำงานเฉพาะตอน localStorage ว่างเท่านั้น` |

---

## 6. ตารางเทียบ: เอกสารต้นฉบับ → ส่วนไหนของ Prompt

| เอกสารต้นฉบับ | ถูกแปลงไปอยู่ที่ |
|---|---|
| [prd.md](prd.md) §5 (F1–F9) | PROMPT A → "ฟีเจอร์ที่ต้องมีครบ" |
| [prd.md](prd.md) §8 (UI) + [architecture.md](architecture.md) §6 | PROMPT A → "ดีไซน์" |
| [prd.md](prd.md) §11 (Locked Decisions) | PROMPT A → "ค่าคงที่ระบบ" |
| [schema.md](schema.md) §2 (4 ตาราง) | PROMPT A → "โครงสร้างข้อมูล" |
| [schema.md](schema.md) §4–5 (overlap + transaction) | PROMPT A → "หัวใจของระบบ" |
| [schema.md](schema.md) §6 (seed) | PROMPT A → "ข้อมูลตัวอย่าง" |
| [architecture.md](architecture.md) §2 (layers) | PROMPT A → "สถาปัตยกรรมที่ต้องการ" |
| [implementation-plan.md](implementation-plan.md) M0–M7 | PROMPT B (ยุบเหลือ 5 เฟส) |
| [AGENTS.md](AGENTS.md) กฎเหล็ก | PROMPT A → "กฎเหล็ก" 4 ข้อ |
| การทดสอบ concurrency 8 requests (curl) | ❌ ทำไม่ได้บนเบราว์เซอร์ — แทนด้วย "เกณฑ์ตรวจรับ" ข้อ 2–4 |

---

## 7. สิ่งที่หายไปเมื่อย้ายมา AI Studio (ต้องบอกนักเรียนให้ชัด)

| ของต้นฉบับ | บน AI Studio | บทเรียนที่ควรสอน |
|---|---|---|
| DB transaction จริง (`BEGIN IMMEDIATE`) | ฟังก์ชัน sync + optimistic lock | ทำไม atomicity ถึงต้องอยู่ที่ชั้นข้อมูล ไม่ใช่ UI |
| ตรวจสิทธิ์ฝั่ง server | ตรวจฝั่ง client | ทำไม client-side guard ไม่ใช่ security (เปิด DevTools แก้ได้) |
| bcrypt (ช้าโดยตั้งใจ) | SHA-256 (เร็ว) | ทำไม password hash ต้องช้า (กัน brute-force) |
| ข้อมูลบนเซิร์ฟเวอร์ ใช้ร่วมกันได้ | localStorage แยกตามเบราว์เซอร์ | ทำไมระบบจริงต้องมี backend |
| ทดสอบ concurrency ด้วย curl | ทดสอบด้วยการเปิด 2 แท็บ | race condition คืออะไร |

> **แนะนำวิธีสอน:** ใช้ AI Studio สร้างต้นแบบให้เห็นภาพเร็วภายในคาบเดียว
> แล้วชี้ว่า "จุดไหนของต้นแบบนี้ใช้งานจริงไม่ได้ และทำไมต้นฉบับถึงเลือก Next.js + SQLite"
> — จะได้ทั้งความเร็วของ AI Studio และความเข้าใจสถาปัตยกรรมจริง
