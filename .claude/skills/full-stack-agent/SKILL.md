---
name: full-stack-agent
description: จำลองทีมพัฒนาซอฟต์แวร์แบบ full-stack (BA, PM, UIUX, Programmer, Tester, QA) โดยแบ่งเป็น sub-agent แต่ละแผนก ทำงานต่อกันเป็น pipeline ผ่านไฟล์เอกสารกลาง ใช้เมื่อผู้ใช้ต้องการสร้างฟีเจอร์หรือระบบตั้งแต่ requirement จนถึงทดสอบเสร็จ โดยต้องการให้มีการแบ่งงานตามบทบาทแผนกต่างๆ อย่างเป็นระบบ ไม่ใช่ให้ Claude เขียนโค้ดตรงๆ คนเดียว
---

# Full-Stack Agent Pipeline

Skill นี้จำลองทีมพัฒนาซอฟต์แวร์ โดยแบ่งงานเป็น 6 บทบาท ทำงานต่อเนื่องกันแบบ pipeline
สำคัญ: sub-agent แต่ละตัวไม่ได้ "คุยกันสด" — แต่ละตัวรับ input เป็นไฟล์ ทำงาน แล้วเขียน output เป็นไฟล์ ส่งต่อให้ตัวถัดไป
นี่คือการจำลองการ "ส่งเอกสารข้ามแผนก" ไม่ใช่ห้องแชทรวม

## ลำดับ pipeline

```
User requirement
     │
     ▼
  [BA]        → docs/requirements.md
     │
     ▼
  [PM]        → docs/tasks.md  (แตก task + จัดลำดับความสำคัญ)
     │
     ▼
  [UIUX]       → docs/design-spec.md  (ข้ามได้ถ้าไม่มีส่วน UI)
     │
     ▼
      [Programmer]  → เขียนโค้ดจริงตาม tasks.md (+ design-spec.md ถ้ามี)
            │
            ▼
      [Tester]       → docs/test-report.md
            │
            ▼
      [QA]           → docs/qa-result.md (PASS / FAIL + เหตุผล)
            │
      ┌─────┴─────┐
   FAIL         PASS
      │             │
      ▼             ▼
 กลับไป [Programmer]   สรุปผลให้ผู้ใช้
 (สูงสุด 3 รอบ)
```

## กติกาสำคัญ

1. ทุก agent อ่านเฉพาะไฟล์ที่ตัวเองต้องใช้ ห้ามโหลดประวัติทั้งหมดของ pipeline เพื่อประหยัด token
2. ทุก agent เขียนผลลัพธ์เป็นไฟล์ markdown ในโฟลเดอร์ docs/ เสมอ ห้ามตอบลอยๆ ในแชทอย่างเดียว เพราะ agent ถัดไปต้องอ่านไฟล์นี้ต่อ
3. PM มีอำนาจตัดสินใจสุดท้าย เมื่อแผนกขัดแย้งกัน (เช่น UIUX อยากได้ฟีเจอร์ที่ programmer ทำไม่ทันเวลา)
4. QA ↔ Programmer วนได้สูงสุด 3 รอบ ถ้ายังไม่ผ่าน ให้หยุดแล้วรายงานผู้ใช้ให้ตัดสินใจเอง ห้ามวนไม่จำกัด
5. งานเล็ก/ง่าย ให้ข้าม step ที่ไม่จำเป็นได้ เช่น ถ้าเป็นแค่แก้บั๊กเล็กน้อย ไม่ต้องเรียก BA/UIUX ใหม่ทั้งชุด — ให้ orchestrator (ดู commands/full-stack-agent.md) เป็นคนตัดสินใจว่าจะรันเต็ม pipeline หรือย่อ
6. ถ้าคำตัดสินของ PM (กรณีแผนกขัดแย้งกัน หรือ programmer ทำไม่ได้ตาม spec) ทำให้ scope เปลี่ยนไปจาก docs/requirements.md เดิม ต้องเรียก BA กลับมาอัปเดต docs/requirements.md ให้ตรงกับคำตัดสินเสมอ ห้ามปล่อยให้ requirement doc กับสิ่งที่ implement จริงไม่ตรงกัน
7. โฟลเดอร์ docs/ (และไฟล์ทั้งหมดที่อ้างถึงในไฟล์นี้ เช่น docs/requirements.md) อยู่ที่ project root เสมอ (โฟลเดอร์ที่ orchestrator ถูกเรียกใช้งาน/cwd ของ session) ไม่ใช่ relative กับตำแหน่งของ agent แต่ละตัว

## การเรียกใช้งาน

ผู้ใช้พิมพ์ /full-stack-agent <โจทย์งาน> — ดูรายละเอียดการรันใน commands/full-stack-agent.md

รายละเอียดหน้าที่ของแต่ละแผนกอยู่ใน agents/*.md — อ่านไฟล์ของ agent นั้นก่อนเรียกใช้งานเสมอ เพื่อให้แน่ใจว่าทำตาม scope ที่กำหนดไว้ ไม่ทำงานล้ำแผนกอื่น

| ไฟล์ | บทบาท | รับ input จาก | ส่ง output ไปที่ |
|---|---|---|---|
| agents/ba.md | Business Analyst | โจทย์ผู้ใช้ | docs/requirements.md |
| agents/pm.md | Project Manager | requirements.md | docs/tasks.md |
| agents/uiux.md | UI/UX Designer | requirements.md | docs/design-spec.md |
| agents/programmer.md | Full-stack Programmer | tasks.md + design-spec.md (+ qa-result.md ถ้ามีรอบแก้บั๊ก) | โค้ดจริง + docs/dev-notes.md |
| agents/tester.md | Tester | โค้ดจริง + requirements.md | docs/test-report.md |
| agents/qa.md | QA | test-report.md + requirements.md | docs/qa-result.md |
