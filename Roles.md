# สรุปโครงสร้างสิทธิ์และการทำงานของระบบนัดหมายและบริการสุขภาพ

## 1. ผู้ใช้งานในระบบ (Actors)

ระบบนี้มี **4 บทบาทหลัก** โดย 3 บทบาทแรกเป็น Staff (ใช้ User model) และอีก 1 เป็น Patient (ใช้ Patient model แยกกัน):

| # | Role (DB) | Token Ability | Model | คำอธิบาย |
|---|---|---|---|---|
| 1 | `SUPER_ADMIN` | `role:staff` | User | ผู้ดูแลระบบระดับจังหวัด (อบจ./สสจ.) — cross-tenant access |
| 2 | `HEALTH_CENTER_ADMIN` | `role:staff` | User | ผู้ดูแลระบบประจำ รพ.สต. — scope: ศูนย์ของตนเอง |
| 3 | `STAFF` | `role:staff` | User | เจ้าหน้าที่ห้องตรวจ / หมอ / ผู้ช่วย — scope: ศูนย์ของตนเอง |
| 4 | *(ไม่มี DB role)* | `role:patient` | Patient | ผู้ป่วย / ผู้รับบริการ |

**หมายเหตุ:** ทั้ง 3 Staff role ใช้ token ability เดียวกัน (`role:staff`) การแยกสิทธิ์ทำโดยตรวจสอบ DB roles ผ่าน middleware `role:` เท่านั้น — token ability ไม่ได้แยกระดับ

---

## 2. Patient (ผู้ป่วย / ผู้รับบริการ)

### 2.1 การลงทะเบียนและเข้าสู่ระบบ

| Method | Endpoint | Middleware | คำอธิบาย |
|---|---|---|---|
| `POST` | `/v1/patient/register` | `throttle:patient-register` | ลงทะเบียนด้วย CID, เบอร์โทร, วันเกิด — ห้าม CID/เบอร์ซ้ำ |
| `POST` | `/v1/patient/login` | `throttle:patient-login` | เข้าสู่ระบบด้วย เบอร์โทร+วันเกิด — lockout 30 นาทีหลัง 5 ครั้งผิด |
| `GET` | `/v1/patient/me` | `auth:sanctum`, `abilities:role:patient` | ดูข้อมูลส่วนตัว (masked) |

### 2.2 ค้นหาบริการและสถานที่ (Public — ไม่ต้องล็อกอิน)

| Method | Endpoint | คำอธิบาย |
|---|---|---|
| `GET` | `/v1/patient/categories` | รายการหมวดหมู่การให้บริการ |
| `GET` | `/v1/patient/right-types` | รายการสิทธิการรักษา (dropdown) |
| `POST` | `/v1/patient/nearby-centers` | ค้นหา รพ.สต. ใกล้เคียง (เฉพาะศูนย์ ACTIVE) |
| `GET` | `/v1/patient/health-centers/{id}` | รายละเอียด รพ.สต. |
| `GET` | `/v1/patient/health-centers/{id}/services` | รายการบริการของ รพ.สต. (เฉพาะ Active) |
| `GET` | `/v1/patient/available-slots` | ตรวจสอบช่วงเวลาว่างและโควตา |

### 2.3 การนัดหมาย (Booking)

| Method | Endpoint | คำอธิบาย |
|---|---|---|
| `POST` | `/v1/patient/book` | จองคิวนัดหมาย — ระบบออกเลขคิว Sequential Queue |
| `GET` | `/v1/patient/services/{id}/staff` | รายชื่อหมอที่เลือกได้ (เฉพาะบริการ `allow_staff_selection=true`) |

**เงื่อนไขการจอง:** ห้ามจองซ้ำวันเดียวกัน, ห้ามจองถ้าศูนย์ CLOSING/INACTIVE, ห้ามจองวันหยุดบริการ, ห้ามจองถ้าหมอทั้งหมดลางานวันนั้น

**บริการที่เลือกหมอได้:** ต้องส่ง `staff_id` — หมอต้อง ACTIVE, ไม่ลา, ผูกกับบริการ, ยังไม่เต็มคิว (1 คิว/รอบเวลา/คน)

### 2.4 ประวัติและการจัดการนัดหมาย

| Method | Endpoint | คำอธิบาย |
|---|---|---|
| `GET` | `/v1/patient/appointments` | ดูรายการนัดหมายของตนเอง (mask ข้อมูล) |
| `GET` | `/v1/patient/appointments/{id}` | ดูรายละเอียดนัดหมาย |
| `PATCH` | `/v1/patient/appointments/{id}/cancel` | ยกเลิกนัดหมาย — ต้องเป็น CONFIRMED + ยังไม่ถึงเวลา slot |

**ข้อจำกัด:** ไม่สามารถยกเลิกคิวที่ยกเลิกไปแล้ว หรือ COMPLETED ได้; ไม่สามารถแอบดูนัดหมายของผู้ป่วยคนอื่น

---

## 3. Staff (เจ้าหน้าที่ / หมอ / ผู้ให้บริการ)

> **คำสำคัญ:** Staff ทุกคนถูกจำกัด scope ด้วย `health_center_id` ของตัวเอง (ยกเว้น SUPER_ADMIN ที่ต้องระบุ `health_center_id` ทุกครั้งที่ทำรายการ write)

### 3.1 การเข้าสู่ระบบและโปรไฟล์

| Method | Endpoint | Roles ที่เข้าถึงได้ | คำอธิบาย |
|---|---|---|---|
| `POST` | `/v1/staff/login` | ทุกคน (Public) | เข้าสู่ระบบด้วย username+password — throttle 5 ครั้ง/30 นาที |
| `GET` | `/v1/staff/me` | ทุกคน (auth:sanctum) | ดูข้อมูลโปรไฟล์ตัวเอง |
| `PATCH` | `/v1/staff/me` | ทุกคน (auth:sanctum) | แก้ไขชื่อ/รหัสผ่านตัวเอง |
| `POST` | `/v1/staff/logout` | ทุกคน (auth:sanctum) | ออกจากระบบ (ลบ token) |

**ข้อจำกัด:** Login ได้ **เฉพาะศูนย์ที่มีสถานะ ACTIVE** — ทั้ง CLOSING และ INACTIVE บล็อก Login (403); ผู้ใช้ที่มี token ค้างอยู่จากก่อนปิดศูนย์ยังใช้งาน endpoint ต่อไปได้จนกว่า token จะหมดอายุ (24 ชม.)

### 3.2 Dashboard

| Method | Endpoint | Roles ที่เข้าถึงได้ | คำอธิบาย |
|---|---|---|---|
| `GET` | `/v1/staff/dashboard/summary` | STAFF, HC_ADMIN, SUPER_ADMIN | สถิตินัดหมายประจำวัน — SUPER_ADMIN ไม่ระบุ `health_center_id` = รวมทุกศูนย์ |

### 3.3 การจัดการบริการ (Services)

| Method | Endpoint | Roles ที่เข้าถึงได้ | คำอธิบาย |
|---|---|---|---|
| `GET` | `/v1/staff/services` | STAFF, HC_ADMIN, SUPER_ADMIN | ดูรายการบริการ — STAFF เห็นเฉพาะศูนย์ตัวเอง; SUPER_ADMIN เห็นทุกศูนย์; รองรับ `?date=` ซ่อนบริการที่ staff มอบหมายลางานทั้งหมด |
| `POST` | `/v1/staff/services` | STAFF, HC_ADMIN, SUPER_ADMIN | เพิ่มบริการใหม่ในศูนย์ |
| `PUT` | `/v1/staff/services/{id}` | STAFF, HC_ADMIN, SUPER_ADMIN | แก้ไขข้อมูลบริการ |
| `PATCH` | `/v1/staff/services/{id}/toggle-status` | STAFF, HC_ADMIN, SUPER_ADMIN | เปิด/ปิดสถานะบริการ |
| `DELETE` | `/v1/staff/services/{id}` | **HC_ADMIN, SUPER_ADMIN เท่านั้น** | ลบบริการ — ห้ามลบถ้ามีนัดหมาย CONFIRMED ในอนาคต |
| `PUT` | `/v1/staff/services/{id}/assignees` | STAFF, HC_ADMIN, SUPER_ADMIN | ผูก User (Staff role) เข้ากับบริการ (หน้าที่รับผิดชอบ) — HC_ADMIN จำกัดศูนย์ตัวเอง |
| `PUT` | `/v1/staff/services/{id}/staff` | **HC_ADMIN, SUPER_ADMIN เท่านั้น** | ผูกหมอที่เลือกได้ (selectable staff) เข้ากับบริการ — ห้ามแกะหมอที่มีคิว CONFIRMED ในอนาคต; บันทึก Audit Log |
| `PUT` | `/v1/staff/services/{id}/time-slot-days` | **HC_ADMIN, SUPER_ADMIN เท่านั้น** | ตั้งค่าวันเปิดให้บริการรายช่วงเวลา (Recurring Weekly: 1=จันทร์…7=อาทิตย์) — Hard Availability Gate; ห้ามลบ (ช่วงเวลา, วัน) ที่มีคิว CONFIRMED ในอนาคต; บันทึก Audit Log |

**หมายเหตุ:**
- STAFF สามารถ CRUD Services + toggle ได้ในศูนย์ตัวเอง แต่ **ห้ามลบ**
- HC_ADMIN + SUPER_ADMIN มีสิทธิ์เพิ่มเติมคือ: จัดการ selectable staff (`/staff`) และ operating days (`/time-slot-days`)
- `PUT /services/{id}/staff` ต้องหมอในหมวดหมู่เดียวกับบริการและศูนย์เดียวกัน
- เปิด `allow_staff_selection` ต้อง `capacity_type = PER_MASSEUSE` + มีหมอผูกอย่างน้อย 1 คน; ปิดต้องไม่มีคิว CONFIRMED ในอนาคต

### 3.4 การจัดการนัดหมาย (Appointments & Queue Desk)

| Method | Endpoint | Roles ที่เข้าถึงได้ | คำอธิบาย |
|---|---|---|---|
| `GET` | `/v1/staff/appointments` | STAFF, HC_ADMIN, SUPER_ADMIN | ดูนัดหมายประจำวัน |
| `POST` | `/v1/staff/appointments/walk-in` | STAFF, HC_ADMIN, SUPER_ADMIN | ลงทะเบียน Walk-in หน้างาน |
| `PATCH` | `/v1/staff/appointments/{id}/status` | STAFF, HC_ADMIN, SUPER_ADMIN | อัปเดตสถานะ (CONFIRMED → COMPLETED / CANCELLED / NO_SHOW) |
| `PATCH` | `/v1/staff/appointments/{id}/reassign-staff` | STAFF, HC_ADMIN, SUPER_ADMIN | ย้ายคิวไปหมอคนอื่น (เฉพาะ CONFIRMED + บริการ allow_staff_selection) |
| `POST` | `/v1/staff/appointments/{id}/unmask` | STAFF, HC_ADMIN, SUPER_ADMIN | ดูข้อมูลผู้ป่วยแบบ Unmask (PDPA Audit Log) |

**ข้อจำกัดการมองเห็น (index):**
- **STAFF** (ไม่มี role บริหาร): เห็นเฉพาะนัดหมายของ **Assigned Services** ที่ตัวเองรับผิดชอบ — 403 ถ้า filter service_id ที่ไม่ได้ assign
- **HC_ADMIN / SUPER_ADMIN**: เห็น **ทุกบริการ** ในศูนย์

**ข้อจำกัด Walk-in:**
- **STAFF**: ออกคิว Walk-in ได้เฉพาะ **Assigned Services** — 403 ถ้าบริการไม่ได้ assign; 422 ถ้า staff ที่รับผิดชอบบริการนี้ลางานทั้งหมดวันนี้
- **HC_ADMIN / SUPER_ADMIN**: ออกได้ทุกบริการในศูนย์

**ไม่จำกัดตาม Assigned Services (ทุกบริการในศูนย์):**
- `updateStatus` — อัปเดตสถานะนัดหมายใดก็ได้ในศูนย์
- `reassignStaff` — ย้ายคิวใดก็ได้ในศูนย์
- `unmaskPatientData` — ดูข้อมูลผู้ป่วยใดก็ได้ในศูนย์

### 3.5 การจัดการข้อมูลผู้ป่วย (Patient Data)

| Method | Endpoint | Roles ที่เข้าถึงได้ | คำอธิบาย |
|---|---|---|---|
| `PATCH` | `/v1/staff/patients/{id}/contact` | STAFF, HC_ADMIN, SUPER_ADMIN | แก้ไขเบอร์โทรศัพท์ผู้ป่วย — ตรวจสอบ format + ห้ามซ้ำ |
| `PATCH` | `/v1/staff/patients/{id}` | **HC_ADMIN, SUPER_ADMIN เท่านั้น** | แก้ไขข้อมูลผู้ป่วยเต็มรูปแบบ (ชื่อ/วันเกิด/เพศ/ที่อยู่/CID) — ห้าม CID ซ้ำ; บันทึก Audit Log |
| `GET` | `/v1/staff/admin/patients` | **HC_ADMIN, SUPER_ADMIN เท่านั้น** | ค้นหารายชื่อผู้ป่วย — SUPER_ADMIN เห็นทุกศูนย์; HC_ADMIN เห็นเฉพาะศูนย์ตัวเอง; เห็นข้อมูลเต็ม |

**ข้อจำกัด:** ทุกการ Unmask / แก้ไขข้อมูลจะถูกบันทึก Audit Log พร้อม Mask ข้อมูลใน Log เสมอ

### 3.6 การจัดการบุคลากร (Roster)

| Method | Endpoint | Roles ที่เข้าถึงได้ | คำอธิบาย |
|---|---|---|---|
| `GET` | `/v1/staff/roster` | STAFF, HC_ADMIN, SUPER_ADMIN | ดูรายการบุคลากร — รองรับ `?date=` เพื่อดู effective_status (ถ้ามี leave วันนั้น = LEAVE) |
| `POST` | `/v1/staff/roster` | STAFF, HC_ADMIN, SUPER_ADMIN | เพิ่มบุคลากรใหม่ในศูนย์ |
| `PATCH` | `/v1/staff/roster/{id}/toggle-duty` | STAFF, HC_ADMIN, SUPER_ADMIN | สลับสถานะ ACTIVE ↔ LEAVE — ห้ามสลับถ้า INACTIVE; ห้าม ACTIVE → LEAVE ถ้ามีคิว CONFIRMED ในอนาคต |

**หมายเหตุ:** STAFF สามารถเพิ่มและสลับ duty บุคลากรคนอื่นในศูนย์เดียวกันได้ (ไม่จำกัดเฉพาะตัวเอง)

### 3.7 การจัดการวันลา (Staff Leaves)

| Method | Endpoint | Roles ที่เข้าถึงได้ | คำอธิบาย |
|---|---|---|---|
| `GET` | `/v1/staff/leaves` | STAFF, HC_ADMIN, SUPER_ADMIN | ดูรายการวันลา — รองรับ filter `?date=`, `?staff_user_id=` |
| `POST` | `/v1/staff/leaves` | STAFF, HC_ADMIN, SUPER_ADMIN | ลงวันลา (auto-approved) — ห้ามลงวันลาถ้ามีคิว CONFIRMED ในวันนั้น |
| `DELETE` | `/v1/staff/leaves/{id}` | STAFF, HC_ADMIN, SUPER_ADMIN | ยกเลิก/ลบวันลา |
| `GET` | `/v1/staff/leaves/availability` | STAFF, HC_ADMIN, SUPER_ADMIN | ตรวจสอบว่าบริการเปิดให้บริการในวันที่กำหนดหรือไม่ |

**ข้อจำกัดการจัดการวันลา:**
- **STAFF**: ลงวันลาให้ **ตัวเองเท่านั้น**; ลบได้เฉพาะวันลาของตัวเอง
- **HC_ADMIN**: ลงวันลาให้ **ใครก็ได้ในศูนย์ตัวเอง**; ลบได้เฉพาะวันลาในศูนย์ตัวเอง
- **SUPER_ADMIN**: ลงวันลาให้ใครก็ได้; ลบได้ทุกที่
- ลงวันลาซ้ำวันเดิมจะไม่สร้างรายการใหม่ (idempotent)

### 3.8 การดูรายชื่อ รพ.สต. (Dropdown)

| Method | Endpoint | Roles ที่เข้าถึงได้ | คำอธิบาย |
|---|---|---|---|
| `GET` | `/v1/staff/health-centers` | **ทุกคน (auth:sanctum) — ไม่มี role gate** | SUPER_ADMIN เห็นทุกศูนย์ (ชื่อ/ที่อยู่/พิกัด/สถานะ/has_webhook + INACTIVE); อื่นๆ เห็นเฉพาะศูนย์ตัวเอง (ACTIVE/CLOSING เท่านั้น) |

**หมายเหตุ:** Route มีแค่ `auth:sanctum` (ไม่มี role gate) — แต่ในทางปฏิบัติผู้ป่วย **ไม่สามารถ** ใช้งานได้: controller เรียก `hasRole()` ซึ่ง Patient model ไม่มี relation `roles()` → จะเจอ 500 (RelationNotFoundException) ถ้า Patient token call มา

---

## 4. HC Admin (ผู้ดูแลประจำศูนย์)

HC Admin มีสิทธิ์ **ทุกอย่างที่ STAFF มี** แล้วเพิ่มสิทธิ์เฉพาะดังนี้:

### 4.1 สิทธิ์เพิ่มเติมจาก STAFF

| Method | Endpoint | คำอธิบาย |
|---|---|---|
| `DELETE` | `/v1/staff/services/{id}` | ลบบริการ (STAFF ห้ามลบ) |
| `PUT` | `/v1/staff/services/{id}/staff` | ผูกหมอที่เลือกได้ (selectable staff) เข้ากับบริการ |
| `PUT` | `/v1/staff/services/{id}/time-slot-days` | ตั้งค่าวันเปิดให้บริการ (Operating Days) |
| `PATCH` | `/v1/staff/patients/{id}` | แก้ไขข้อมูลผู้ป่วยเต็มรูปแบบ (ชื่อ/วันเกิด/เพศ/ที่อยู่/CID) |
| `GET` | `/v1/staff/admin/patients` | ค้นหารายชื่อผู้ป่วยในศูนย์ตัวเอง |

### 4.2 การจัดการผู้ใช้งาน (User Management)

| Method | Endpoint | คำอธิบาย |
|---|---|---|
| `GET` | `/v1/staff/admin/users` | ดูรายการผู้ใช้ในศูนย์ตัวเอง |
| `POST` | `/v1/staff/admin/users` | สร้างผู้ใช้ใหม่ — **จำกัด role_ids ได้เฉพาะ STAFF** |
| `PATCH` | `/v1/staff/admin/users/{id}` | แก้ไขผู้ใช้ — ห้ามแก้ SUPER_ADMIN; ห้ามย้ายศูนย์; จำกัด role เป็น STAFF |
| `DELETE` | `/v1/staff/admin/users/{id}` | ลบผู้ใช้ — ห้ามลบ SUPER_ADMIN; ห้ามลบตัวเอง; ห้ามลบถ้ามีคิว CONFIRMED ในอนาคต |
| `GET` | `/v1/staff/admin/roles` | ดูรายการ role — **เห็นเฉพาะ STAFF** |

### 4.3 การจัดการ Time Slots

| Method | Endpoint | คำอธิบาย |
|---|---|---|
| `GET` | `/v1/staff/admin/time-slots` | ดูรายการช่วงเวลาเฉพาะศูนย์ตัวเอง |
| `PATCH` | `/v1/staff/admin/time-slots/{id}/toggle-status` | เปิด/ปิดสถานะช่วงเวลาเฉพาะในศูนย์ตัวเอง |

**หมายเหตุ:** HC Admin **สามารถเปิด/ปิด** ช่วงเวลาในศูนย์ตัวเองได้ แต่ **ไม่สามารถ** Create/Update/Delete ช่วงเวลาได้ (SUPER_ADMIN only)

### 4.4 การจัดการข้อมูล รพ.สต.

| Method | Endpoint | คำอธิบาย |
|---|---|---|
| `PATCH` | `/v1/staff/health-centers/{id}` | แก้ไขข้อมูลศูนย์ของตนเอง (ชื่อ/เบอร์/ที่อยู่/พิกัด) — **ห้ามแก้ `code`**; บันทึก Audit Log |

---

## 5. Super Admin (ผู้ดูแลระบบสูงสุด)

> Super Admin มี **ทุกสิทธิ์ที่ HC Admin มี** แล้วเพิ่มสิทธิ์ข้ามศูนย์ (cross-tenant) ดังนี้:

### 5.1 Cross-Center Access

- ดู Dashboard Aggregate ครอบคลุม **ทุกศูนย์** (ไม่ระบุ `health_center_id` = รวมทุกศูนย์)
- ค้นหา/กรองข้อมูลแยกตามศูนย์ได้
- **ทุกรายการ write ต้องระบุ `health_center_id`** ทุกครั้ง — ถ้าไม่ระบุระบบจะปฏิเสธ (422)

### 5.2 Super Admin Only Endpoints

| Method | Endpoint | คำอธิบาย |
|---|---|---|
| `PATCH` | `/v1/staff/discord/webhook` | ตั้งค่า/ยกเลิก Discord Webhook สำหรับแต่ละศูนย์ — ต้องระบุ `health_center_id` |
| `PATCH` | `/v1/staff/admin/health-centers/{id}/toggle-status` | เปิด/ปิดรับการจองของศูนย์ — ACTIVE ↔ CLOSING, INACTIVE → ACTIVE |
| `GET` | `/v1/staff/admin/health-centers` | รายชื่อศูนย์ทั้งหมด (ทุก status + INACTIVE) พร้อม Filter |
| `POST` | `/v1/staff/admin/time-slots` | เพิ่มช่วงเวลาใหม่ในศูนย์ที่ระบุ |
| `PUT` | `/v1/staff/admin/time-slots/{id}` | แก้ไขช่วงเวลา (start_time / end_time / is_active) |
| `DELETE` | `/v1/staff/admin/time-slots/{id}` | ลบช่วงเวลา — ห้ามลบถ้ามี CONFIRMED appointment ในวันนี้หรืออนาคต |
| `PATCH` | `/v1/staff/admin/time-slots/{id}/toggle-status` | เปิด/ปิดสถานะช่วงเวลา (ระบุ health_center_id) |

### 5.3 Cross-Center Operations (ใช้ endpoint เดียวกับ HC Admin แต่ scope ข้ามศูนย์)

| Operation | Endpoint | ข้อจำกัดเพิ่มเติม |
|---|---|---|
| แก้ไขข้อมูลผู้ป่วย | `PATCH /v1/staff/patients/{id}` | ต้องระบุ `health_center_id`; ผู้ป่วยต้องมีประวัติคิวที่ศูนย์นั้น |
| ผูกหมอเข้ากับบริการ | `PUT /v1/staff/services/{id}/staff` | ต้องระบุ `health_center_id`; หมอต้องในศูนย์เดียวกันและหมวดหมู่เดียวกับบริการ |
| ตั้งค่าวันเปิดให้บริการ | `PUT /v1/staff/services/{id}/time-slot-days` | ต้องระบุ `health_center_id` |
| แก้ไขข้อมูล รพ.สต. | `PATCH /v1/staff/health-centers/{id}` | แก้ได้ทุกฟิลด์ **รวมถึง `code`** ( SUPER_ADMIN only ); ต้องระบุ `health_center_id` ให้ตรงกับ `{id}` |

### 5.4 สถานะ 3 ระดับของศูนย์สุขภาพ (Open/Close Lifecycle)

| สถานะ | ความหมาย |
|---|---|
| `ACTIVE` | เปิดรับจองตามปกติ |
| `CLOSING` | ปิดรับจองใหม่ แต่ยังให้บริการ/เคลียร์คิวที่ค้าง — ผู้ป่วยยังเห็น/ยกเลิกนัดเดิมได้; เจ้าหน้าที่ **ที่มี token ค้างอยู่** ยังจัดการคิวค้างได้ (แต่ **Login ใหม่ถูกบล็อก** เนื่องจาก login ต้องเป็นศูนย์ ACTIVE) |
| `INACTIVE` | ปิดสมบูรณ์ — ซ่อนจากทุกคน (Patient 404, Staff ไม่สามารถ Login ได้) |

**Auto-Close:** เมื่อศูนย์อยู่ในสถานะ CLOSING และคิว CONFIRMED หมดแล้ว ระบบจะเปลี่ยนเป็น INACTIVE โดยอัตโนมัติผ่าน Background Job พร้อมบันทึก Audit Log (`AUTO_CLOSE_HEALTH_CENTER`)

---

## 6. ตารางเปรียบเทียบสิทธิ์โดยย่อ

| ความสามารถ | STAFF | HC_ADMIN | SUPER_ADMIN | Patient |
|---|:---:|:---:|:---:|:---:|
| **Dashboard** | ศูนย์ตัวเอง | ศูนย์ตัวเอง | ทุกศูนย์ | — |
| **Services: ดูรายการ** | ✔ | ✔ | ✔ | ✔ (public) |
| **Services: เพิ่ม/แก้ไข/toggle** | ✔ | ✔ | ✔ | — |
| **Services: ลบ** | — | ✔ | ✔ | — |
| **Services: ผูก Assignees** | ✔ | ✔ | ✔ | — |
| **Services: ผูก Selectable Staff** | — | ✔ | ✔ | — |
| **Services: ตั้ง Operating Days** | — | ✔ | ✔ | — |
| **Appointments: ดูรายการ** | Assigned services only | ทุกบริการ | ทุกบริการ | ของตัวเอง |
| **Walk-in** | Assigned services only | ทุกบริการ | ทุกบริการ | — |
| **อัปเดตสถานะ / ย้ายคิว / Unmask** | ทุกบริการในศูนย์ | ทุกบริการ | ทุกบริการ | ยกเลิกนัดตัวเอง |
| **Patient: แก้ไขเบอร์โทร** | ✔ | ✔ | ✔ | — |
| **Patient: แก้ไขข้อมูลเต็ม** | — | ✔ | ✔ | — |
| **Patient: ค้นหารายชื่อ** | — | ✔ | ✔ (ทุกศูนย์) | — |
| **Roster: ดู/เพิ่ม/สลับ duty** | ✔ (ศูนย์ตัวเอง) | ✔ | ✔ | — |
| **Leave: ลงวันลา** | ตัวเองเท่านั้น | ใครก็ได้ในศูนย์ | ใครก็ได้ | — |
| **Leave: ลบวันลา** | ตัวเองเท่านั้น | ในศูนย์ตัวเอง | ทุกที่ | — |
| **User Management** | — | ✔ (STAFF only, ศูนย์ตัวเอง) | ✔ ทุก role | — |
| **Time Slots: ดูรายการ** | — | ✔ (ศูนย์ตัวเอง) | ✔ (ทุกศูนย์) | — |
| **Time Slots: Toggle Status** | — | ✔ (ศูนย์ตัวเอง) | ✔ (ทุกศูนย์) | — |
| **Time Slots: CRUD (เพิ่ม/แก้ไข/ลบ)** | — | — | ✔ (ทุกศูนย์) | — |
| **Health Centers: แก้ไขข้อมูล** | — | ศูนย์ตัวเอง (ห้ามแก้ code) | ทุกศูนย์ (แก้ code ได้) | — |
| **Health Centers: Toggle Status** | — | — | ✔ | — |
| **Discord Webhook** | — | — | ✔ | — |

---

## 7. จุดเด่นด้านความปลอดภัยและ Business Logic

1. **Data Privacy (PDPA):** ข้อมูลผู้ป่วยถูก Mask เป็นค่าเริ่มต้น; ทุกการ Unmask / แก้ไขข้อมูลถูกบันทึก Audit Log พร้อม Mask ข้อมูลใน Log เสมอ
2. **Strict Multi-Tenancy:** เจ้าหน้าที่จัดการได้เฉพาะข้อมูลในศูนย์ของตนเอง; STAFF ยังถูกจำกัดเพิ่มเติมด้วย Assigned Services สำหรับการดูรายการนัดหมายและ Walk-in
3. **Queue & Capacity Control with Leave Integration:** ป้องกัน Overbooking, ป้องกันการจองซ้ำวันเดียวกัน; ถ้า staff ทั้งหมดลางาน บริการจะถูกซ่อนและบล็อก Walk-in อัตโนมัติ
4. **Operating Days (Recurring Weekly Availability Gate):** แต่ละบริการกำหนดวันเปิดให้บริการรายช่วงเวลา (1=จันทร์…7=อาทิตย์) เป็น Hard Gate; Invariant บังคับให้บริการ Active ต้องมีตารางไม่ว่าง; Guard กันการลบ (ช่วงเวลา, วัน) ที่มีคิว CONFIRMED ในอนาคต
5. **Health Center Closing Lifecycle:** สถานะ 3 ระดับ (ACTIVE → CLOSING → INACTIVE); การปิดอัตโนมัติเมื่อคิว CONFIRMED หมด; สลับสถานะได้เฉพาะ SUPER_ADMIN
6. **Staff Selection:** บริการที่ตั้ง `allow_staff_selection=true` ให้ผู้ป่วย/เจ้าหน้าที่เลือกหมอได้ — 1 คิว/รอบเวลา/คน, ตรวจสอบวันลาและผูกบริการอัตโนมัติ
7. **Last Super Admin Guard:** ห้ามลบ/ปิดใช้งาน SUPER_ADMIN คนสุดท้ายที่ยัง ACTIVE
