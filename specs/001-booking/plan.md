# แผนเทคนิค: จองคิวตรวจสุขภาพ (Booking)

## 1. สรุปแนวทาง
ฟีเจอร์นี้ให้ผู้รับบริการที่ยืนยันตัวตนแล้วเลือกแพ็กเกจ วัน และช่วงเวลาตรวจสุขภาพ เพื่อรับหมายเลขคิวและยืนยันการจองภายใน 3 นาที ตาม FR-BKG-01 ถึง FR-BKG-06 การทำงานหลักจะอยู่ที่หน้าเว็บสำหรับเลือกวันที่และช่วงเวลา, API ดึงข้อมูล slot และสร้าง booking, และ worker สำหรับส่งข้อความยืนยันแบบ asynchronous ตาม IF-NOT-01 ระบบต้องป้องกันการจองซ้ำในวันเดียวกันและต้องไม่แสดงผลที่อาจทำให้มีการจองซ้อนเกิดขึ้นเมื่อ slot เต็ม

## 2. เทคโนโลยีที่ใช้

| สิ่งที่เลือก | มาจาก | หมายเหตุ |
|---|---|---|
| React + Vite | ทีมเลือกเอง ไม่ได้มาจาก spec | สำหรับหน้าเลือกแพ็กเกจ วัน และช่วงเวลาว่าง |
| Python FastAPI | ทีมเลือกเอง ไม่ได้มาจาก spec | สำหรับ API จองคิวและค้นหาช่วงเวลาว่าง |
| MySQL | CON-TECH-01 | เก็บข้อมูลการจองและข้อมูล slot ตามโครงสร้างที่กำหนด |
| JWT / session validation | IF-IDP-01 | ตรวจว่าเคยยืนยันตัวตนแล้วก่อนใช้งานฟีเจอร์ |
| HIS lookup by CID -> HN | IF-HIS-01 | ใช้เพื่อค้นหาผู้รับบริการและแสดงต่อระบบภายใน โดยไม่เก็บเลขบัตรประชาชนในตารางการจอง |
| SMS/LINE async notifier | IF-NOT-01 | ส่งข้อความยืนยันแบบ asynchronous และไม่ให้การจองรอผลส่งข้อความ |
| Audit log store | DOM-PDPA-01 | บันทึกผู้เข้าถึง เวลา และรหัสผู้รับบริการ สำหรับการเข้าถึงข้อมูลสุขภาพ |

## 3. โมเดลข้อมูล

### 3.1 Entity ที่ต้องมี

| Entity | ฟิลด์หลัก | รองรับ FR/Constraint |
|---|---|---|
| user_profile | user_id, hn, identity_status, last_login_at | IF-IDP-01, FR-BKG-02 |
| booking | booking_id, user_id, hn, package_id, booking_date, slot_id, queue_no, status, created_at | FR-BKG-02, FR-BKG-04, FR-BKG-05, DOM-PDPA-01 |
| slot_template | slot_id, date, time_start, time_end, package_id, quota_total | FR-BKG-01, FR-BKG-03, FR-BKG-06 |
| booking_snapshot | booking_id, slot_id, quota_before, quota_after, version_no, updated_at | FR-BKG-04, AC-BKG-01, AC-BKG-03 |
| notification_queue | message_id, booking_id, channel, payload, status, retry_count, next_retry_at | FR-BKG-05, NFR-REL-02, IF-NOT-01 |
| audit_log | log_id, access_user, accessed_at, patient_hn, action_type | DOM-PDPA-01 |

### 3.2 กฎข้อมูลที่ต้องเห็นชัด
- ตารางการจองต้องไม่มีฟิลด์เลขบัตรประชาชน ตาม IF-HIS-01
- การยืนยันตัวตนต้องเป็น precondition ก่อนเข้าถึงข้อมูลผู้รับบริการ ตาม IF-IDP-01
- audit log เก็บข้อมูลผู้เข้าถึง เวลา และ HN อย่างน้อย 1 ปี ตาม DOM-PDPA-01

## 4. API / หน้าจอ

### 4.1 หน้าจอ
- Page: Booking selection
  - Input: package_id, selected_date, selected_slot_id
  - Output: list of available dates and slots with remaining quota
  - รองรับ: FR-BKG-01, FR-BKG-06
- Page: Booking confirmation
  - Input: booking payload
  - Output: queue number, success state, notification queued state
  - รองรับ: FR-BKG-04, FR-BKG-05
- Page: Booking denial notice
  - Input: duplicate active queue on same day
  - Output: reject message + existing queue number
  - รองรับ: FR-BKG-02
- Page: Slot full warning
  - Input: chosen slot becomes occupied during confirmation
  - Output: alert + 3 alternative slots
  - รองรับ: FR-BKG-03

### 4.2 Endpoint ที่คาดว่าจะมี
- GET /api/v1/booking/slots?date_from=&date_to=&package_id=
  - Output: list of dates and slots with remaining quota
  - รองรับ: FR-BKG-01, FR-BKG-06
- POST /api/v1/booking/validate
  - Input: user_id, package_id, date, slot_id
  - Output: valid / duplicate / full_slot / alternative_slots
  - รองรับ: FR-BKG-02, FR-BKG-03
- POST /api/v1/booking
  - Input: user_id, hn, package_id, date, slot_id
  - Output: booking_id, queue_no, notification_status
  - รองรับ: FR-BKG-04
- POST /api/v1/booking/notifications/retry
  - Input: message_id
  - Output: retry accepted
  - รองรับ: FR-BKG-05, NFR-REL-02
- GET /api/v1/audit-log
  - Input: user_id, patient_hn
  - Output: audit entries
  - รองรับ: DOM-PDPA-01

## 5. ตารางตรวจ Constraints

| Constraint ID | ถูกนำไปใช้ที่ไหนใน plan | สถานะ |
|---|---|---|
| CON-TECH-01 | MySQL ใช้เป็น datastore สำหรับ booking, slot, notification_queue | ใช้แล้ว |
| DOM-PDPA-01 | audit_log entity และ API /api/v1/audit-log; บันทึกผู้เข้าถึง เวลา และ HN อย่างน้อย 1 ปี | ใช้แล้ว |
| IF-IDP-01 | booking API และ page selection ต้องตรวจ precondition ว่ายืนยันตัวตนแล้วก่อนอนุญาตเข้าถึงข้อมูลผู้รับบริการ | ใช้แล้ว |
| IF-HIS-01 | user_profile / booking ใช้ HN แทนเลขบัตรประชาชน และไม่มีฟิลด์ CID ในตารางการจอง | ใช้แล้ว |
| IF-NOT-01 | notification_queue + async notifier; การจองไม่รอผลส่งข้อความ | ใช้แล้ว |

## 6. แผนทดสอบจาก Acceptance Criteria

| AC ID | ชื่อ test | ทดสอบอย่างไร |
|---|---|---|
| AC-BKG-01 | test_AC_BKG_01_booking_success_reduces_quota | ตั้ง slot 09.00 quota=1, ทำ booking สำเร็จ, ตรวจว่าบันทึกสร้าง booking, queue_no แสดงผล, quotaลดเหลือ 0 |
| AC-BKG-02 | test_AC_BKG_02_reject_duplicate_same_day | ตั้งผู้รับบริการมีคิวยังไม่ได้ใช้ในวันเดียวกัน, พยายามจองอีกครั้ง, ตรวจว่าปฏิเสธและแสดง queue_no เดิม |
| AC-BKG-03 | test_AC_BKG_03_slot_full_shows_three_alternatives | ตั้ง slot 09.00 quota=1 และมีผู้ใช้คนอื่นยืนยันก่อน, ทำ booking ของคนถัดไป, ตรวจว่าระบบแจ้ง “ช่วงเวลาเต็ม” และแสดง 3 ตัวเลือกที่ใกล้ที่สุด |
| AC-BKG-04 | test_AC_BKG_04_notification_failure_keeps_booking | จำลอง SMS/LINE ส่งไม่สำเร็จ, ตรวจว่าการจองยังถูกบันทึก, queue_no แสดงผล, และมี item ใน notification_queue ที่ต้อง retry ภายใน 5 นาที |
| AC-BKG-05 | test_AC_BKG_05_search_slots_p95_under_2s | ใช้การจำลอง 200 concurrent requests, ตรวจ p95 response time <= 2s |
| AC-BKG-06 | test_AC_BKG_06_audit_log_generated | เปิดดูข้อมูลการจองของผู้รับบริการ, ตรวจว่า audit_log มีผู้เข้าถึง เวลา และ HN | 

## 7. ลำดับงาน
1. สร้าง schema MySQL สำหรับ slot, booking, notification_queue, audit_log และกำหนด index ที่จำเป็น (FR-BKG-01, FR-BKG-04, DOM-PDPA-01)
2. สร้าง API ดึง slot ว่างพร้อมจำนวนที่นั่งคงเหลือแบบรวมข้อมูล package/date/slot (FR-BKG-01, FR-BKG-06)
3. สร้าง validation สำหรับ duplicate same-day queue และ slot-full detection พร้อม alternative slots (FR-BKG-02, FR-BKG-03)
4. สร้าง booking flow สำหรับบันทึกการจอง, ออกหมายเลขคิว, และลด quota ทันที (FR-BKG-04, AC-BKG-01)
5. สร้าง notification_queue และ worker สำหรับ retry ภายใน 5 นาที เมื่อส่งข้อความไม่สำเร็จ (FR-BKG-05, NFR-REL-02, IF-NOT-01)
6. เพิ่ม audit log สำหรับการเข้าถึงข้อมูลผู้รับบริการ (DOM-PDPA-01, AC-BKG-06)
7. ทดสอบครบตาม AC-BKG-01 ถึง AC-BKG-06 พร้อมตั้งค่า load test สำหรับ NFR-PERF-01

## 8. สิ่งที่ยังไม่ทำ
- ไม่มี Open Questions ใน Spec v2 เนื่องจากคำถามเดิมได้ตัดสินใจเป็น ASM-03 ถึง ASM-06 แล้ว
- ส่วนที่ยังจะต้องรอคำตอบจากทีมสำหรับการพัฒนาต่อคือ: หากระบบแจ้งเตือนมีความล้มเหลวแบบอื่นนอกเหนือจาก timeout/connection error/reject จะถือเป็นกรณี retry หรือไม่ (ยังไม่มีข้อความกำหนดเป็นตัวชี้ชัดใน spec)
- หากทีมต้องการให้มีหลักเกณฑ์ “มากกว่า 3 ตัวเลือก” ในหลักคำนวณ alternative slots จะต้องกลับมาดูความต้องการใหม่อีกครั้ง

## 9. ประเด็นพัฒนาที่ควรระวัง
- ต้องใช้ timezone Asia/Bangkok ในทุกการคำนวณ date และ same-day validation ตาม ASM-02
- การส่งข้อความและ audit log จะต้องแยกเป็น async pipeline เพื่อไม่ให้การจองติดชะลอ
- การคำนวณ quota และ queue number ต้องเป็น atomic transaction เพื่อป้องกัน race condition เมื่อมีผู้ใช้พร้อมกันมาก
