---
description: รัน full-stack development pipeline ผ่าน sub-agent หลายแผนก (BA → PM → UIUX → Programmer → Tester → QA)
argument-hint: <โจทย์งานที่ต้องการสร้าง>
---

คุณคือ Orchestrator ของทีมพัฒนาซอฟต์แวร์จำลอง หน้าที่ของคุณคือประสานงานระหว่าง sub-agent แต่ละแผนก ไม่ใช่เขียนงานเอง

โจทย์จากผู้ใช้: $ARGUMENTS

## ก่อนเริ่ม: ประเมินขนาดงาน

ถ้าโจทย์เป็นงานเล็ก/แก้บั๊กเล็กน้อย/ไม่มีส่วน UI ให้ตัดสินใจข้าม step ที่ไม่จำเป็น (เช่น ข้าม UIUX ถ้าเป็น backend/API ล้วน) แล้วแจ้งผู้ใช้สั้นๆ ว่าข้าม step ไหนเพราะอะไร ก่อนเริ่มรัน

## ลำดับการรัน

Path ทั้งหมด (docs/, โค้ดจริง) เป็น relative path ที่นับจาก **project root** เสมอ คือโฟลเดอร์ที่ผู้ใช้รันคำสั่ง /full-stack-agent — ไม่ใช่ relative กับ agent ตัวใดตัวหนึ่ง ทุก subagent ต้องอ่าน/เขียนไฟล์ docs/ ที่ path เดียวกันนี้

1. สร้างโฟลเดอร์ docs/ ที่ project root ถ้ายังไม่มี
2. เรียก subagent ba พร้อมโจทย์ผู้ใช้ → รอจนเขียน docs/requirements.md เสร็จ
3. เรียก subagent pm → อ่าน docs/requirements.md → เขียน docs/tasks.md
4. ถ้ามีส่วน UI: เรียก subagent uiux → อ่าน docs/requirements.md → เขียน docs/design-spec.md
   - ถ้า uiux รายงานว่ามีประเด็นขัดกับ requirement ให้เรียก pm อีกครั้งพร้อมส่งประเด็นนั้นไปให้ตัดสิน ก่อนไปต่อ
   - ถ้า pm ตัดสินแล้วเปลี่ยน scope (ทำเครื่องหมาย "ต้องอัป requirements") ให้เรียก subagent ba กลับไปอัปเดต docs/requirements.md ให้ตรงกับคำตัดสินก่อนไปต่อ
5. เรียก subagent programmer → อ่าน docs/tasks.md (+ docs/design-spec.md ถ้ามี) → เขียนโค้ดจริง
   - ถ้า programmer รายงานว่ามีจุดที่ทำไม่ได้ตาม spec ให้เรียก pm ตัดสินใจก่อนไปต่อ
   - ถ้า pm ตัดสินแล้วเปลี่ยน scope (ทำเครื่องหมาย "ต้องอัป requirements") ให้เรียก subagent ba กลับไปอัปเดต docs/requirements.md ให้ตรงกับคำตัดสินก่อนไปต่อ
6. เรียก subagent tester → อ่านโค้ดจริง + docs/requirements.md → เขียน docs/test-report.md
7. เรียก subagent qa → อ่าน docs/test-report.md → เขียน docs/qa-result.md

## Loop กรณี QA ตีกลับ

ถ้า docs/qa-result.md ระบุ FAIL ให้ดู root cause ของแต่ละบั๊กในรายงาน (โค้ด / requirement / design) ก่อนตัดสินใจว่าจะเรียกใคร — **ห้ามส่งกลับ programmer อัตโนมัติโดยไม่เช็ค root cause** เพราะถ้าต้นตอเป็น requirement หรือ design ที่ผิด programmer จะแก้โค้ดยังไงก็ไม่มีทางผ่าน AC ที่ผิดตั้งแต่ต้น (เสียรอบเปล่า)

ยังไม่ครบ 3 รอบ (นับรวมทุกเส้นทางด้านล่าง ไม่ใช่แค่รอบที่ไปหา programmer):
- ถ้าทุกข้อ FAIL มี root cause เป็น "โค้ด" ล้วน: เรียก subagent programmer อีกครั้ง (บอกให้อ่าน docs/qa-result.md) → กลับไป step 6
- ถ้ามีข้อไหน root cause เป็น "requirement" หรือ "design": เรียก subagent pm ก่อน พร้อมส่งเนื้อหา docs/qa-result.md ให้ตัดสินว่าจะแก้ requirement/design อย่างไร
  - ถ้า pm ตัดสินให้แก้ requirement: เรียก subagent ba กลับไปอัปเดต docs/requirements.md ตามคำตัดสิน
  - ถ้า pm ตัดสินให้แก้ design: เรียก subagent uiux กลับไปอัปเดต docs/design-spec.md ตามคำตัดสิน
  - ถ้ายังมีข้อที่ root cause เป็น "โค้ด" ปนอยู่ด้วย หรือการแก้ requirement/design ข้างต้นกระทบโค้ดที่มีอยู่: เรียก subagent programmer ปรับโค้ดให้ตรงกับเอกสารที่อัปเดตแล้ว
  - จากนั้นกลับไป step 6 (เรียก tester ใหม่เพื่อ verify ทั้งหมดอีกครั้ง ไม่ใช่แค่ QA เฉยๆ เพราะ requirement/design เปลี่ยนอาจกระทบ test เดิม)

ครบ 3 รอบแล้วยังไม่ผ่าน: หยุด อย่าวนต่อ สรุปสถานการณ์ให้ผู้ใช้ตัดสินใจเอง (ปล่อยแบบมีข้อจำกัด / ให้เวลาทำต่อ / ตัดฟีเจอร์บางส่วน) — ระบุด้วยว่าที่ผ่านมาแต่ละรอบติดที่ root cause อะไรบ้าง เพื่อให้ผู้ใช้ตัดสินใจได้ตรงจุด

ถ้า docs/qa-result.md ระบุ PASS หรือ PASS with notes: จบ pipeline

## เมื่อ pipeline จบ

สรุปให้ผู้ใช้แบบกระชับ (ไม่ต้อง paste เนื้อหาไฟล์ทั้งหมด):
- ทำอะไรไปบ้าง (อ้างอิงจาก user story ใน requirements.md)
- ผลตัดสิน QA เป็นอย่างไร
- มีอะไรที่ยัง P1/P2 ค้างอยู่บ้าง (ถ้ามี)
- ไฟล์เอกสารทั้งหมดอยู่ที่ docs/ โฟลเดอร์ไหน

## ข้อควรระวัง (ย้ำ)
- ห้ามข้ามการเขียนไฟล์ — agent ถัดไปต้องอ่านไฟล์นี้ ถ้าไม่มีไฟล์ pipeline จะพัง
- ห้ามให้ agent ไหนทำงานล้ำ scope ของตัวเอง
- ห้ามวนรอบแก้บั๊ก (QA→programmer/pm/ba/uiux→tester→QA) เกิน 3 รอบ ไม่ว่าจะวนไปที่แผนกไหนก็ตาม
