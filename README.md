# 🏢 Smart Meeting Room Booking — Vibe Coding Teaching Kit

> ชุดเอกสารสำหรับ**สอนสร้างระบบจริงด้วยเทคนิค Vibe Coding** (เขียนโปรแกรมร่วมกับ AI)
> นักเรียนใช้เอกสารชุดนี้ป้อนให้ AI แล้วสร้างระบบจองห้องประชุมขึ้นใหม่ทีละขั้น จนได้ระบบที่ทำงานได้จริง

ระบบเป้าหมาย: **Next.js 14 + TypeScript + Tailwind + SQLite** — เน้นการกันจองชนกัน (double booking) ระดับฐานข้อมูล, จัดการห้อง/ภาพ/ผู้ใช้, UI แบบ Clean & Modern

---

## 📚 เอกสารในชุดนี้

| ไฟล์ | ใช้ตอบคำถาม | อ่านเมื่อ |
|---|---|---|
| [prd.md](prd.md) | จะสร้าง **อะไร** และ **ทำไม** | เริ่มต้น เข้าใจภาพรวม |
| [AGENTS.md](AGENTS.md) | **กติกา** ที่ AI ต้องทำตาม | ป้อนให้ AI ทุกครั้ง (สำคัญสุด) |
| [implementation-plan.md](implementation-plan.md) | สร้าง **ตามลำดับ** อย่างไร (M0–M7) | ระหว่างลงมือ |
| [architecture.md](architecture.md) | ระบบ **ต่อกัน** อย่างไร | เข้าใจ flow / ออกแบบ |
| [schema.md](schema.md) | **ฐานข้อมูล** หน้าตาอย่างไร | สร้าง data layer |
| [progress.md](progress.md) | ทำไป **ถึงไหนแล้ว** | ติดตามความคืบหน้า |
| [PROMPTING.md](PROMPTING.md) | **สั่ง AI** อย่างไร | คู่มือ prompt copy-paste |

---

## 🚀 เริ่มต้นใช้สอน (3 ขั้น)

1. **เปิดเครื่องมือ AI coding** (Claude Code, Cursor, Windsurf ฯลฯ) ในโฟลเดอร์ที่มีเอกสารชุดนี้
2. **ป้อน Bootstrap Prompt** จาก [PROMPTING.md](PROMPTING.md) STEP 0
3. **สั่งทีละ milestone** M0 → M7 (ตรวจงานทุกขั้นก่อนไปต่อ)

> หัวใจ: ป้อน **AGENTS.md** ให้ AI เสมอ — เป็นสิ่งที่ทำให้นักเรียนแต่ละคนได้ผลลัพธ์ตรงแนวเดียวกัน

---

## ▶️ รันระบบ (เมื่อสร้างเสร็จ / ดูเฉลย)

```bash
npm install
npm run db:migrate && npm run db:seed
npm run dev          # http://localhost:3000 (หรือ 3001 ถ้าพอร์ตไม่ว่าง)
```

**บัญชีทดลอง:** `jira / password123` (admin) · `staff / password123` (user)

---

## 🧩 จะได้เรียนรู้อะไร

- **Concurrency & Atomicity** — กันจองชนกันด้วย SQL transaction (`BEGIN IMMEDIATE`)
- **Auth & Authorization** — session, role guard ฝั่ง server, การปิดบัญชี
- **Data layer pattern** — แยก DB ออกจาก UI เพื่อเปลี่ยน engine ได้
- **Design system** — token + primitive เพื่อ UI ที่สม่ำเสมอ
- **การทำงานกับ AI** — สั่งทีละขั้น, ตรวจงานจริง, ดึงกลับเข้าแนว, แก้ error

---

## 🛠️ Tech Stack

`Next.js 14 (App Router)` · `TypeScript` · `Tailwind CSS` · `SQLite (better-sqlite3)` · `Zod` · `bcryptjs`

---

## 📁 โครงสร้าง (เมื่อสร้างเสร็จ)

```
app/        หน้าเว็บ (route groups: auth / app / admin) + API routes
components/ UI primitives + feature components
lib/        db (data layer) · auth · booking logic · validation · constants
data/       app.db (SQLite — สร้างจาก migrate)
public/uploads/  ไฟล์ภาพห้อง
```

---

## 📝 หมายเหตุการ Deploy

ระบบออกแบบให้รันเป็น **single Node process บนเครื่องที่มี persistent disk** (เช่น VM / คอนเทนเนอร์เดียว)
ไม่เหมาะ serverless แบบ multi-instance — เหตุผลดู [architecture.md](architecture.md) §7

---

*ชุดเอกสารนี้ถอดแบบจากระบบที่สร้างเสร็จและทดสอบแล้ว — ใช้สร้างขึ้นใหม่ให้เหมือนต้นฉบับได้*
