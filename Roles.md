# สรุปโครงสร้างสิทธิ์และการทำงานของระบบนัดหมายและบริการสุขภาพ

> **อัปเดตเมื่อ:** 2026-09-23
> **อ้างอิง code:** `bd73d63` (QRS-QueueReservationSystem, branch master)
>
> เอกสารนี้เขียนให้ตรงกับ **พฤติกรรมจริงของ code** (รวมถึงจุดที่เป็น gap/bug ในปัจจุบัน) ไม่ใช่เอกสารดีไซน์ในอุดมคติ จุดที่เป็น gap จะระบุด้วยสัญญลักษณ์ ⚠️

---

## 1. ผู้ใช้งานในระบบ (Actors)

ระบบนี้มี **4 บทบาทหลัก** โดย 3 บทบาทแรกเป็น Staff (ใช้ User model) และอีก 1 เป็น Patient (ใช้ Patient model แยกกัน):

| # | Role (DB) | Token Ability | Model | คำอธิบาย |
|---|---|---|---|---|
| 1 | `SUPER_ADMIN` | `role:staff` | User | ผู้ดูแลระบบระดับจังหวัด (อบจ./สสจ.) — cross-tenant access |
| 2 | `HEALTH_CENTER_ADMIN` | `role:staff` | User | ผู้ดูแลระบบประจำ รพ.สต. — scope: ศูนย์ของตนเอง |
| 3 | `STAFF` | `role:staff` | User | เจ้าหน้าที่ห้องตรวจ / หมอ / ผู้ช่วย — scope: ศูนย์ของตนเอง |
| 4 | *(ไม่มี DB role)* | `role:patient` | Patient | ผู้ป่วย / ผู้รับบริการ |

**หมายเหตุเกี่ยวกับ middleware ที่ตรวจสิทธิ์:**
- Staff ทุก route ถูกกำกับโดย middleware `role:STAFF,HEALTH_CENTER_ADMIN,SUPER_ADMIN` (alias `role`) ซึ่งตรวจ **DB roles** ผ่าน `User::roles()` relation (`user_roles` pivot) เป็นหลัก — ไม่ได้แยกระดับด้วย token ability
- middleware `role:` มี fallback ตรวจ token abilities ด้วย แต่ **fallback นี้ใช้งานไม่ได้จริง** ⚠️: token ของ staff ได้ ability `role:staff` (พิมพ์เล็ก) ขณะที่ argument ใน middleware เป็น `STAFF` (พิมพ์ใหญ่) → `in_array()` ไม่ match → การตัดสิทธิ์ขึ้นอยู่กับ DB roles เพียงอย่างเดียว
- `role:patient` ใช้จริงใน patient routes ผ่าน middleware `abilities:role:patient`
- หน้าเชิงเทคนิค: `SUPER_ADMIN` ไม่มี bypass ฮาร์ดโค้ดใน middleware — การ "ข้าม" scope เกิดจากการที่ controller ตรวจ `hasRole('SUPER_ADMIN')` แล้วไม่บังคับ health_center_id ของตัวเอง

---

## 2. Patient (ผู้ป่วย / ผู้รับบริการ)

### 2.1 การลงทะเบียนและเข้าสู่ระบบ

| Method | Endpoint | Middleware | คำอธิบาย |
|---|---|---|---|
| `POST` | `/v1/patient/register` | `throttle:patient-register` (5 ครั้ง/นาที/IP) | ลงทะเบียน — สร้าง Patient + PatientRight (is_primary=true) + ออก token ผ่าน DB transaction |
| `POST` | `/v1/patient/login` | `throttle:patient-login` (10 ครั้ง/นาที/เบอร์) | เข้าสู่ระบบด้วย เบอร์โทร+วันเกิด — lock 30 นาทีหลังผิดวันเกิด 5 ครั้ง |
| `GET` | `/v1/patient/me` | `auth:sanctum`, `abilities:role:patient` | ดูข้อมูลส่วนตัว (masked) |

**ฟิลด์ register (บังคับ):** `cid` (13 หลัก, unique), `first_name`, `last_name`, `phone_number` (10 หลัก, unique), `birth_date`, `right_type_id` (ต้องมี `exists:right_types,id`); optional: `gender` (MALE/FEMALE/OTHER), `subdistrict`, `district`, `province`, `full_address`, `main_hospital_name`

**พฤติกรรมจริงของ login (ต่างจากที่คิดกันทั่วไป):**
- เบอร์โทรที่ **ยังไม่เคยลงทะเบียน** → คืน **HTTP 200** `{ is_registered: false, token: null, patient: null }` (ไม่ใช่ 401) พร้อม message "ไม่พบประวัติผู้รับบริการ กรุณาลงทะเบียนใหม่"
- **วันเกิดผิด** → 401 "ข้อมูลวันเดือนปีเกิดไม่ถูกต้อง (เหลือโอกาสอีก N ครั้ง)"; ผิดครบ 5 ครั้ง → **423** ระงับ 30 นาที (`locked_until = now()+30min`)
- เบอร์ที่ยังล็อก → 423 "บัญชีถูกระงับชั่วคราว...กรุณารออีก N นาที"
- วันเกิดถูก → reset `failed_login_attempts=0`, `locked_until=null`, ออก token `role:patient`

### 2.2 ค้นหาบริการและสถานที่ (Public — ไม่ต้องล็อกอิน)

| Method | Endpoint | คำอธิบาย |
|---|---|---|
| `GET` | `/v1/patient/categories` | หมวดหมู่บริการ — เฉพาะ `status=ACTIVE`, เรียงตาม `sort_order` |
| `GET` | `/v1/patient/right-types` | สิทธิการรักษา — เฉพาะ `status=ACTIVE` (UCS, OFC, SSS, CASH, OTHER) |
| `POST` | `/v1/patient/nearby-centers` | ค้นหา รพ.สต. ใกล้เคียง (Haversine; **เฉพาะ** ACTIVE + มีบริการ active ในหมวดที่เลือก; **ไม่มี** radius filter) |
| `GET` | `/v1/patient/health-centers/{id}` | รายละเอียด รพ.สต. — **ACTIVE หรือ CLOSING** เห็นได้; INACTIVE → 404; คืน `status` ของศูนย์ด้วย |
| `GET` | `/v1/patient/health-centers/{id}/services` | บริการของศูนย์ — เฉพาะ services `is_active=true` **และ** มี pivot `service_time_slot` ≥ 1 แถว **และ** อยู่หมวดหมู่ ACTIVE; แต่ละรายการคืน `allow_staff_selection`; envelope คืน `health_center_status`; response ผ่าน Resource (**ไม่มี** `created_at`/`updated_at`) |
| `GET` | `/v1/patient/available-slots` | ตรวจสอบช่วงเวลาว่าง — พารามิเตอร์ **มีแค่** `service_id` + `date` (ต้อง `>= today`) |

**หมายเหตุ available-slots (พฤติกรรมจริง):**
- ไม่รับ `health_center_id` / `time_slot_id` — service ระบุศูนย์เอง
- ถ้าศูนย์ไม่ใช่ ACTIVE (CLOSING/INACTIVE) → คืน **200 `slots: []`** (ไม่ใช่ error)
- เช็ค "เปิดวันนี้ไหม" ผ่าน bit `days_mask` บน pivot `service_time_slot` + slot ต้อง `is_active`
- `available_count` คำนวณแบบ snapshot (ไม่มี row lock) → แสดงผลอาจ stale ได้เล็กน้อย; การจองจริงจะ re-check ใน transaction ⚠️
- **`PER_MASSEUSE`** max capacity = นับ STAFF ACTIVE ในศูนย์+หมวดเดียวกัน (รวมคนที่ลา!) — ระบบไม่ตัดวันลาออกในขั้นนี้
- **staff-selection** (`allow_staff_selection=true`): `available_count` = จำนวน staff ที่ว่าง (ACTIVE + ไม่ลา + ผูกบริการ) ในรอบนั้น, **PER_STAFF_PER_SLOT_LIMIT = 1**

### 2.3 การนัดหมาย (Booking)

| Method | Endpoint | คำอธิบาย |
|---|---|---|
| `POST` | `/v1/patient/book` | จองคิว — ระบบออกเลขคิว Sequential (ต่อศูนย์/ต่อวัน) |
| `GET` | `/v1/patient/services/{id}/staff` | หมอที่เลือกได้ (auth แล้ว) — เฉพาะ `allow_staff_selection=true`; คืน `id, name, position` |

**ลำดับการเช็คใน code (BookingService::createAppointment — transaction + lockForUpdate):**
1. Service ต้อง `is_active` → ผิด 422
2. ศูนย์ต้อง `ACTIVE` ผ่าน `HealthCenter::isBookable()` → **CLOSING / INACTIVE บล็อก** "ศูนย์ปิดรับการจองใหม่"
3. วันที่ต้องเปิดตาม `days_mask` ของ pivot + slot `is_active` → ผิด 422 "บริการนี้ไม่เปิดให้บริการในวันที่เลือก"
4. **เช็คจองซ้ำ** เฉพาะ appointment ที่ `status=CONFIRMED` วันที่+บริการเดียวกัน → 422 (ดูหัวข้อถัดไป)
5. ถ้า `allow_staff_selection=true`: ต้องส่ง `staff_id` บังคับ; เช็ค staff ACTIVE + ผูก `service_staff` + ไม่ลา + ยังไม่อิ่มคิว (1 คิว/รอบเวลา/คน) → ผิด 422
6. เช็ค capacity (เฉพาะนับ CONFIRMED): `PER_DAY` นับทั้งวัน, `PER_SLOT`/`PER_MASSEUSE` นับต่อ slot → เต็ม 422
7. เลขคิว = max(`appointment_number`) ของ `(health_center_id, appointment_date)` + 1 (unique constraint รองรับ)
8. สถานะเริ่มต้น = `CONFIRMED`, `booked_by_type = SELF`; dispatch Discord notification (async)

**ข้อปลีกย่อยที่นักพัฒนาต้องรู้:**
- ⚠️ **ข้อความ error ของ booking ถูกกลืนหมด** — client เห็นเพียง `422 "ไม่สามารถจองคิวได้ กรุณาลองใหม่อีกครั้ง"` สาเหตุจริง (เช่น "คิวเต็ม", "จองซ้ำ") อยู่ใน server log เท่านั้น
- ✅ **patient booking เช็ค "staff ทั้งหมดลา" เหมือน walk-in แล้ว** — `BookingService` เรียก `StaffLeave::isServiceAvailable()` ก่อนจอง (แก้ gap #6); ถ้าไม่มี pool (assigned/selectable) ที่กำหนด → ถือว่าเปิดให้บริการ (ให้ capacity/operating-day ตัดสิน)
- จองซ้ำได้ถ้าคiuก่อนหน้าเป็น CANCELLED/COMPLETED/NO_SHOW (นับเฉพาะ CONFIRMED)

### 2.4 ประวัติและการจัดการนัดหมาย

| Method | Endpoint | คำอธิบาย |
|---|---|---|
| `GET` | `/v1/patient/appointments` | รายการนัดหมายของตัวเอง (mask ชื่อ/เบอร์) — paginate, default 20, max 100 |
| `GET` | `/v1/patient/appointments/{id}` | รายละเอียดนัด — **ทุกสถานะ** ดูได้; response key คือ `ticket` |
| `PATCH` | `/v1/patient/appointments/{id}/cancel` | ยกเลิกนัด — เงื่อนไขตามด้านล่าง |

**เงื่อนไข cancel (ที่เขียนใน route/query):**
- ต้องเป็น `CONFIRMED`
- ต้องเป็นของตัวเอง (`patient_id`) → ไม่ใช่ → **404**
- ⚠️ เวลาตัดคือ **`end_time` ของ slot** (ไม่ใช่ start_time): ยกเลิกได้ถ้า `appointment_date > วันนี้` หรือถ้าเป็นวันนี้ต้อง `end_time > now()` — คือยกเลิกได้แม้ slot เริ่มไปแล้ว ขอแค่ยังไม่จบรอบ
- CANCELLED/COMPLETED/NO_SHOW → ห้าม (query ไม่ match → 404); ตั้งค่า `cancelled_at`, `cancellation_reason` (default "ผู้ป่วยขอยกเลิกเองผ่านระบบ")

**Masking (ตามจริง):**
- ชื่อ: first_name 2 ตัวแรก + `***` + last_name 2 ตัวแรก + `***` (เช่น `สม*** ใจ***`)
- เบอร์: `081-***-1234` (3 หลักแรก + `-***-` + 4 หลักท้าย)
- CID: `1-****-*****-99` (1 หลักแรก + `-****-*****-` + 2 หลักท้าย)
- `/me` คืน: `id, masked_cid, masked_name, masked_phone, gender, rights[]` — ไม่คืน birth_date
- Login response คืนแค่ `id, masked_name, masked_phone`

---

## 3. Staff (เจ้าหน้าที่ / หมอ / ผู้ให้บริการ)

> **คำสำคัญ:** Staff ทุกคนถูกจำกัด scope ด้วย `health_center_id` ของตัวเอง (ยกเว้น SUPER_ADMIN ที่ต้องระบุ `health_center_id` ทุกครั้งในรายการ write ตามกฎหลัก — **ยกเว้น** time-slots ⚠️ มีรายละเอียดใน §5)

### 3.1 การเข้าสู่ระบบและโปรไฟล์

| Method | Endpoint | Roles ที่เข้าถึงได้ | คำอธิบาย |
|---|---|---|---|
| `POST` | `/v1/staff/login` | ทุกคน (Public) | username+password; ต้อง user `status=ACTIVE`; ต้องศูนย์ `ACTIVE` หรือ `CLOSING` |
| `GET` | `/v1/staff/me` | ทุกคน (auth:sanctum) | ดูโปรไฟล์ตัวเอง + roles + health center |
| `PATCH` | `/v1/staff/me` | ทุกคน (auth:sanctum) | แก้ชื่อ/รหัสเอง (password hash ใหม่) |
| `POST` | `/v1/staff/logout` | ทุกคน (auth:sanctum) | ลบ token ปัจจุบัน |

**พฤติกรรมจริงของ login:**
- User `status != ACTIVE` (รวม INACTIVE) → ถูกมองเสมือน "ไม่รู้จัก" → **422** "ชื่อผู้ใช้งานหรือรหัสผ่านไม่ถูกต้อง" (เหมือน username ผิด)
- ศูนย์เป็น **INACTIVE** → **403** "รพ.สต. ต้นสังกัดถูกระงับการใช้งานชั่วคราว" (CLOSING ยัง login ได้ — ให้ staff จัดการคิวที่เหลือ)
- SUPER_ADMIN **ข้าม** การตรวจศูนย์ (ไม่ต้องผูก health center, `health_center_id` เป็น null ได้)
- Token อายุ **24 ชม.** (Sanctum `expiration => 1440`); logout ลบ token; ลบ user → ลบ tokens ทั้งหมด

**⚠️ Throttle staff login (มี bug):** middleware `ThrottleStaffLogin` ตั้งใจลิมิต 5 ครั้ง/30 นาที แต่**นับเฉพาะ status 401/403** ขณะที่ login ผิด (username/password) คืน **422** (ValidationException) → ทุกความพยายามผิด จะไปโดน `RateLimiter::clear()` แทนการ `hit()` → **counter ไม่เคยขึ้น → ลิมิตไม่ทำงานจริงในทางปฏิบัติ** เฉพาะปลายทางที่คืน 401/403 จริงถึงจะนับ

### 3.2 Dashboard

| Method | Endpoint | Roles ที่เข้าถึงได้ | คำอธิบาย |
|---|---|---|---|
| `GET` | `/v1/staff/dashboard/summary` | STAFF, HC_ADMIN, SUPER_ADMIN | สถิติวัน (default วันนี้, รับ `?date=`) — SUPER_ADMIN ไม่ระบุ `health_center_id` = รวมทุกศูนย์ |

### 3.3 การจัดการบริการ (Services)

| Method | Endpoint | Roles ที่เข้าถึงได้ | คำอธิบาย |
|---|---|---|---|
| `GET` | `/v1/staff/services` | STAFF, HC_ADMIN, SUPER_ADMIN | ดูบริการ — STAFF/HC_ADMIN ศูนย์ตัวเอง; SUPER_ADMIN ทุกศูนย์ หรือ `?health_center_id=`; `?date=` ซ่อนบริการที่ staff (assigned/selectable) ลางานทั้งหมดวันนั้น (เฉพาะเมื่อมี center scope ชัดเจน) |
| `POST` | `/v1/staff/services` | STAFF, HC_ADMIN, SUPER_ADMIN | เพิ่มบริการ — **STAFF ทำได้จริง**; บังคับ `allow_staff_selection=false` ตอนสร้างเสมอ; `capacity_type` ∈ PER_MASSEUSE/PER_SLOT/PER_DAY |
| `PUT` | `/v1/staff/services/{id}` | STAFF, HC_ADMIN, SUPER_ADMIN | แก้บริการ — **STAFF ทำได้จริง**; เปิด staff-selection ต้อง PER_MASSEUSE + มี selectable staff ≥1; ปิดต้องไม่มีคิว CONFIRMED ตั้งแต่วันนี้เป็นต้นไป |
| `PATCH` | `/v1/staff/services/{id}/toggle-status` | STAFF, HC_ADMIN, SUPER_ADMIN | เปิด/ปิดบริการ — **STAFF ทำได้จริง** (ไม่มี in-controller gate) |
| `DELETE` | `/v1/staff/services/{id}` | **HC_ADMIN, SUPER_ADMIN เท่านั้น** | ลบบริการ (**STAFF → 403 "ไม่มีสิทธิ์ลบบริการ"**); 422 ถ้ามีคิว CONFIRMED ตั้งแต่วันนี้; ข้อความ error อื่นกลืนเป็น 422 ทั่วไป |
| `PUT` | `/v1/staff/services/{id}/assignees` | STAFF, HC_ADMIN, SUPER_ADMIN | ผูก User (role **STAFF** เท่านั้น) เข้ากับบริการ — **STAFF ทำได้จริง**; user ต้องอยู่ศูนย์เดียวกัน |
| `PUT` | `/v1/staff/services/{id}/staff` | **HC_ADMIN, SUPER_ADMIN เท่านั้น** | ผูกหมอที่เลือกได้ (selectable staff, pivot `service_staff`) — STAFF ไม่ได้; หมอต้องศูนย์เดียวกัน + หมวดหมู่เดียวกับบริการ; แกะหมอที่มีคิว CONFIRMED อนาคตไม่ได้; Audit Log |
| `PUT` | `/v1/staff/services/{id}/time-slot-days` | **HC_ADMIN, SUPER_ADMIN เท่านั้น** | ตั้งค่าวันเปิด (Recurring Weekly ผ่าน `days_mask` bit 1-7); ห้ามลบ (slot,วัน) ที่มีคิว CONFIRMED อนาคต; Audit Log |

### 3.4 การจัดการนัดหมาย (Appointments & Queue Desk)

| Method | Endpoint | Roles ที่เข้าถึงได้ | คำอธิบาย |
|---|---|---|---|
| `GET` | `/v1/staff/appointments` | STAFF, HC_ADMIN, SUPER_ADMIN | ดูนัดประจำวัน — ดูข้อจำกัดด้านล่าง |
| `POST` | `/v1/staff/appointments/walk-in` | STAFF, HC_ADMIN, SUPER_ADMIN | ลง Walk-in หน้างาน |
| `PATCH` | `/v1/staff/appointments/{id}/status` | STAFF, HC_ADMIN, SUPER_ADMIN | เปลี่ยนสถานะ — transition เดียว: CONFIRMED → COMPLETED/CANCELLED/NO_SHOW |
| `PATCH` | `/v1/staff/appointments/{id}/reassign-staff` | STAFF, HC_ADMIN, SUPER_ADMIN | ย้ายคิวไปหมออื่น — เฉพาะ CONFIRMED + `allow_staff_selection`; Audit Log |
| `POST` | `/v1/staff/appointments/{id}/unmask` | STAFF, HC_ADMIN, SUPER_ADMIN | ดูข้อมูลผู้ป่วยเต็ม (PDPA Audit Log); `throttle:unmask` 10 req/min |

**ข้อจำกัดการมองเห็น (index) — ตามจริง:**
- **STAFF** (ไม่มี role บริหาร): เห็นเฉพาะนัดของ **Assigned Services** (`service_user` pivot) ของตัวเอง; 403 ถ้า filter `service_id` ที่ไม่ได้รับมอบหมาย
- **HC_ADMIN / SUPER_ADMIN**: เห็น**ทุกบริการ**ในศูนย์ (SUPER_ADMIN ไม่ส่ง `?health_center_id` = เห็นทุกศูนย์)
- **Walk-in**: STAFF ออกได้เฉพาะ Assigned Services (403 ผิด); HC_ADMIN/SUPER_ADMIN ออกได้ทุกบริการ; 422 ถ้าบริการ "ปิดชั่วคราวเพราะเจ้าหน้าที่ทั้งหมดลางานวันนี้"
- **updateStatus / reassignStaff / unmaskPatientData**: **STAFF ทำได้ทุกคิวในศูนย์** (ไม่จำกัด assigned services)

### 3.5 การจัดการข้อมูลผู้ป่วย (Patient Data)

| Method | Endpoint | Roles ที่เข้าถึงได้ | คำอธิบาย |
|---|---|---|---|
| `PATCH` | `/v1/staff/patients/{id}/contact` | STAFF, HC_ADMIN, SUPER_ADMIN | แก้เบอร์โทร — regex 10 หลัก + unique; Audit Log (mask เบอร์) |
| `PATCH` | `/v1/staff/patients/{id}` | **HC_ADMIN, SUPER_ADMIN เท่านั้น** | แก้ข้อมูลเต็ม (ชื่อ/นามสกุล/วันเกิด/เพศ/ที่อยู่/CID) — cid 13 หลัก unique; Audit Log (mask) |
| `GET` | `/v1/staff/admin/patients` | **HC_ADMIN, SUPER_ADMIN เท่านั้น** | ค้นหาผู้ป่วย — ผู้ป่วยต้อง**เคยมีประวัติคิวที่ศูนย์นั้น** (scope ผ่าน queue history, ไม่มีคอลัมน์ health_center_id บน patient); filter q/cid/phone/gender/province/district |

**เทนเนนต์ไอโซเลชันผู้ป่วย:** ผูกผ่าน `whereHas('appointments', health_center_id=...)` — ผู้ป่วยที่ยังไม่เคยมาศูนย์ไม่โผล่

### 3.6 การจัดการบุคลากร (Roster)

| Method | Endpoint | Roles ที่เข้าถึงได้ | คำอธิบาย |
|---|---|---|---|
| `GET` | `/v1/staff/roster` | STAFF, HC_ADMIN, SUPER_ADMIN | ดูบุคลากร — `?date=` คำนวณ `effective_status=LEAVE` ถ้ามี leave วันนั้น |
| `POST` | `/v1/staff/roster` | STAFF, HC_ADMIN, SUPER_ADMIN | เพิ่มบุคลากร — **STAFF ทำได้จริง**; status ∈ ACTIVE/INACTIVE/LEAVE |
| `PATCH` | `/v1/staff/roster/{id}/toggle-duty` | STAFF, HC_ADMIN, SUPER_ADMIN | สลับ ACTIVE ↔ LEAVE เท่านั้น (INACTIVE → 422 ห้าม); ACTIVE→LEAVE บล็อก 422 ถ้ามีคิว CONFIRMED ≥1 ตั้งแต่วันนี้ |

**หมายเหตุ:** STAFF ธรรมดาสามารถเพิ่ม + สลับ duty บุคลากรคนอื่นในศูนย์เดียวกันได้ (ไม่จำกัดเฉพาะตัวเอง)

### 3.7 การจัดการวันลา (Staff Leaves)

| Method | Endpoint | Roles ที่เข้าถึงได้ | คำอธิบาย |
|---|---|---|---|
| `GET` | `/v1/staff/leaves` | STAFF, HC_ADMIN, SUPER_ADMIN | ดูวันลา — filter `?date=`, `?staff_user_id=`; response ผ่าน `StaffLeaveResource` (**ไม่มี** `created_at`/`updated_at`) |
| `POST` | `/v1/staff/leaves` | STAFF, HC_ADMIN, SUPER_ADMIN | ลงวันลา (auto-approved) — `leave_date >= today` (ย้อนหลังไม่ได้); บล็อกถ้ามีคิว CONFIRMED ในวันนั้น; **idempotent** (ซ้ำวันเดิมคืน row เดิม, HTTP 200); ถ้ายังไม่ลิงก์โปรไฟล์หมอกับบัญชี → **ลิงก์อัตโนมัติ** เมื่อระบุตัวได้ไม่กำกวม (ศูนย์เดียว [+ หมวดหมู่ที่ตรงกับบริการที่ได้รับมอบหมาย]; ถ้า ambiguos หลายโปรไฟล์ → ไม่เดา) เพื่อให้ระบบลาของหมอใน roster ถูกนำไปกรองในหน้าจองผู้ป่วยจริง |
| `DELETE` | `/v1/staff/leaves/{id}` | STAFF, HC_ADMIN, SUPER_ADMIN | ลบวันลา — scope ตามด้านล่าง |
| `GET` | `/v1/staff/leaves/availability` | STAFF, HC_ADMIN, SUPER_ADMIN | เช็คว่าบริการเปิดให้บริการวันที่นั้นหรือไม่ (`service_id`+`date` บังคับ) |

**ข้อจำกัดการจัดการวันลา (ตามจริง):**
- **STAFF**: ลงให้**ตัวเองเท่านั้น**; ลบได้เฉพาะของตัวเอง (**ถ้าไม่ใช่ → 404** "ไม่พบวันลาที่ระบุ" ซึ่งปกปิดว่ามีอยู่ — ไม่ใช่ 403)
- **HC_ADMIN**: ลงให้ใครก็ได้ในศูนย์ตัวเอง (Request rule เช็ค `health_center_id` ตรง); ลบได้เฉพาะ leave ของศูนย์ตัวเอง (อื่น → 404)
- **SUPER_ADMIN**: ลงให้ใครก็ได้; ลบได้ทุกที่
- ลงวันลาซ้ำวันเดิม → ไม่สร้างซ้ำ (firstOrCreate)

### 3.8 การดูรายชื่อ รพ.สต. (Dropdown)

| Method | Endpoint | Roles ที่เข้าถึงได้ | คำอธิบาย |
|---|---|---|---|
| `GET` | `/v1/staff/health-centers` | **ทุกคน (auth:sanctum) — ไม่มี role gate** | SUPER_ADMIN: ทุกศูนย์ (รายละเอียดเต็ม + status ทั้งหมด + has_webhook); STAFF/HC_ADMIN: ศูนย์ตัวเอง เฉพาะ ACTIVE/CLOSING (INACTIVE ซ่อน), คืนแค่ id/code/name/has_webhook |

**⚠️ หมายเหตุ route:** Route มีแค่ `auth:sanctum` (ไม่มี role gate) แต่ controller เรียก `hasRole()` ซึ่ง Patient model ไม่มี relation `roles()` → ถ้า Patient token call จะเจอ **500 (RelationNotFoundException)** ในทางปฏิบัติจึงใช้งานได้เฉพาะ staff token

---

## 4. HC Admin (ผู้ดูแลประจำศูนย์)

HC Admin มีสิทธิ์ **ทุกอย่างที่ STAFF มี** แล้วเพิ่มสิทธิ์เฉพาะดังนี้:

### 4.1 สิทธิ์เพิ่มเติมจาก STAFF

| Method | Endpoint | คำอธิบาย |
|---|---|---|
| `DELETE` | `/v1/staff/services/{id}` | ลบบริการ (STAFF ห้ามลบ) |
| `PUT` | `/v1/staff/services/{id}/staff` | ผูกหมอที่เลือกได้ (selectable staff) เข้ากับบริการ |
| `PUT` | `/v1/staff/services/{id}/time-slot-days` | ตั้งค่าวันเปิดให้บริการ (Operating Days) |
| `PATCH` | `/v1/staff/patients/{id}` | แก้ไขข้อมูลผู้ป่วยเต็มรูปแบบ |
| `GET` | `/v1/staff/admin/patients` | ค้นหารายชื่อผู้ป่วยในศูนย์ตัวเอง |

**สิทธิ์ที่ HC_ADMIN มีเพิ่ม (แต่ STAFF ไม่มี):** ดู/จัดการ user management, ดู time slots, toggle time slot ในศูนย์ตัวเอง, แก้ health center ของตัวเอง, syncStaff/syncTimeSlotDays — ดู §4.2-4.4

### 4.2 การจัดการผู้ใช้งาน (User Management)

| Method | Endpoint | คำอธิบาย |
|---|---|---|
| `GET` | `/v1/staff/admin/users` | ดูผู้ใช้ — HC_ADMIN เฉพาะศูนย์ตัวเอง; SUPER_ADMIN ทุกศูนย์ (filter ได้); response ผ่าน `AdminUserResource` — `roles` เป็น array-string, **ไม่มี** `created_at`/`updated_at`/`discord_webhook_url` |
| `POST` | `/v1/staff/admin/users` | สร้างผู้ใช้ — **HC_ADMIN จำกัด role_ids ได้เฉพาะ STAFF (422); บังคับ health_center_id ของตัวเอง**; SUPER_ADMIN สร้างได้ทุก role/ทุกศูนย์ (null ได้) |
| `PATCH` | `/v1/staff/admin/users/{id}` | แก้ผู้ใช้ — HC_ADMIN: ห้ามแก้ SUPER_ADMIN (403); ห้ามข้ามศูนย์ (404); role STAFF เท่านั้น |
| `DELETE` | `/v1/staff/admin/users/{id}` | ลบผู้ใช้ — ห้ามลบตัวเอง (422); HC_ADMIN ห้ามลบ SUPER_ADMIN (403); ห้ามลบคนสุดท้าย (ดู guard); ห้ามลบถ้ามีคิว CONFIRMED ตั้งแต่วันนี้; ลบแล้ว: staff status=INACTIVE + detach assignedServices + ลบ tokens + soft delete |
| `GET` | `/v1/staff/admin/roles` | ดู role — SUPER_ADMIN เห็นทั้งหมด; **HC_ADMIN เห็นเฉพาะ STAFF** |

**Last Super Admin Guard:** ห้าม @ลบ / @ถอด role SUPER_ADMIN / @เปลี่ยน status เป็น INACTIVE ของ SUPER_ADMIN คนสุดท้ายที่ยัง ACTIVE (เช็คผ่าน `isLastActiveSuperAdmin` — นับ user ACTIVE ที่ถือ role SUPER_ADMIN ไม่รวมตัวเอง) → 422

### 4.3 การจัดการ Time Slots

| Method | Endpoint | คำอธิบาย |
|---|---|---|
| `GET` | `/v1/staff/admin/time-slots` | ดู slot — HC_ADMIN เฉพาะศูนย์ตัวเอง; SUPER_ADMIN ทุกศูนย์ (หรือกำหนด `health_center_id`) |
| `PATCH` | `/v1/staff/admin/time-slots/{id}/toggle-status` | เปิด/ปิด slot — **HC_ADMIN เฉพาะของศูนย์ตัวเอง; SUPER_ADMIN ข้ามศูนย์ได้** |

**หมายเหตุ:** HC Admin เปิด/ปิดได้เฉพาะ slot ศูนย์ตัวเอง แต่ **Create/Update/Delete = SUPER_ADMIN only**; slot แต่ละตัวเป็นของศูนย์ (`health_center_id` FK + unique `(health_center_id, start_time, end_time)`)

### 4.4 การจัดการข้อมูล รพ.สต.

| Method | Endpoint | คำอธิบาย |
|---|---|---|
| `PATCH` | `/v1/staff/health-centers/{id}` | แก้ศูนย์ตัวเอง (ชื่อ/เบอร์/ที่อยู่/พิกัด) — **ห้ามแก้ `code`** (403); ต้องเป็นศูนย์ตัวเอง (ต่าง → 404); Audit Log |

---

## 5. Super Admin (ผู้ดูแลระบบสูงสุด)

> Super Admin มี **ทุกสิทธิ์ที่ HC Admin มี** แล้วเพิ่มสิทธิ์ข้ามศูนย์ (cross-tenant) ดังนี้:

### 5.1 Cross-Center Access

- ดู Dashboard / บริการ / นัด / roster / leaves / ผู้ป่วย / health-centers แบบ**ทุกศูนย์** (ไม่ระบุ `health_center_id` = รวมทุกศูนย์)
- **กฎหลัก:** ทุกรายการ write เฉพาะเจาะจง (service, patient, health-center, user, discord) ต้องระบุ `health_center_id` ใน request → ไม่ส่ง = **422** "SUPER_ADMIN ต้องระบุ health_center_id"; ส่งแล้วไม่มีศูนย์นั้น = 404
- **⚠️ ข้อยกเว้น (deviation):** `StaffTimeSlotController` (index/store/update/destroy/toggle-status time-slots) **ไม่บังคับ `health_center_id`** — ไม่ส่งจะได้ `healthCenterId = null` → สร้าง slot แบบ center-less ได้จริง และ update/destroy ไร้ขอบเขตศูนย์ (ต่างจาก pattern ของ trait `ResolvesHealthCenterScope` ที่ controller อื่นใช้ร่วมกัน) — น่าจะเป็น bug
- **⚠️ ข้อยกเว้น:** `POST /admin/users` ให้ SUPER_ADMIN สร้าง user แบบ center null ได้

### 5.2 Super Admin Only Endpoints

| Method | Endpoint | คำอธิบาย |
|---|---|---|
| `PATCH` | `/v1/staff/discord/webhook` | ตั้งค่า/ยกเลิก Discord Webhook (ส่ง null = ลบ) — **บังคับ `health_center_id` (422/404)**; regex URL `https://discord.com/api/webhooks/{id}/{token}` |
| `PATCH` | `/v1/staff/admin/health-centers/{id}/toggle-status` | สลับสถานะศูนย์ — ACTIVE↔CLOSING, INACTIVE→ACTIVE; **ต้องระบุ `health_center_id` ตรงกับ path {id}**; มี `lockForUpdate` + Audit Log |
| `GET` | `/v1/staff/admin/health-centers` | ทุกศูนย์ ทุก status พร้อม filter (q/code/province/district/status) + paginate |
| `POST` | `/v1/staff/admin/time-slots` | เพิ่ม slot — SUPER_ADMIN only; ⚠️ ไม่ส่ง `health_center_id` → สร้าง center-less ได้ |
| `PUT` | `/v1/staff/admin/time-slots/{id}` | แก้ slot — SUPER_ADMIN only; ⚠️ ไม่ส่ง center → แก้ได้ข้ามศูนย์ |
| `DELETE` | `/v1/staff/admin/time-slots/{id}` | ลบ slot — SUPER_ADMIN only; **⚠️ บล็อกถ้ามี appointment ใดๆ ที่ใช้ slot นี้ (ทุกวัน ทุกสถานะ)** — ไม่ใช่แค่ CONFIRMED อนาคต; error กลับ message เจาะจง "ไม่สามารถลบช่วงเวลาได้ เนื่องจากมีคิวที่ใช้ช่วงเวลานี้อยู่ กรุณาเปลี่ยนสถานะเป็นปิดใช้งานแทน" 422 |

### 5.3 Cross-Center Operations (ใช้ endpoint เดียวกับ HC Admin แต่ scope ข้ามศูนย์)

| Operation | Endpoint | ข้อจำกัดเพิ่มเติม |
|---|---|---|
| แก้ไขข้อมูลผู้ป่วย | `PATCH /v1/staff/patients/{id}` | ต้องระบุ `health_center_id`; ผู้ป่วยต้องมีประวัติคิวที่ศูนย์นั้น |
| ผูกหมอเข้ากับบริการ | `PUT /v1/staff/services/{id}/staff` | ต้องระบุ `health_center_id`; หมอต้องในศูนย์เดียวกันและหมวดหมู่เดียวกับบริการ |
| ตั้งค่าวันเปิดให้บริการ | `PUT /v1/staff/services/{id}/time-slot-days` | ต้องระบุ `health_center_id` |
| แก้ไขข้อมูล รพ.สต. | `PATCH /v1/staff/health-centers/{id}` | แก้ได้ทุกฟิลด์ **รวมถึง `code`**; `health_center_id` ต้องตรงกับ `{id}` |

### 5.4 สถานะ 3 ระดับของศูนย์สุขภาพ (Open/Close Lifecycle)

| สถานะ | ความหมาย |
|---|---|
| `ACTIVE` | เปิดรับจองตามปกติ |
| `CLOSING` | ปิดรับจองใหม่, ผู้ป่วยจองไม่ได้ (isBookable=false), แต่ staff login ได้ (เฉพาะ INACTIVE บล็อก login) + ผู้ป่วยยังเห็น/ยกเลิกนัดเดิมได้ |
| `INACTIVE` | ปิดสมบูรณ์ — ซ่อนจากผู้ป่วย (404) และ staff ใหม่ login ไม่ได้ |

**Transitions ที่ทำได้ (SUPER_ADMIN toggle):** ACTIVE→CLOSING, CLOSING→ACTIVE, INACTIVE→ACTIVE (ไม่มี ACTIVE/CLOSING→INACTIVE ผ่าน endpoint นี้ — INACTIVE เกิดจาก auto-close job เท่านั้น)

**⚠️ Auto-Close (จบตามจริง):**
- **ไม่มี scheduler/cron** — `CloseHealthCenterJob` ถูก dispatch เมื่อศูนย์อยู่ใน CLOSING และ (1) staff อัปเดตสถานะ appointment เป็น terminal (COMPLETED/CANCELLED/NO_SHOW) หรือ (2) ผู้ป่วยยกเลิกคิวผ่าน `PATCH /v1/patient/appointments/{id}/cancel`
- Job เช็ค: ศูนย์ยังเป็น CLOSING? มี CONFIRMED appointment เหลืออยู่ไหม? ถ้าไม่มี → เปลี่ยนเป็น INACTIVE + Audit Log `AUTO_CLOSE_HEALTH_CENTER`
- **⚠️ เช็ค CONFIRMED ไม่กรองวันที่** — คิว CONFIRMED ในอดีต (ที่ยังไม่ถูกเปลี่ยนสถานะ) ก็ถือว่า "ยังค้าง" → บล็อกไม่ให้ศูนย์ปิดได้ ซึ่งอาจเป็น bug

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
| **Appointments: ดูรายการ** | Assigned services only | ทุกบริการในศูนย์ | ทุกบริการทุกศูนย์ | ของตัวเอง |
| **Walk-in** | Assigned services only | ทุกบริการในศูนย์ | ทุกบริการทุกศูนย์ | — |
| **อัปเดตสถานะ / ย้ายคิว / Unmask** | ทุกคิวในศูนย์ | ทุกคิวในศูนย์ | ทุกคิวทุกศูนย์ | ยกเลิกนัดตัวเอง |
| **Patient: แก้ไขเบอร์โทร** | ✔ | ✔ | ✔ | — |
| **Patient: แก้ไขข้อมูลเต็ม** | — | ✔ | ✔ | — |
| **Patient: ค้นหารายชื่อ** | — | ✔ (ประวัติคิวในศูนย์) | ✔ (ทุกศูนย์) | — |
| **Roster: ดู/เพิ่ม/สลับ duty** | ✔ (ศูนย์ตัวเอง) | ✔ | ✔ | — |
| **Leave: ดู** | ✔ | ✔ | ✔ | — |
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

## 7. จุดเด่นด้านความปลอดภัยและ Business Logic (ตามที่ code enforce จริง)

| # | กลไก | layer ที่ enforce | หมายเหตุ |
|---|---|---|---|
| 1 | **Data Privacy (PDPA)** — ข้อมูลผู้ป่วย mask เป็นค่าเริ่มต้น; Unmask/แก้ไขบันทึก Audit Log | Resource/accessor + Controller/Service | ดูตาราง Audit Log ด้านล่าง — **ไม่ใช่ทุก action ที่ mask** |
| 2 | **Strict Multi-Tenancy** — staff จัดการเฉพาะศูนย์ตัวเอง | Trait `ResolvesHealthCenterScope` + Service `where(health_center_id)->findOrFail()` | SUPER_ADMIN ระบุ center; **ข้อยกเว้น time-slots (วางอาจผิด)** |
| 3 | **Assigned Services scope** — STAFF เห็นนัด/Walk-in เฉพาะบริการที่ assign | Controller (index/walkInBooking) | updateStatus/reassign/unmask ไม่จำกัด assigned |
| 4 | **Operating Days Hard Gate** — แต่ละ (service,slot) มี `days_mask` (bit 1-7) | Request validation (book/walk-in) + `OperatingDayService::isSlotAvailableOnDate` | pivot `service_time_slot`; slot ต้อง active |
| 5 | **Capacity & anti-overbooking** — นับ CONFIRMED ลดจาก quota; staff-selection 1 คิว/รอบ/คน | `CapacityService` + `lockForUpdate` ใน transaction | ยังมีจุด: ข้อความ booking กลืน 422, แสดงผล snapshot ไม่ lock |
| 6 | **Confirmed-queue guards** — ห้ามลบ/ปิดสิ่งที่กำลังมีคิว (services, staff, time-slot-days, user, leave, duty) | Service + Controller | เว้น DELETE time-slot ที่ block **ทุก** appointment |
| 7 | **Health Center Lifecycle** — ACTIVE/CLOSING/INACTIVE + auto-close job | Service (SUPER_ADMIN toggle) + `CloseHealthCenterJob` | auto-close เช็ค CONFIRMED ไม่กรองวันที่ (อาจ bug) |
| 8 | **Staff Selection** — `allow_staff_selection=true` บังคับ PER_MASSEUSE + selectable staff ≥1; ป้องกันการแกะหมอที่มีคิว | Service `update` + `syncStaff` | เปิด/ปิด + ผูกหมอต้องระดับ admin |
| 9 | **Last Super Admin Guard** — ห้ามลบ/ปิด SUPER_ADMIN คนสุดท้ายที่ ACTIVE | `StaffUserManagementController` + `isLastActiveSuperAdmin` | |
| 10 | **Audit Trail** — 11 actions รายละเอียดด้านล่าง | Service/Controller/Job | |

### 7.1 ตาราง Audit Log (action + ข้อมูลที่เก็บ/mask ตามจริง)

| Action | เกิดขึ้นเมื่อ | ข้อมูลใน payload | Mask? |
|---|---|---|---|
| `UNMASK_PATIENT_DATA` | staff ดูข้อมูลผู้ป่วยเต็ม | `appointment_id` (ข้อมูลจริงคืน client แต่ไม่เก็บใน log) | — |
| `UPDATE_PATIENT_CONTACT` | แก้เบอร์โทรผู้ป่วย | `field=phone_number, old, new` | เบอร์ mask `063-***-1234` |
| `UPDATE_PATIENT_PROFILE` | แก้ข้อมูลผู้ป่วยเต็ม | per-field `field/old/new` | cid mask `1-****-*****-99`, เบอร์ mask; ฟิลด์อื่นเก็บเต็ม |
| `UPDATE_HEALTH_CENTER` | แก้ข้อมูล รพ.สต. | per-field old/new | **ไม่ mask** (ค่าเต็ม) |
| `UPDATE_HEALTH_CENTER_STATUS` | toggle สถานะศูนย์ | `from, to` | — |
| `AUTO_CLOSE_HEALTH_CENTER` | auto-close job | `from=CLOSING, to=INACTIVE` (user_id=null) | — |
| `APPOINTMENT_STAFF_REASSIGN` | ย้ายคิวไปหมออื่น | `before_staff_id, after_staff_id` | — |
| `SERVICE_SELECTABLE_STAFF_SYNC` | ผูกหมอที่เลือกได้ | `before, after` (staff id arrays) | — |
| `SERVICE_TIME_SLOT_DAYS_SYNC` | ตั้ง operating days | `before, after` (slot + days) | — |
| `USER_CREATED` | สร้าง user | `username, name, health_center_id, role_ids` | **ไม่ mask** |
| `USER_UPDATED` | แก้ user | `before/after: name, status, role_ids` | **ไม่ mask** |
| `USER_DELETED` | ลบ user | `username, name, health_center_id, staff_ids` | **ไม่ mask** |

> ⚠️ เอกสารฉบับเก่าเขียนว่า "ทุกการ Unmask / แก้ไขถูกบันทึกพร้อม Mask ใน Log เสมอ" — **ไม่จริง** ตาม code: หลาย action (health-center, user) เก็บค่าเต็มโดยไม่ mask; `audit_logs` มีเพียงคอลัมน์ `payload` (json) ไม่มีคอลัมน์ `masked_data`

---

## 8. ตาราง Endpoint ทั้งหมด (เทียบกับ routes/api.php)

### 8.1 Staff

| Method | Endpoint | Role gate (middleware) |
|---|---|---|
| POST | `/v1/staff/login` | public (`throttle.staff`) |
| GET | `/v1/staff/me` | auth:sanctum |
| PATCH | `/v1/staff/me` | auth:sanctum |
| POST | `/v1/staff/logout` | auth:sanctum |
| GET | `/v1/staff/dashboard/summary` | STAFF,HC_ADMIN,SUPER_ADMIN |
| GET | `/v1/staff/services` | STAFF,HC_ADMIN,SUPER_ADMIN |
| POST | `/v1/staff/services` | STAFF,HC_ADMIN,SUPER_ADMIN |
| PUT | `/v1/staff/services/{id}` | STAFF,HC_ADMIN,SUPER_ADMIN |
| PATCH | `/v1/staff/services/{id}/toggle-status` | STAFF,HC_ADMIN,SUPER_ADMIN |
| DELETE | `/v1/staff/services/{id}` | STAFF,HC_ADMIN,SUPER_ADMIN (controller gate: HC_ADMIN+SUPER_ADMIN) |
| GET | `/v1/staff/appointments` | STAFF,HC_ADMIN,SUPER_ADMIN |
| POST | `/v1/staff/appointments/walk-in` | STAFF,HC_ADMIN,SUPER_ADMIN |
| PATCH | `/v1/staff/appointments/{id}/status` | STAFF,HC_ADMIN,SUPER_ADMIN |
| PATCH | `/v1/staff/appointments/{id}/reassign-staff` | STAFF,HC_ADMIN,SUPER_ADMIN |
| POST | `/v1/staff/appointments/{id}/unmask` | STAFF,HC_ADMIN,SUPER_ADMIN + `throttle:unmask` |
| GET | `/v1/staff/roster` | STAFF,HC_ADMIN,SUPER_ADMIN |
| POST | `/v1/staff/roster` | STAFF,HC_ADMIN,SUPER_ADMIN |
| PATCH | `/v1/staff/roster/{id}/toggle-duty` | STAFF,HC_ADMIN,SUPER_ADMIN |
| PATCH | `/v1/staff/patients/{id}/contact` | STAFF,HC_ADMIN,SUPER_ADMIN |
| PUT | `/v1/staff/services/{id}/assignees` | STAFF,HC_ADMIN,SUPER_ADMIN |
| GET | `/v1/staff/leaves` | STAFF,HC_ADMIN,SUPER_ADMIN |
| POST | `/v1/staff/leaves` | STAFF,HC_ADMIN,SUPER_ADMIN |
| DELETE | `/v1/staff/leaves/{id}` | STAFF,HC_ADMIN,SUPER_ADMIN |
| GET | `/v1/staff/leaves/availability` | STAFF,HC_ADMIN,SUPER_ADMIN |
| PATCH | `/v1/staff/discord/webhook` | SUPER_ADMIN |
| PATCH | `/v1/staff/admin/health-centers/{id}/toggle-status` | SUPER_ADMIN |
| GET | `/v1/staff/admin/health-centers` | SUPER_ADMIN |
| GET | `/v1/staff/admin/users` | SUPER_ADMIN,HC_ADMIN |
| POST | `/v1/staff/admin/users` | SUPER_ADMIN,HC_ADMIN |
| PATCH | `/v1/staff/admin/users/{id}` | SUPER_ADMIN,HC_ADMIN |
| DELETE | `/v1/staff/admin/users/{id}` | SUPER_ADMIN,HC_ADMIN |
| GET | `/v1/staff/admin/roles` | SUPER_ADMIN,HC_ADMIN |
| GET | `/v1/staff/admin/time-slots` | SUPER_ADMIN,HC_ADMIN |
| PATCH | `/v1/staff/admin/time-slots/{id}/toggle-status` | SUPER_ADMIN,HC_ADMIN |
| POST | `/v1/staff/admin/time-slots` | SUPER_ADMIN |
| PUT | `/v1/staff/admin/time-slots/{id}` | SUPER_ADMIN |
| DELETE | `/v1/staff/admin/time-slots/{id}` | SUPER_ADMIN |
| PATCH | `/v1/staff/patients/{id}` | SUPER_ADMIN,HC_ADMIN |
| GET | `/v1/staff/admin/patients` | SUPER_ADMIN,HC_ADMIN |
| PATCH | `/v1/staff/health-centers/{id}` | SUPER_ADMIN,HC_ADMIN |
| GET | `/v1/staff/health-centers` | auth:sanctum (ไม่มี role gate) |
| PUT | `/v1/staff/services/{id}/time-slot-days` | SUPER_ADMIN,HC_ADMIN |
| PUT | `/v1/staff/services/{id}/staff` | SUPER_ADMIN,HC_ADMIN |

### 8.2 Patient

| Method | Endpoint | Role gate (middleware) |
|---|---|---|
| POST | `/v1/patient/login` | public (`throttle:patient-login`) |
| POST | `/v1/patient/register` | public (`throttle:patient-register`) |
| GET | `/v1/patient/categories` | public |
| GET | `/v1/patient/right-types` | public |
| POST | `/v1/patient/nearby-centers` | public |
| GET | `/v1/patient/health-centers/{id}` | public |
| GET | `/v1/patient/health-centers/{id}/services` | public |
| GET | `/v1/patient/available-slots` | public |
| GET | `/v1/patient/me` | auth:sanctum + abilities:role:patient |
| POST | `/v1/patient/book` | auth:sanctum + abilities:role:patient |
| GET | `/v1/patient/services/{id}/staff` | auth:sanctum + abilities:role:patient |
| GET | `/v1/patient/appointments` | auth:sanctum + abilities:role:patient |
| GET | `/v1/patient/appointments/{id}` | auth:sanctum + abilities:role:patient |
| PATCH | `/v1/patient/appointments/{id}/cancel` | auth:sanctum + abilities:role:patient |

---

## 9. สรุป Gap / จุดที่ควรแก้ไขใน code (พบระหว่างการตรวจ verify)

| # | จุด | ประเภท | ผลกระทบ |
|---|---|---|---|
| 1 | staff login throttle นับเฉพาะ 401/403 → login ผิด (422) ไม่ทำให้ลิมิตทำงาน | bug | ไม่มีป้องกัน brute-force staff login |
| 2 | `StaffTimeSlotController` ไม่บังคับ `health_center_id` → SUPER_ADMIN สร้าง slot center-less / แก้ข้ามศูนย์ได้ | gap/consistency | ละเมิดกฎ cross-tenant; ข้อมูลไม่ผูกศูนย์ |
| 3 | Auto-close เช็ค CONFIRMED โดยไม่กรองวันที่ → คิวเก่าในอดีตบล็อกการปิดศูนย์ | bug | ศูนย์อาจไม่ปิดเป็น INACTIVE |
| 4 | booking error ทั้งหมด กลืนเป็นข้อความเดียว | design | ผู้ใช้รู้สาเหตุไม่ได้; debug ต้องดู log |
| 5 | DELETE time-slot บล็อก**ทุก** appointment (ทุกวันที่/สถานะ) แม้จะตั้งใจจะบล็อกเฉพาะคิวล่วงหน้า | bug | ลบ slot เก่าไม่ได้ |
| 6 | patient booking ไม่เช็ค "staff ทั้งหมดลา" (ต่างจาก walk-in) — ✅ **แก้แล้ว**: BookingService เรียก `StaffLeave::isServiceAvailable()` เหมือน walk-in; pool ว่าง → capacity ตัดสิน | fixed | — |
| 7 | `GET /v1/staff/health-centers` ถ้า call ด้วย Patient token → 500 | gap | route ควรมี role gate |
| 8 | role middleware fallback token ability dead (`role:staff` vs `STAFF`) | dead code | สร้างความเข้าใจผิด; ควรลบ fallback หรือแก้ case |