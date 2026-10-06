# Dev Lifecycle Pipeline

จำลองกระบวนการพัฒนาซอฟต์แวร์ตามหลัก SDLC ครบวงจร (BA, PM, Architect, UI/UX, Programmer, Tester, QA, Release) โดยแบ่งเป็น sub-agent แต่ละแผนก ทำงานประสานกันเป็น pipeline ผ่านการส่งต่อไฟล์เอกสาร (File Handoff) ในโปรเจกต์

## 🚀 จุดเด่น (Key Features)

- **File-based Pipeline**: Sub-agent แต่ละตัวทำงานผ่านการอ่าน-เขียนไฟล์ใน `docs/` แทนการคุยสด เพื่อควบคุม context
- **Strict Role Scoping**: กำหนดขอบเขตหน้าที่และข้อห้ามของแต่ละบทบาทชัดเจน ป้องกันการทำงานล้ำหน้าที่ (Role Bleed)
- **Synchronized Requirement & Scope**: เมื่อเกิดข้อขัดแย้งหรือข้อจำกัดทางเทคนิค PM จะเป็นผู้ตัดสินใจ และ Orchestrator จะเรียก BA มาอัปเดต Requirement เสมอ
- **Conditional Architect & Release**: เรียก Architect/Release เฉพาะมุมมองที่เข้าเงื่อนไขของงานจริง (schema, API, external dependency, security, deploy) ไม่รันทุกอย่างแบบ default
- **Circuit Breaker**: วนรอบการแก้บั๊กระหว่าง QA ↔ Programmer สูงสุด 3 รอบ เพื่อป้องกัน loop อนันต์

---

## 👥 บทบาทในทีม (Agents)

| Agent | บทบาท | รับ Input จาก | ส่ง Output ไปที่ |
|---|---|---|---|
| **BA** (`agents/ba.md`) | Business Analyst | Requirement ดิบจากผู้ใช้ | `docs/requirements.md` |
| **PM** (`agents/pm.md`) | Project Manager | `requirements.md` | `docs/tasks.md` |
| **Architect** (`agents/architect.md`) | Technical Architect | `requirements.md` + `tasks.md` | `docs/architecture-spec.md` (ข้ามได้ถ้างานเล็ก) |
| **UI/UX** (`agents/uiux.md`) | UI/UX Designer | `requirements.md` | `docs/design-spec.md` (ข้ามได้ถ้าไม่มี UI) |
| **Programmer** (`agents/programmer.md`) | Full-stack Programmer | `tasks.md` + `design-spec.md` + `architecture-spec.md` | โค้ดจริง + `docs/dev-notes.md` |
| **Tester** (`agents/tester.md`) | Software Tester | โค้ดจริง + `requirements.md` + `architecture-spec.md` | `docs/test-report.md` |
| **QA** (`agents/qa.md`) | Quality Assurance | `test-report.md` + `requirements.md` | `docs/qa-result.md` |
| **Release** (`agents/release.md`) | Release Engineer | `requirements.md` + `qa-result.md` (ต้อง PASS) | `docs/release-plan.md` (ข้ามได้ถ้าไม่ deploy) |

Architect และ Release อ่าน methodology จาก `commands/*-architect.md` (system-design, db, api, resilience, security, deployment, observability) เฉพาะไฟล์ที่เกี่ยวข้องกับงานนั้นจริง

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
 [Architect]  → docs/architecture-spec.md (เฉพาะงานที่กระทบ design)
     │
     ▼
  [UI/UX]     → docs/design-spec.md (ข้ามได้ถ้าเป็น Backend/API)
     │
     ▼
[Programmer]  → เขียนโค้ดจริงตาม spec
     │
     ▼
  [Tester]    → docs/test-report.md (รวม security checklist)
     │
     ▼
   [QA]       → docs/qa-result.md (PASS / FAIL)
     │
  ┌──┴──┐
FAIL   PASS
  │     │
  │     └─► [Release] → docs/release-plan.md (ข้ามได้ถ้าไม่ deploy) → สรุปผลให้ผู้ใช้
  │
  └───────► วนกลับ Programmer (สูงสุด 3 รอบ)
```

---

## 📦 ติดตั้ง

```
/plugin marketplace add phumrapee47/claude-full-stack-agent
/plugin install dev-lifecycle@dev-lifecycle
```

อัปเดตเป็นเวอร์ชันล่าสุด:

```
/plugin marketplace update dev-lifecycle
```

methodology ของ architect ทั้ง 7 ตัวถูก bundle มาใน plugin นี้แล้ว ไม่ต้องติดตั้ง `architect-skills` แยก

---

## 💻 วิธีเรียกใช้งาน

เปิด Claude Code ในโฟลเดอร์โปรเจกต์ของคุณ แล้วพิมพ์:

```
/dev-lifecycle:dev-lifecycle <ระบุโจทย์หรือฟีเจอร์ที่ต้องการสร้าง>
```

(command ที่มาจาก plugin มี prefix เป็นชื่อ plugin เสมอ และ sub-agent จะชื่อ `dev-lifecycle:ba`, `dev-lifecycle:pm` ฯลฯ)

---

## 🗂️ โครงสร้าง repo

```
├── .claude-plugin/marketplace.json
└── plugins/dev-lifecycle/
    ├── .claude-plugin/plugin.json
    ├── agents/                 ← ba, pm, architect, uiux, programmer, tester, qa, release
    ├── commands/dev-lifecycle.md   ← orchestrator
    └── skills/dev-lifecycle/
        ├── SKILL.md
        └── references/         ← methodology ของ *-architect ทั้ง 7 ตัว
```
