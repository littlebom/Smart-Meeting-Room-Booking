# PROMPTING.md — คู่มือสั่ง Prompt สำหรับ Vibe Coding

> วิธีใช้เอกสารชุดนี้สร้างระบบขึ้นใหม่ด้วย AI (Claude Code / Cursor / Windsurf ฯลฯ)
> Prompt ทุกอันด้านล่าง **copy-paste ได้เลย** — แก้เฉพาะชื่อ milestone

---

## หลักการ 5 ข้อ (อ่านก่อนเริ่ม)

1. **ทีละ milestone** — อย่าสั่ง "สร้างทั้งระบบ" รวดเดียว สั่ง M0 → ตรวจ → M1 → ตรวจ ...
2. **ให้ AI ตรวจงานจริงทุกขั้น** — typecheck + ยิง endpoint จริง ไม่ใช่แค่ "น่าจะใช้ได้"
3. **ให้ AGENTS.md เป็นกฎ** — ทุก prompt อ้างอิงกติกาในนั้น เพื่อกันโค้ดหลุดแนว
4. **commit ทุก milestone** — ย้อนกลับได้ + เห็นพัฒนาการ
5. **เจอ error → ส่งให้ AI พร้อมบอกให้ดู gotchas** — อย่าแก้มั่ว

---

## STEP 0 — เปิดงาน (Bootstrap Prompt)

> ใช้ครั้งเดียวตอนเริ่ม เพื่อให้ AI อ่าน context ทั้งหมดก่อน

```
ฉันจะสร้างระบบจองห้องประชุมด้วยเทคนิค vibe coding
ในโปรเจกต์มีเอกสาร 6 ไฟล์: prd.md, AGENTS.md, implementation-plan.md,
architecture.md, schema.md, progress.md

งานของคุณ:
1) อ่านทั้ง 6 ไฟล์
2) สรุปให้ฉันสั้น ๆ ว่า: เราจะสร้างอะไร, tech stack, ลำดับ milestone M0–M7,
   และ "กฎเหล็ก" จาก AGENTS.md ที่ห้ามพลาด
3) ยังไม่ต้องเขียนโค้ด — รอฉันสั่งทีละ milestone

ยึด AGENTS.md เป็นกติกาการทำงานตลอดทั้งโปรเจกต์
```

---

## STEP 1 — สั่งทำทีละ Milestone (Template)

> เปลี่ยน `M0` เป็น milestone ที่ต้องการ ใช้ซ้ำได้ทุกขั้น

```
ทำ Milestone M0 ตาม implementation-plan.md

ข้อกำหนด:
- ทำตาม convention และกฎเหล็กใน AGENTS.md อย่างเคร่งครัด
- ใช้ schema.md เป็นแหล่งอ้างอิงโครงสร้างฐานข้อมูล
- ใช้ architecture.md ดูว่าแต่ละชั้นต่อกันอย่างไร
- เมื่อเขียนเสร็จ ให้รันส่วน "✅ ตรวจ" ของ M0 จริง แล้วแปะผลลัพธ์ให้ฉันดู
- อัปเดต progress.md (กาเช็ก M0 + verification log)

อย่าทำ milestone ถัดไปจนกว่าฉันจะสั่ง
```

---

## STEP 2 — Prompt เฉพาะแต่ละ Milestone (จุดเน้น)

> ใช้เสริม template ข้างบนเมื่ออยากเน้นจุดสำคัญของ milestone นั้น

**M1 (Auth):**
```
ทำ M1 ตาม implementation-plan.md
เน้น: guard ต้องทำฝั่ง server (requirePageUser/Admin ที่ layout),
session ต้องเป็น signed cookie (HMAC) ตาม architecture.md §4
ตรวจ: login ถูก/ผิด (200/401) และบัญชีปิด (403)
```

**M3 (Booking Core) — สำคัญสุด:**
```
ทำ M3 ตาม implementation-plan.md
เน้น: การจองต้องใช้ createBookingTx (overlap check + INSERT ใน transaction เดียว)
ตาม schema.md §5 — ห้ามเขียน INSERT booking นอก transaction

บังคับทดสอบก่อนบอกว่าเสร็จ:
- overlap → 409, adjacent → 201, นอกเวลา → 400
- Concurrency: ยิง 8 request พร้อมกัน slot เดียวกัน ต้องได้สำเร็จ 1 / ปฏิเสธ 7
แปะผล curl จริงให้ฉันดู
```

**M6 (User Management):**
```
ทำ M6 ตาม implementation-plan.md
เน้น safety guards: ต้องมี admin ใช้งานได้ ≥ 1 คน, ลบตัวเองไม่ได้,
ลบผู้ใช้ที่มีประวัติจองไม่ได้, ปิดบัญชีแล้ว login ไม่ได้
ตรวจครบทุกเคสด้วย curl
```

---

## STEP 3 — Prompt ตรวจงาน (Verification)

> ใช้เมื่อ AI บอกว่าเสร็จแต่ยังไม่ได้พิสูจน์

```
ก่อนปิดงานนี้ ให้พิสูจน์ว่าใช้ได้จริง:
1) รัน npx tsc --noEmit (ต้องไม่มี error)
2) ยิง endpoint ที่เพิ่งทำด้วย curl จริง แล้วแปะ request + response + status code
ถ้าไม่ผ่าน ให้แก้จนผ่าน อย่ารายงานว่าเสร็จถ้ายังไม่ได้ทดสอบ
```

---

## STEP 4 — Prompt ดึงกลับเข้าแนว (Drift Correction)

> ใช้เมื่อ AI เขียนหลุดสไตล์ (เช่น hardcode สี, query นอก data layer)

```
ตรวจว่าโค้ดที่เพิ่งเขียนตรงกับ AGENTS.md §3 (กฎเหล็ก) และ §5 (conventions) ไหม
โดยเฉพาะ: ตรวจสิทธิ์ฝั่ง server, เขียน DB ผ่าน lib/db เท่านั้น,
ใช้ design token แทน hardcode สี, validate ด้วย Zod
ถ้ามีจุดไหนไม่ตรง ให้แก้ให้ตรงทั้งหมด
```

---

## STEP 5 — Prompt แก้ Error

> แปะ error เต็ม ๆ แล้วให้ AI เทียบกับ gotchas

```
ฉันเจอ error นี้:
<วาง error เต็ม ๆ ตรงนี้>

ดูตาราง Gotchas ใน AGENTS.md §8 ว่าตรงกับเคสไหน แล้วแก้ตามนั้น
อธิบายสาเหตุสั้น ๆ ก่อนแก้
```

ตัวอย่างที่เจอบ่อย: `Cannot find module './xxx.js'` = `.next` cache เพี้ยน → ปิด dev, `rm -rf .next`, dev ใหม่ (**อย่า build ขณะ dev รัน**)

---

## STEP 6 — Prompt ปิดท้ายแต่ละ Milestone

```
สรุปสิ่งที่ทำใน milestone นี้: ไฟล์ที่สร้าง/แก้, ผลทดสอบ, และอัปเดต progress.md
จากนั้นช่วยร่างข้อความ commit สั้น ๆ (เป็นภาษาอังกฤษ) สำหรับ milestone นี้
```

แล้วนักเรียน commit เอง:
```bash
git add -A && git commit -m "M3: booking core with conflict prevention"
```

---

## ลำดับ Prompt ทั้งหมด (สรุป 1 หน้า)

```
STEP 0  Bootstrap (ครั้งเดียว)
─ loop ต่อ milestone M0..M7 ─
STEP 1  สั่งทำ Mx
STEP 3  ให้ตรวจงานจริง
(STEP 4 ถ้าหลุดแนว / STEP 5 ถ้า error)
STEP 6  สรุป + commit
─ จบ loop ─
```

---

## เคล็ดลับสำหรับผู้สอน

- **ให้นักเรียนอ่าน error เอง** ก่อนโยนให้ AI — ฝึกดีบั๊ก
- **หยุดที่จุดตรวจ** — ถ้า concurrency test (M3) ไม่ผ่าน ห้ามไป M4 เป็นบทเรียนเรื่อง atomicity
- **เทียบกับเฉลย** — เก็บโค้ดต้นฉบับไว้คนละ branch (เช่น `solution`) ให้นักเรียนเทียบเมื่อจบ
- **เน้นว่า AGENTS.md คือสิ่งที่ทำให้ผลลัพธ์เหมือนกัน** — ถ้าไม่มีไฟล์นี้ นักเรียนแต่ละคนจะได้โค้ดคนละแบบ
