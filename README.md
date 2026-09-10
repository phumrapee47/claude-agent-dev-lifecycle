# Claude Full-Stack Agent Pipeline

จำลองทีมพัฒนาซอฟต์แวร์แบบ full-stack (BA, PM, UI/UX, Programmer, Tester, QA) โดยแบ่งเป็น sub-agent แต่ละแผนก ทำงานประสานกันเป็น pipeline ผ่านการส่งต่อไฟล์เอกสาร (File Handoff) ในโปรเจกต์

## 🚀 จุดเด่น (Key Features)

- **File-based Pipeline**: Sub-agent แต่ละตัวทำงานผ่านการอ่าน-เขียนไฟล์ใน `docs/` แทนการคุยสด เพื่อควบคุม context และประหยัด Token
- **Strict Role Scoping**: กำหนดขอบเขตหน้าที่และข้อห้ามของแต่ละบทบาทชัดเจน ป้องกันการทำงานล้ำหน้าที่ (Role Bleed)
- **Synchronized Requirement & Scope**: เมื่อเกิดข้อขัดแย้งหรือข้อจำกัดทางเทคนิค PM จะเป็นผู้ตัดสินใจ และ Orchestrator จะเรียก BA มาอัปเดต Requirement เสมอ
- **Circuit Breaker**: วนรอบการแก้บั๊กระหว่าง QA ↔ Programmer สูงสุด 3 รอบ เพื่อป้องกัน loop อนันต์

---

## 👥 บทบาทในทีม (Agents)

| Agent | บทบาท | รับ Input จาก | ส่ง Output ไปที่ |
|---|---|---|---|
| **BA** (`agents/ba.md`) | Business Analyst | Requirement ดิบจากผู้ใช้ | `docs/requirements.md` |
| **PM** (`agents/pm.md`) | Project Manager | `requirements.md` | `docs/tasks.md` |
| **UI/UX** (`agents/uiux.md`) | UI/UX Designer | `requirements.md` | `docs/design-spec.md` (ข้ามได้ถ้าไม่มี UI) |
| **Programmer** (`agents/programmer.md`) | Full-stack Programmer | `tasks.md` + `design-spec.md` | โค้ดจริง + `docs/dev-notes.md` |
| **Tester** (`agents/tester.md`) | Software Tester | โค้ดจริง + `requirements.md` | `docs/test-report.md` |
| **QA** (`agents/qa.md`) | Quality Assurance | `test-report.md` + `requirements.md` | `docs/qa-result.md` |

---

## 🔄 ลำดับการทำงาน (Workflow)

```text
User Requirement
     │
     ▼
  [BA]        → docs/requirements.md
     │
     ▼
  [PM]        → docs/tasks.md (แตก task P0/P1/P2)
     │
     ▼
  [UI/UX]     → docs/design-spec.md (ข้ามได้ถ้าเป็น Backend/API)
     │
     ▼
[Programmer]  → เขียนโค้ดจริงตาม spec
     │
     ▼
  [Tester]    → docs/test-report.md
     │
     ▼
   [QA]       → docs/qa-result.md (PASS / FAIL)
     │
  ┌──┴──┐
FAIL   PASS
  │     │
  │     └─► จบ pipeline สรุปผลให้ผู้ใช้
  │
  └───────► วนกลับ Programmer (สูงสุด 3 รอบ)
```

---

## 📦 การติดตั้งและการใช้งาน (Installation & Usage)

### วิธีที่ 1: ติดตั้งแบบ Global (ใช้ได้กับทุกโปรเจกต์ในเครื่อง)
คัดลอกไฟล์ใน `.claude/` ไปไว้ที่ `~/.claude/` (Home directory):

```bash
# บน Windows (PowerShell)
Copy-Item -Recurse -Force .claude\* $HOME\.claude\

# บน macOS / Linux
cp -r .claude/* ~/.claude/
```

### วิธีที่ 2: ติดตั้งเฉพาะโปรเจกต์ (Project Workspace)
คัดลอกโฟลเดอร์ `.claude/` ไปวางที่ root ของโปรเจกต์ที่ต้องการใช้งาน

---

## 💻 วิธีเรียกใช้งาน

เปิด Claude Code ในโฟลเดอร์โปรเจกต์ของคุณ แล้วพิมพ์:

```bash
/full-stack-agent <ระบุโจทย์หรือฟีเจอร์ที่ต้องการสร้าง>
```

ตัวอย่าง:
```bash
/full-stack-agent สร้างระบบ Todo List มีระบบ filter สถานะ และบันทึกข้อมูลลง LocalStorage

