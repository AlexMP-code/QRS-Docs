# คู่มือลำดับการเรียก API ทุก Feature (QRS)

> **อัปเดตเมื่อ:** 2026-10-01
> **อ้างอิง code:** `6db006a` (QRS-QueueReservationSystem, branch master)
> **อ้างอิงเอกสาร:** `Roles.md` (สิทธิ์/ขอบเขต) · `openapi.yaml` (schema) · `Feature-First.yaml` (flow ตามหน้าจอ)
>
> เอกสารนี้เขียนให้ตรงกับ **พฤติกรรมจริงของ code**
>
> **การอ้างอิงตำแหน่งในโค้ด** ใช้รูปแบบ `ไฟล์.php` หรือ `ClassName::method()` เท่านั้น
> ตัวเลขบรรทัดถูกตัดออกเพราะหมดอายุทันที่มีใครแก้ไฟล์นั้น — ซึ่งเกิดขึ้นเสมอ
> เมื่อต้องการรู้ว่าตรงกับโค้ดจริงหรือไม่ ให้ grep ตามชื่อฟังก์ชันแทน

---

## สารบัญ

- [Part 0 — กติกากลางที่ต้องรู้ก่อน](#part-0--กติกากลางที่ต้องรู้ก่อน)
- [Part A — ฝั่งผู้รับบริการ](#part-a--ฝั่งผู้รับบริการ)
  - [A1. ค้นหาศูนย์ + ดูบริการ](#a1-ค้นหาศูนย์--ดูบริการ-public)
  - [A2. สมัครสมาชิก / เข้าสู่ระบบ](#a2-สมัครสมาชิก--เข้าสู่ระบบ)
  - [A3. จองคิว — บริการที่เปิดให้เลือกหมอ](#a3-จองคิว--บริการที่เปิดให้เลือกหมอ-allow_staff_selection--true)
  - [A4. จองคิว — บริการที่ไม่เปิดเลือกหมอ](#a4-จองคิว--บริการที่ไม่เปิดเลือกหมอ-allow_staff_selection--false)
  - [A5. ประวัติการจอง + ยกเลิก](#a5-ประวัติการจอง--ยกเลิก)
- [Part B — ฝั่งเจ้าหน้าที่](#part-b--ฝั่งเจ้าหน้าที่)
  - [B1. Login / โปรไฟล์ตัวเอง / Logout](#b1-login--โปรไฟล์ตัวเอง--logout)
  - [B2. Dashboard](#b2-dashboard)
  - [B3. จัดการบริการ](#b3-จัดการบริการ)
  - [B4. เปิดเลือกหมอ (สำคัญ — ต้องทำตามลำดับ)](#b4-เปิดเลือกหมอ-สำคัญ--ต้องทำตามลำดับ)
  - [B5. มอบหมายผู้รับผิดชอบบริการ](#b5-มอบหมายผู้รับผิดชอบบริการ)
  - [B6. จัดการช่วงเวลา (Time Slots)](#b6-จัดการช่วงเวลา-time-slots)
  - [B7. โต๊ะเคาน์เตอร์คิว](#b7-โต๊ะเคาน์เตอร์คิว)
  - [B8. Roster (ทะเบียนบุคลากร/หมอนวด)](#b8-roster-ทะเบียนบุคลากรหมอนวด)
  - [B9. วันลา](#b9-วันลา)
  - [B10. ค้นหาและแก้ข้อมูลผู้รับบริการ](#b10-ค้นหาและแก้ข้อมูลผู้รับบริการ)
  - [B11. จัดการศูนย์สุขภาพ](#b11-จัดการศูนย์สุขภาพ)
  - [B12. จัดการผู้ใช้ (บัญชี staff)](#b12-จัดการผู้ใช้-บัญชี-staff)
- [Part C — Invariant และข้อจำกัดที่ต้องระวัง](#part-c--invariant-และข้อจำกัดที่ต้องระวัง)

**อ่านก่อนใช้งาน — สี่ข้อที่ทำให้พลาดบ่อยที่สุด:**

| # | ข้อ | ดู |
|---|---|---|
| 1 | ศูนย์ (ผู้รับบริการของใคร) ≠ บริการ (ใครทำงานกับคิวนี้ได้) — มี 2 ขอบเขตแยกกัน | [0.3](#03-กฎ-health_center_id-สำคัญที่สุดของระบบนี้) · [0.3a](#03a-ขอบเขตระดับบริการ-ขอบเขตที่สอง-นอกเหนือจากศูนย์) |
| 2 | `users` = บัญชีเข้าใช้ · `staff` = โปรไฟล์คนงาน — 1:1 | [0.3b](#03b-users--บัญชี--staff--โปรไฟล์คนงาน) |
| 3 | มีโควตาต่อนาทีทุก endpoint — ได้ 429 เมื่อชน | [B2.3](#b23-โควตาการเรียก-api-rate-limit) |

> หมายเหตุเรื่องตัวเลข: เอกสารนี้จงใจ**ไม่บันทึกตัวเลขจำนวนศูนย์ จำนวนบริการ หรือจำนวนบทบาท**
> เพราะเป็นค่าจากฐานข้อมูลจริงที่เปลี่ยนไปเรื่อย ๆ และเคยทำให้คนอ่านเชื่อว่าเป็นค่าปัจจุบัน
> ให้นับจากฐานข้อมูลที่ใช้งานจริงเสมอ
| 4 | ตัวเลขสรุปต้องตรงกับรายการที่เห็น — ใช้ bundle เพื่อให้แน่ใจ | [B2.2](#b22-หน้าจอหนึ่งจอ--สามชิ้นในคำขอเดียว-แนะนำ) |

---

# Part 0 — กติกากลางที่ต้องรู้ก่อน

## 0.1 รูปแบบ response

**สำเร็จ**
```json
{ "success": true, "message": "...", "data": ... }
```

**ล้มเหลว** (controller ที่ทำเอง)
```json
{ "success": false, "message": "ข้อความภาษาไทย" }
```

**ล้มเหลวจาก FormRequest** (validation) — เป็น JSON ของ Laravel มี `message` + `errors` ต่อ field
```json
{ "message": "ข้อความแรกที่เจอ", "errors": { "field": ["ข้อความ"] } }
```

> ⚠️ `errors` จะมีเฉพาะ validation ระดับ Request — ดูหัวข้อ 0.6 เรื่อง error ที่ถูกกลืน

## 0.2 Token

| ฝั่ง | ได้ token จาก | Token ability | Middleware |
|---|---|---|---|
| Staff | `POST /v1/staff/login` | `role:staff` | `auth:sanctum` + `role:STAFF,HEALTH_CENTER_ADMIN,SUPER_ADMIN` |
| Patient | `POST /v1/patient/login` หรือ `/register` | `role:patient` | `auth:sanctum` + `abilities:role:patient` |

ส่งทุก request หลัง login: `Authorization: Bearer <token>`

- **Patient token → staff route = 403** (ไม่มี role ใน `users`)
- **Staff token → patient route = 403** (ability ไม่ใช่ `role:patient`)

## 0.3 กฎ `health_center_id` (สำคัญที่สุดของระบบนี้)

ระบบเป็น **multi-tenancy แบบ strict** — ผ่าน trait `ResolvesHealthCenterScope` (app/Traits/ResolvesHealthCenterScope.php)

| ประเภท | Method ที่ใช้ | STAFF / HC_ADMIN | SUPER_ADMIN |
|---|---|---|---|
| **อ่าน / list** | `resolveFilterHealthCenterId()` | บังคับศูนย์ตัวเอง ส่งมาก็ไม่มีผล | ไม่ส่ง = **ดูทุกศูนย์**; ส่ง = กรองศูนย์นั้น |
| **เขียน / เจาะจง** | `resolveTargetHealthCenterId()` | บังคับศูนย์ตัวเอง | **ต้องส่งเสมอ** — ไม่ส่ง = `422 "SUPER_ADMIN ต้องระบุ health_center_id"`; ส่ง id ที่ไม่มี = `404` |

> ✅ ข้อยกเว้นที่เคยมี **แก้แล้ว** — ดูหัวข้อ C.1
> ปัจจุบันทุก endpoint ใช้ตัวแก้ scope กลางนี้ ไม่มีทางลัดแล้ว
>
> ⚠️ การเรียก `resolveTargetHealthCenterId()` ต้องอยู่ **นอก `try`**
> เพราะมันเรียก `abort()` ซึ่งถูก `catch` ทั่วไปกินแล้วกลายเป็น 500
> เมื่อโค้ดใหม่เพิ่ม endpoint ต้องระวังข้อนี้

## 0.3a ขอบเขตระดับบริการ (ขอบเขตที่สอง นอกเหนือจากศูนย์)

ศูนย์ตอบว่า "ข้อมูลนี้อยู่ศูนย์ไหน" — ส่วน**ขอบเขตระดับบริการ** ตอบว่า "คนที่มีบทบาท STAFF ทำอะไรได้"

เกณฑ์นี้อยู่ที่ `ResolvesHealthCenterScope` จุดเดียว ใช้โดยทุกทางที่ดูหรือแตะคิว

**คิวเป็นของศูนย์ ไม่ใช่ของคน** — การมอบหมายบริการเป็นเรื่อง "ใครมีอำนาจทำงานกับคิวนี้" ไม่ใช่ความเป็นเจ้าของคิว

**STAFF ผ่านเมื่อเข้าเงื่อนไขอย่างน้อยหนึ่งทาง:**
1. **ได้รับมอบหมายบริการนั้น** — มีแถวใน `service_user`
2. **เป็นผู้ถูกระบุในคิวนั้น** — `appointments.staff_id` ชี้ไปที่โปรไฟล์ของเขา

ทางที่สองมีไว้เพราะคนที่ให้บริการจริงย่อมรู้ว่าเกิดอะไรขึ้นกับคิวนั้น และบางคนไม่ได้อยู่ในบริการที่ตนถูกมอบหมาย

**ที่ใช้เกณฑ์นี้:**

| ทาง | ถูกปฏิเสธเมื่อ |
|---|---|
| `GET /staff/appointments` | กรองรายการให้เห็นเฉพาะบริการที่มอบหมาย + filter `service_id` ที่ไม่ได้มอบหมาย → 403 |
| `PATCH .../status` | ไม่เข้าเงื่อนไขทั้งสองทาง → 403 |
| `PATCH .../reassign-staff` | เหมือนกัน |
| `POST .../unmask` | เหมือนกัน — และ **ไม่เขียน audit log** เมื่อถูกปฏิเสธ |
| `GET /staff/dashboard/summary` · `/bundle` | นับเฉพาะบริการที่มอบหมาย |
| `PATCH /staff/patients/{id}/contact` | ผู้รับบริการต้องเคยมีคิวในบริการที่มอบหมาย → 403 |

**สิ่งที่ไม่ถูกจำกัด:**
- **รายชื่อบุคลากร (`GET /staff/roster`) = ทั้งศูนย์** เสมอ สำหรับ STAFF ด้วย
  เพราะใครเข้าเวรหรือลาวันนี้เป็นเรื่องของศูนย์ ไม่ใช่ของบริการเดียว
- **HC_ADMIN / SUPER_ADMIN** = ไม่ถูกจำกัดทุกเส้นทาง
- **หน้าเคาน์เตอร์ (`POST /walk-in`)** = ของผู้ดูแลศูนย์เท่านั้น STAFF เรียกไม่ได้เลย (ดู B7.3)

**บริการที่ไม่ผูกชื่อบุคลากร** (นับตามช่วงเวลา/ทั้งวัน) คิวจะไม่มีผู้ถูกระบุ → ทางที่สองใช้ไม่ได้ ต้องได้รับมอบหมายหรือเป็นผู้ดูแลเท่านั้น

**403 ไม่ใช่ 404** — เพราะเจ้าหน้าที่ปฏิบัติงานเห็นคิวนั้นอยู่ในหน้าจออยู่แล้ว การตอบว่าไม่พบจะทำให้เขากดซ้ำโดยไม่รู้สาเหตุ

**ประดับขอบเขตคือระดับบริการ ไม่ใช่ระดับคน** — ผู้ที่ได้รับมอบหมายบริการหนึ่ง ทำงานกับทุกคิวของบริการนั้นได้ แม้ผู้รับบริการจะเลือกบุคลากรคนอื่น

## 0.3b `users` = บัญชี · `staff` = โปรไฟล์คนงาน

สองคำนี้คนละชั้น — สับสนบ่อยที่สุดในการเขียน frontend

| | `users` | `staff` |
|---|---|---|
| คือ | บัญชีเข้าใช้งาน (มี username/password) | โปรไฟล์คนงาน |
| มี role | ✅ (`user_roles`) | — |
| มี `health_center_id` | ✅ บังคับ NOT NULL (รวมผู้ดูแลระบบ) | ✅ |
| ความสัมพันธ์ | `users.id` ↔ `staff.user_id` | **1:1** (`uq_staff_user_id` unique) |
| ใช้ทำอะไร | login, สิทธิ์, โควตานับ | เข้าเวร, วันลา, โควตาต่อหมอ, ผูกบริการ |

- **ทุกบัญชีที่สร้างต้องมีโปรไฟล์ `staff`** — สร้างในคำขอเดียวกัน (ดู B12.3)
- **โปรไฟล์เก่าที่ไม่มีบัญชียังทำงานได้** — `staff.user_id` เป็น nullable จึงยังให้บริการและนับโควตาได้ เขาแค่ลาไม่ได้
- **บัญชีที่ไม่มีโปรไฟล์ ลงวันลาไม่ได้** (422) เพราะระบบไม่รู้ว่าคนนี้คือใคร การเดาจากหมวดหมู่เคยทำให้ผูกผิดคน
- **ลบบัญชี** → โปรไฟล์ถูกตั้ง `user_id = null` + `status = INACTIVE` (ดู B12.5) ทำให้ผู้รับบริการเลือกจองเขาไม่ได้อีก

## 0.4 `capacity_type` — 3 แบบ (กำหนดว่านับโควตายังไง)

`CapacityService::getMaxCapacity()` (app/Services/CapacityService.php)

| ค่า | โควตาต่อรอบเวลามาจาก | ต้องส่งตอนสร้าง | ลักษณะ |
|---|---|---|---|
| `PER_MASSEUSE` | นับบุคลากรที่ **ผูกกับบริการนี้โดยตรง** (`service_user`) และยังให้บริการอยู่ ไม่ใช่ทั้งหมวดหมู่ | — | 1 คิว/หมอ/รอบ |
| `PER_SLOT` | `service_capacity.max_per_slot` (ค่า default 1) | `max_per_slot` | ตายตัวต่อรอบเวลา |
| `PER_DAY` | `service_capacity.max_per_day` (ค่า default 10) | `max_per_day` | **นับรวมทั้งวัน ไม่ล้างทุกรอบ** |

> `PER_MASSEUSE` นับจากคนที่ผูกกับบริการ ไม่ใช่คนทั้งหมวดหมู่ เพราะบุคลากรในหมวดเดียวกันอาจไม่ได้รับผิดชอบบริการนี้
> ดู `CapacityService::countAssignedStaff()` (app/Services/CapacityService.php)

**สองอันดับที่คนสับสนบ่อย:**
1. **`allow_staff_selection=true` override ทุกอย่าง** — ข้าม `capacity_type` ไปนับ `selectableStaff()` แทน (CapacityService.php)
2. **โควตา "ต่อหมอ = 1 คิว/รอบ"** (`PER_STAFF_PER_SLOT_LIMIT = 1`, CapacityService.php) บังคับ**เฉพาะ**ตอนเลือกหมอ ไม่เกี่ยวกับ `capacity_type` อื่น

## 0.5 วงจรชีวิตศูนย์สุขภาพ

```
ACTIVE ──toggle-status──> CLOSING ──คิว CONFIRMED หมด──> INACTIVE
   │                        │
   │                        └── toggle-status ──> ACTIVE (เปิดใหม่ได้)
   └── toggle-status ──> CLOSING
```

| สถานะ | รับจองใหม่ | staff login | เห็นใน patient API |
|---|---|---|---|
| `ACTIVE` | ✅ | ✅ | ✅ |
| `CLOSING` | ❌ | ✅ (ยังจัดการคิวค้างได้) | ✅ |
| `INACTIVE` | ❌ | ❌ 403 | ❌ 404 |

- `HealthCenter::isBookable()` = `status === 'ACTIVE'` (app/Models/HealthCenter.php)
- Login บล็อกเฉพาะ INACTIVE (app/Http/Controllers/Api/Staff/StaffAuthController.php)
- **Auto-close ไม่มี cron** — ถูก dispatch เมื่อ staff เปลี่ยนสถานะคิวเป็น terminal หรือผู้รับบริการกดยกเลิก ขณะศูนย์เป็น CLOSING (`StaffAppointmentService.php`, `PatientHistoryController.php`) → `CloseHealthCenterJob` เช็คว่าไม่มีคิว CONFIRMED แล้วเปลี่ยนเป็น INACTIVE + audit (app/Jobs/CloseHealthCenterJob.php)

## 0.6 error สองชนิดที่ต้องแยกให้ออก

`POST /v1/patient/book` แยกข้อผิดพลาดออกเป็นสองชนิด (PatientBookingController.php)

| ชนิด | สถานะ | ผู้รับบริการเห็น |
|---|---|---|
| **เงื่อนไขทางธุรกิจ** (`BookingException`) | ตามรหัสที่กำหนด เช่น 422 / 409 | ข้อความจริง เช่น `"ท่านมีรายการจองบริการนี้ในวันที่เลือกเรียบร้อยแล้ว"` |
| **สิทธิการรักษาไม่ตรงกับผู้รับบริการ** | 422 | `"สิทธิการรักษาที่ระบุไม่พบในข้อมูลของผู้รับบริการคนนี้"` — ปฏิเสธแทนที่จะเงียบแล้วสร้างนัดหมายที่ไม่มีสิทธิผูกไว้ (BookingService.php) |

| **ข้อผิดพลาดของระบบ** (อย่างอื่น) | 500 | `"ไม่สามารถจองคิวได้ กรุณาลองใหม่อีกครั้ง"` — ไม่เผยรายละเอียดภายใน |

> ถ้า debug แล้วจองไม่ผ่าน แต่ข้อความเป็นกลาง แปลว่าตกไปที่ชั้นระบบ → ดู `storage/logs/laravel.log` (มี `Booking failed: <ข้อความจริง>`)
> validation ของ FormRequest (เช่น `staff_id` ไม่ผูกกับบริการ) ไม่ผ่านเส้นทางนี้ จึงตอบ 422 พร้อม `errors` ชัดเจนเสมอ

## 0.7 เลขคิว (queue number)

- `appointment_number` = `max(appointment_number)` ของ `(health_center_id, appointment_date)` + 1
- `queue_no` = `"Q-"` + เติมศูนย์ข้างหน้า 3 หลัก → `Q-001`, `Q-002`
- **รีเซ็ตทุกวัน และใช้ร่วมกันทุกบริการของศูนย์นั้น** (ไม่ใช่เลขแยกรายบริการ)
- `BookingService.php`

---

# Part A — ฝั่งผู้รับบริการ

## A1. ค้นหาศูนย์ + ดูบริการ (public)

> ไม่ต้อง login ทุก endpoint ในหัวข้อนี้

### A1.1 ได้หมวดหมู่บริการ (ต้องมีก่อนค้นหา)

**`GET /v1/patient/categories`**
คืนหมวดหมู่ที่ `status=ACTIVE` เรียงตาม `sort_order` — field: `id`, `name`, `icon`
`PatientLocationController.php`

### A1.2 ได้สิทธิการรักษา (ต้องมีก่อนสมัครสมาชิก)

**`GET /v1/patient/right-types`**
คืนสิทธิที่ `status=ACTIVE` — field: `id`, `code`, `name`, `description`
`PatientLocationController.php`

### A1.3 ค้นหาศูนย์ใกล้เคียง

**`POST /v1/patient/nearby-centers`**
```json
{ "latitude": 13.7563, "longitude": 100.5018, "category_id": 1 }
```
| field | rule |
|---|---|
| `latitude` | required, numeric, -90..90 |
| `longitude` | required, numeric, -180..180 |
| `category_id` | required, `exists:categories,id` |

คืนเฉพาะศูนย์ **`ACTIVE`** ที่ **มีบริการของหมวดหมู้นั้นและ `is_active=true`** เรียงตาม `distance_km` จริง (สูตร Haversine)
พร้อมบริการของหมวดหมู้นั้นฝังมาให้ → ใช้เลือกศูนย์ได้เลยโดยไม่ต้องเปิด detail
`NearbyCentersRequest.php` · `LocationService.php`

### A1.4 ดูรายละเอียดศูนย์

**`GET /v1/patient/health-centers/{id}`**
เห็นได้ทั้ง `ACTIVE` และ `CLOSING`; `INACTIVE` → **404**
`PatientLocationController.php`

### A1.5 ดูบริการของศูนย์

**`GET /v1/patient/health-centers/{id}/services`**
เห็นได้เฉพาะบริการที่ผ่าน **4 เงื่อนไขพร้อมกัน** (`PatientLocationController.php`):
1. `services.is_active = true`
2. มี pivot `service_time_slot` ≥ 1 แถว (คือมีการตั้งวันเปิดให้บริการแล้ว)
3. `category.status = ACTIVE`
4. ศูนย์เป็น ACTIVE/CLOSING

แต่ละรายการคืน: `id`, `name`, `description`, `capacity_type`, **`allow_staff_selection`**, `category`, `capacity`, `time_slots[]`
envelope มี `health_center_status` ด้วย → **เช็คสถานะศูนย์ได้จากที่นี่ที่เดียว**

> หมายเหตุ: endpoint นี้**ไม่**กรองบริการที่หมอลาหมด (ต่างจาก staff `services?date=`) — ผู้รับบริการจะเห็นบริการแล้วค่อยพบว่าเต็มตอนกดดูช่วงเวลา

---

## A2. สมัครสมาชิก / เข้าสู่ระบบ

> ระบบผู้รับบริการ **ไม่ใช้ password** — ยืนยันตัวตนด้วย **เบอร์โทร (10 หลัก) + วันเดือนปีเกิด**

### A2.1 ลงทะเบียนใหม่

**`POST /v1/patient/register`** (throttle `patient-register`)
```json
{
  "cid": "5100000000011",
  "first_name": "สมชาย",
  "last_name": "ใจดี",
  "phone_number": "0899876543",
  "birth_date": "1990-01-01",
  "gender": "MALE",
  "right_type_id": 1,
  "subdistrict": "...", "district": "...", "province": "...",
  "full_address": "...",
  "main_hospital_name": "..."
}
```

| field | rule |
|---|---|
| `cid` | required, **13 หลักตัวเลข**, unique ใน `patients` |
| `phone_number` | required, **10 หลักตัวเลข**, unique ใน `patients` |
| `birth_date` | required, `Y-m-d` |
| `gender` | optional, `MALE/FEMALE/OTHER` |
| `right_type_id` | **required**, `exists:right_types,id` |
| `main_hospital_name` | optional, max 255 |

สร้าง `patients` + `patient_rights` (ตั้ง `is_primary=true`) ใน transaction เดียว แล้วออก token ให้เลย
→ **201** `{ success, message, token, patient: { id, masked_name } }`
`PatientRegisterRequest.php` · `PatientAuthController.php`

### A2.2 เข้าสู่ระบบ

**`POST /v1/patient/login`** (throttle `patient-login`)
```json
{ "phone_number": "0899876543", "birth_date": "1990-01-01" }
```

**3 ผลลัพธ์ที่ต้องแยกให้ออก:**

| สถานการณ์ | HTTP | คือ | client ต้องทำ |
|---|---|---|---|
| เบอร์ไม่มีในระบบ | **200** | `is_registered: false, token: null` | พาไปหน้าสมัคร |
| วันเกิดผิด (ครบ 5 ครั้ง) | **423** | `"บัญชีถูกระงับชั่วคราว 30 นาที"` | ปิดปุ่ม login |
| สำเร็จ | **200** | `is_registered: true, token, patient{id, masked_name, masked_phone}` | เก็บ token |

- วันเกิดผิดครั้งที่ 1-4 → **401** พร้อม `"เหลือโอกาสอีก N ครั้ง"` และ `failed_login_attempts++`
- ครบ 5 ครั้ง → ตั้ง `locked_until = now()+30min` → รอบถัดไปตอบ 423 ทันที
- login สำเร็จ → reset `failed_login_attempts = 0`, `locked_until = null`
`PatientAuthService.php`

### A2.3 ดูโปรไฟล์ตัวเอง

**`GET /v1/patient/me`** (ต้อง token)
คืนข้อมูลแบบ **masked** (ชื่อ/เบอร์/เลขบัตรถูกปิดบัง) + `rights[]`
`PatientAuthController.php`

---

## A3. จองคิว — บริการที่เปิดให้เลือกหมอ (`allow_staff_selection = true`)

> เงื่อนไข: service ต้องเป็น `capacity_type = PER_MASSEUSE` **และ** มีหมอผูกใน `service_staff` แล้ว
> เงื่อนไขของ `capacity_type` อยู่ใน 0.4 — บริการที่จะเปิดเลือกหมอต้องเป็น `PER_MASSEUSE` และมีหมอผูกใน `service_staff` แล้ว

### ขั้นค้นหา (public ไม่ต้อง login)

**1. `GET /v1/patient/health-centers/{id}`** — ดูศูนย์
> ถ้ายังไม่รู้ศูนย์ ใช้ `POST /v1/patient/nearby-centers` (A1.3) ค้นก่อน

**2. `GET /v1/patient/health-centers/{id}/services`** — ได้ `service_id` ของบริการนวด + ตรวจ `allow_staff_selection: true`

**3. `GET /v1/patient/available-slots?service_id={id}&date=YYYY-MM-DD`**

| query | rule |
|---|---|
| `service_id` | required, ต้องเป็นบริการที่ `is_active=true` และยังไม่ถูก soft delete |
| `date` | required, `Y-m-d`, **≥ วันนี้** (ย้อนหลังไม่ได้) |

`AvailableSlotsRequest.php`

**response แบบเปิดเลือกหมอ** — แต่ละ slot มี `staff[]`:
```json
{
  "success": true,
  "service_name": "นวดแผนไทย...",
  "capacity_type": "PER_MASSEUSE",
  "date": "2026-09-25",
  "slots": [{
    "time_slot_id": 1, "time_label": "09:00-09:30",
    "start_time": "09:00", "end_time": "09:30",
    "max_capacity": 2,
    "available_count": 2, "is_full": false,
    "staff": [
      { "staff_id": 1, "name": "...", "position": "หมอนวด", "available": true },
      { "staff_id": 2, "name": "...", "position": "หมอนวด", "available": false }
    ]
  }]
}
```

**กติกาการคำนวณ:**
- slot ที่ไม่เปิดในวันนั้นจะ**ไม่ปรากฏเลย** (กรองที่ `getAvailableSlotIdsOnDate`) — ไม่ใช่ตอบ `available: false`
- **slot ที่เลยเวลาสิ้นสุดไปแล้ววันนี้ก็ไม่ปรากฏ** (กรองที่ `hasSlotEnded`) เพราะจองไม่ได้แล้ว — ดู `PatientBookingController.php`
- `staff[]` = หมอที่ `status=ACTIVE` + ผูกใน `service_staff` ของบริการนี้ + **ไม่ลาในวันนั้น** (`StaffLeave::getSelectableStaff`, app/Models/StaffLeave.php)
- `available` = หมอคนนั้นยังไม่มีคิวในรอบนั้น (โควตา 1 คิว/รอบ/คน)
- `max_capacity` = จำนวนหมอที่ผูกไว้ (ไม่ใช่ `max_per_slot`)
- ถ้าศูนย์ไม่ ACTIVE → คืน `slots: []` แต่ `success: true` (PatientBookingController.php)

> ⚠️ ถ้า response **ไม่มี key `staff` เลย** แปลว่า `allow_staff_selection=false` — ไม่ใช่เรื่อง capacity_type

### ขั้นจอง (ต้องมี Patient token)

**4. `POST /v1/patient/login`** หรือ **`POST /v1/patient/register`** → ได้ token (A2)

**5. `POST /v1/patient/book`** — `Authorization: Bearer <token>`
```json
{
  "service_id": 1,
  "time_slot_id": 1,
  "appointment_date": "2026-09-25",
  "staff_id": 1,
  "patient_right_id": 1
}
```

| field | rule |
|---|---|
| `service_id` | required, `exists:services,id` |
| `time_slot_id` | required — **ต้องเป็น slot ที่เปิดในวันที่เลือกจริง** (เช็คจาก `service_time_slot` + `days_mask`) |
| `appointment_date` | required, `Y-m-d`, **≥ วันนี้** |
| `staff_id` | nullable ใน Request — **แต่บังคับตอน business** ต้องมีเมื่อเปิดเลือกหมอ |
| `patient_right_id` | optional — ส่งมาแล้วหาไม่เจอ **ปฏิเสธ 422** ไม่เงียบ เพราะข้อมูลที่ส่งมาไม่ถูกต้อง |

`BookAppointmentRequest.php`

**ตรวจสอบตามลำดับ** (`BookingService::createAppointment`, app/Services/BookingService.php) — ทั้งหมดอยู่ใน transaction เดียว ใช้ `lockForUpdate()` กัน race:

| # | ตรวจอะไร | ผลลัพธ์เมื่อไม่ผ่าน |
|---|---|---|
| 1 | `service.is_active` | ปิดชั่วคราว |
| 2 | ศูนย์ `isBookable()` = ACTIVE | ศูนย์ปิดรับการจองใหม่ |
| 3 | `time_slot` เปิดในวันนั้น | บริการไม่เปิดในวันที่เลือก |
| 4 | ช่วงเวลาสิ้นสุดยังไม่ผ่านมาแล้ว (**เฉพาะการจองเอง**) | ช่วงเวลานี้เลยกำหนดแล้ว |
| 5 | `isServiceAvailable` (บุคลากรไม่ลาครบทั้งหมด) | บริการปิดชั่วคราว |
| 6 | ไม่จอง service เดิมซ้ำวันเดิม (CONFIRMED) | ท่านมีรายการซ้ำแล้ว |
| 7 | **เปิดให้เลือกบุคลากร** → `staff_id` ต้องมี | กรุณาเลือกบุคลากร |
| 8 | บุคลากร ACTIVE + ผูกกับบริการนี้ + ไม่ลา | เลือกคนนี้ไม่ได้ |
| 9 | บุคลากรยังไม่เต็มรอบ (1 คิว/รอบ/คน) | เต็มคิวในรอบนี้แล้ว |
| — | *กรณีไม่เปิดให้เลือกบุคลากร* → โควตาว่าง (`lockForUpdate`) | คิวในรอบเวลานี้เต็มแล้ว |
| 10 | สิทธิการรักษาที่แนบยังไม่หมดอายุ | สิทธิหมดอายุแล้ว |
| 11 | ออกเลขคิว = `max+1` ของ (ศูนย์, วันนั้น) + ดัชนีกันซ้ำ | ชนกัน → 409 ให้ลองใหม่ |

> ข้อ 4 **ไม่ใช้กับนัดหมายเดินเข้ารับบริการที่หน้าเคาน์เตอร์** เพราะผู้รับบริการมาถึงหน้าเคาน์เตอร์แล้วต้องได้รับบริการ
> ข้อ 8 ตรวจ 2 ชั้น: `isStaffAvailableForService()` (StaffLeave.php) และ `isStaffSlotAvailable()` (CapacityService.php)
> ข้อ 10 ใช้กับทุกทางเข้า ทั้งจองเองและหน้าเคาน์เตอร์
> ข้อความ error ของ `POST /v1/patient/book` **ถูกกลืน** → ดูหัวข้อ 0.6

**Response 201:**
```json
{
  "success": true, "message": "จองคิวสำเร็จ",
  "data": {
    "appointment_id": 1, "queue_no": "Q-001",
    "health_center": "...", "service_name": "...",
    "appointment_date": "2026-09-25", "time_slot": "09:00-09:30",
    "staff": { "id": 1, "name": "...", "position": "หมอนวด" }
  }
}
```
`PatientBookingController.php`

### ตรวจสอบ / ยกเลิก (optional)

**6. `GET /v1/patient/appointments/{id}`** — ดูรายละเอียด (เฉพาะของตัวเอง)
**`PATCH /v1/patient/appointments/{id}/cancel`** — ยกเลิก (A5)

---

## A4. จองคิว — บริการที่ไม่เปิดเลือกหมอ (`allow_staff_selection = false`)

**ต่างจาก A3 อยู่ 2 จุดเท่านั้น:**

| จุด | `allow_staff_selection = true` | `allow_staff_selection = false` |
|---|---|---|
| ขั้น 3 — `available-slots` | มี `staff[]` ให้เลือกต่อ slot | **ไม่มี key `staff` เลย** |
| ขั้น 5 — `book` | ต้องส่ง `staff_id` | **ไม่ส่ง** `staff_id` |
| การนับโควตา | ต่อหมอ (1 คิว/รอบ/คน) | ตาม `capacity_type` (รวม/ต่อรอบ/ต่อวัน) |
| response `staff` | คืนหมอที่เลือก | คืน `null` → `"เจ้าหน้าที่": "ระบบจัดสรร"` ใน Discord |

**ขั้นตอนเดียวกันเป๊ะ** — A1.1 → A1.5 → ขั้น 3 → login/register → `POST /book` → A5

ผลลัพธ์ขั้น 3 แบบไม่เลือกหมอ:
```json
{
  "slots": [{
    "time_slot_id": 1, "time_label": "09:00-09:30",
    "max_capacity": 1, "available_count": 1, "is_full": false
  }]
}
```

**`PER_DAY` ต่างอีก:** ค่า `available_count` คือ **ทั้งวันรวมกัน** ไม่ใช่รายรอบ — ถ้าเต็ม 1 คิว รอบอื่นของวันเดียวกันจะเป็น `available_count: 0` ด้วย (`CapacityService.php`)

---

## A5. ประวัติการจอง + ยกเลิก

### A5.1 ดูประวัติ

**`GET /v1/patient/appointments?per_page=20`**
- คืนเฉพาะของตัวเอง เรียงวันที่ล่าสุดขึ้นมา
- `per_page` ถูก clamp ระหว่าง 1-100 (ส่งเกินจะถูกตัด ไม่ error)
- envelope มี `meta: { current_page, last_page, per_page, total }`
`PatientHistoryController.php`

### A5.2 ดูรายละเอียด 1 ใบ

**`GET /v1/patient/appointments/{id}`**
คืน key `ticket` (ไม่ใช่ `data`) — ของผู้รับบริการอื่น → 404
`PatientHistoryController.php`

### A5.3 ยกเลิก

**`PATCH /v1/patient/appointments/{id}/cancel`**
```json
{ "reason": "ติดธุระ" }   // optional, max 500
```

**ยกเลิกได้เฉพาะเมื่อ 2 เงื่อนไขพร้อมกัน** (`PatientHistoryController.php`):
1. `status = CONFIRMED`
2. เป็นวันอนาคต **หรือ** เป็นวันนี้แต่ยังไม่เลยเวลาสิ้นสุดของรอบ (`timeSlot.end_time > เวลาปัจจุบัน`)

ไม่ผ่านเงื่อนไขใดข้อ → **404** (ไม่ใช่ 422 — เพราะ query ถูกจำกัดเงื่อนไขไว้ก่อน `findOrFail`)

- `reason` ไม่ส่ง → default `"ผู้รับบริการขอยกเลิกเองผ่านระบบ"`
- ถ้าศูนย์เป็น CLOSING → dispatch `CloseHealthCenterJob` ลองปิดศูนย์อัตโนมัติ (`:71-73`)
- **ยกเลิกแล้วย้อนกลับไม่ได้** — `VALID_TRANSITIONS` ไม่มีทางออกจาก CANCELLED

---

# Part B — ฝั่งเจ้าหน้าที่

> ทุก endpoint ใน Part B ต้องมี Staff token และผ่าน role gate `STAFF, HEALTH_CENTER_ADMIN, SUPER_ADMIN`
> **ระลึกกฎ `health_center_id` ในหัวข้อ 0.3 ทุกครั้ง**

## B1. Login / โปรไฟล์ตัวเอง / Logout

**1. `POST /v1/staff/login`** (throttle `throttle.staff`) — ไม่ต้องมี token
```json
{ "username": "...", "password": "..." }
```
**200:**
```json
{
  "success": true, "message": "เข้าสู่ระบบสำเร็จ", "token": "...",
  "user": { "id": 1, "name": "...", "username": "...", "role": "HEALTH_CENTER_ADMIN",
            "health_center": { "id": 1, "name": "...", "code": "..." } }
}
```
**ที่ควรรู้:**
- ชื่อผู้ใช้/รหัสผิด → `ValidationException` 422 (ไม่ใช่ 401)
- `health_center` เป็น `null` สำหรับ SUPER_ADMIN และไม่ถูกตรวจ
- ศูนย์เป็น `INACTIVE` → **403** `"ศูนย์สุขภาพต้นสังกัดถูกระงับการใช้งานชั่วคราว"`
- ศูนย์เป็น `CLOSING` → **ยัง login ได้** (StaffAuthController.php)

**2. `GET /v1/staff/me`** — ข้อมูลตัวเอง + `roles[]` + `health_center` เต็มรูปแบบ
> สำหรับ STAFF/HC_ADMIN ค่านี้แทน `GET /v1/staff/health-centers` ได้ (ซึ่งคืนแค่ศูนย์ตัวเอง 1 แถว)

**3. `PATCH /v1/staff/me`** — แก้ชื่อ / เปลี่ยนรหัสผ่าน
```json
{ "name": "ชื่อใหม่" }
```
```json
{ "current_password": "เดิม", "password": "ใหม่อย่างน้อย 6 ตัว", "password_confirmation": "ใหม่อย่างน้อย 6 ตัว" }
```
เปลี่ยนรหัสผ่าน **ต้องส่ง `current_password` ที่ถูกต้อง** เสมอ (UpdateProfileRequest.php)

**4. `POST /v1/staff/logout`** — ลบ token ปัจจุบันเท่านั้น (token อื่นยังใช้ได้)

---

## B2. Dashboard

### B2.1 สรุปเพียงอย่างเดียว

**`GET /v1/staff/dashboard/summary?date=YYYY-MM-DD`**

- `date` optional, default = วันนี้
- HC_ADMIN/STAFF → ศูนย์ตัวเอง; SUPER_ADMIN ไม่ส่ง `health_center_id` = **รวมทุกศูนย์**
```json
{
  "success": true, "date": "2026-09-24",
  "summary": { "total": 12, "waiting": 5, "completed": 6, "cancelled": 1, "no_show": 0 }
}
```
นับจาก `appointments` สถานะ `CONFIRMED/COMPLETED/CANCELLED/NO_SHOW` ของวันนั้น — query เดียวจบ
**STAFF → นับเฉพาะบริการที่ถูกมอบหมาย** (ดู 0.3a) ตัวเลขจึงตรงกับรายการคิวที่เขาเห็นเสมอ

`StaffDashboardController::getSummary()`

### B2.2 หน้าจอหนึ่งจอ — สามชิ้นในคำขอเดียว (แนะนำ)

**`GET /v1/staff/dashboard/bundle?date=YYYY-MM-DD`**

หน้าจอโต๊ะเคาน์เตอร์ต้องใช้สามอย่าง: ตัวเลขสรุป รายการคิว และรายชื่อบุคลากร
ถ้าเรียกแยกทีละ endpoint แต่ละคำขออาจได้วันที่ต่างกัน ทำให้ตัวเลขสรุปไม่ตรงกับรายการที่เห็น

```json
{
  "success": true, "date": "2026-09-24",
  "summary": { "total": 3, "waiting": 2, "completed": 1, "cancelled": 0, "no_show": 0 },
  "appointments": [ /* เหมือน GET /staff/appointments */ ],
  "roster": [ /* เหมือน GET /staff/roster?date= */ ]
}
```

**ขอบเขตของแต่ละชิ้นไม่เปลี่ยน การรวมไม่ใช่การเปิดสิทธิ์:**

| ชิ้น | STAFF | ผู้ดูแล |
|---|---|---|
| `summary` | เฉพาะบริการที่มอบหมาย | ทั้งศูนย์ |
| `appointments` | เฉพาะบริการที่มอบหมาย | ทั้งศูนย์ |
| `roster` | **ทั้งศูนย์** | ทั้งศูนย์ |

- `summary.total` **เท่ากับ** จำนวนใน `appointments` เสมอ — มี test ล็อกไว้
- `date` เดียวกันกับทั้งสามชิ้น
- ข้อมูลผู้รับบริการใน `appointments` ยังถูกปิดบังเหมือนเดิม (ไม่มี `cid` หรือที่อยู่)
- `roster` คืน `effective_status` (รวมวันลา) · ไม่ส่ง `date` จะไม่มีฟิลด์นี้
- **endpoint เดิมทั้งสามยังทำงานเหมือนเดิม** — ย้ายมาใช้ bundle ทีละหน้าได้

`StaffDashboardController::bundle()`

### B2.3 โควตาการเรียก API (Rate Limit)

**ทุก endpoint ใน `routes/api.php` ถูกจำกัดอัตราโดยอัตโนมัติ** ผ่าน middleware group `api`

| ผู้เรียก | ต่อนาที |
|---|---|
| เจ้าหน้าที่ปฏิบัติงาน · ผู้ดูแลศูนย์ · ผู้ดูแลระบบ | **240** |
| ผู้รับบริการ · ยังไม่ยืนยันตัวตน | **60** |

- **นับแยกตามบัญชี ไม่ใช่ตาม IP** — ผู้เรียกคนเดียวที่ยิงหนักไม่กระทบคนอื่น
- **เก็บตัวนับใน Redis** (`CACHE_STORE=redis`) จึงนับรวมทุก instance
- เป็น **fixed window** — ขอบเขตแรงสุดคือโควตาเต็มในช่วงเวลาใดช่วงหนึ่ง แล้วเป็นศูนย์ทันทีเมื่อข้ามเส้น
  ถ้าหน้าจอยิงแน่นตอนเปิดจนชนเพดาน ผู้ใช้จะเจอ 429 ทันทีและต้องรอข้ามเส้น

**ลิมิตเฉพาะทางที่เข้มกว่านี้** (ทำงานทับโควตารวม):

| ลิมิต | ค่า | ใช้กับ |
|---|---|---|
| `throttle:unmask` | 10/นาที | `POST /staff/appointments/{id}/unmask` (PDPA) |
| `throttle:patient-login` | 10/นาที **ต่อเบอร์โทร** | `POST /patient/login` |
| `throttle:patient-register` | 5/นาที ต่อ IP | `POST /patient/register` |
| `throttle.staff` | 5 ครั้ง / 30 นาที | `POST /staff/login` (ดู B1) |

> `patient-login` นับต่อเบอร์โทร ไม่ใช่ต่อ IP เพราะต้องกันการเดารหัสผ่านการลองเบอร์ทีละเบอร์
> ถ้าใช้ IP ผู้โจมตีเปลี่ยน IP แล้วเดาได้ไม่จำกัด

`AppServiceProvider::boot()`

---

## B3. จัดการบริการ

### B3.1 ดูรายการบริการ

**`GET /v1/staff/services`**

| query | ผล |
|---|---|
| — | HC_ADMIN/STAFF = ศูนย์ตัวเอง; SUPER_ADMIN = ทุกศูนย์ |
| `?health_center_id=` | SUPER_ADMIN กรองศูนย์ |
| `?date=YYYY-MM-DD` | **ซ่อนบริการที่หมอทั้งหมดลาในวันนั้น** (เฉพาะเมื่อมี center scope ชัดเจน) |

`StaffServiceController.php` — การกรองใช้ `StaffLeave::isServiceAvailable()` ตัวเดียวกับ booking

### B3.2 เพิ่มบริการ

**`POST /v1/staff/services`**
```json
{
  "category_id": 1,
  "name": "นวดแผนไทย",
  "description": "...",
  "capacity_type": "PER_SLOT",
  "max_per_slot": 2,
  "max_per_day": 10,
  "time_slots": [
    { "time_slot_id": 1, "days": [1,2,3,4,5] },
    { "time_slot_id": 2, "days": [1,2,3,4,5] }
  ]
}
```

| field | rule |
|---|---|
| `category_id` | required, `exists:categories,id` |
| `name` | required, max 255 |
| `capacity_type` | required, `PER_MASSEUSE/PER_SLOT/PER_DAY` |
| `max_per_slot` | required **เฉพาะ** PER_SLOT, integer ≥ 1 |
| `max_per_day` | required **เฉพาะ** PER_DAY, integer ≥ 1 |
| `time_slots` | required, array ≥ 1 |
| `time_slots[].time_slot_id` | required, `exists:time_slots,id` |
| `time_slots[].days` | required, array ≥ 1, ค่า 1-7 (1=จันทร์) |
| `allow_staff_selection` | **ส่งได้แค่ `false`/`0`** — ส่ง `true` → 422 |

**ผลลัพธ์:** สร้าง `services` (บังคับ `allow_staff_selection=false`, `is_active=true`) + `service_capacity` (default 1/10) + `service_time_slot` ใน transaction เดียว
`CreateServiceRequest.php` · `StaffServiceManagementService.php`

### B3.3 แก้ไขบริการ

**`PUT /v1/staff/services/{id}`** — ส่งเฉพาะ field ที่ต้องการแก้

| field | rule |
|---|---|
| `allow_staff_selection: true` | ได้เฉพาะ `PER_MASSEUSE` **และ** มีหมอใน `service_staff` ≥ 1 คน — ไม่งั้น 422 |
| `allow_staff_selection: false` | 422 ถ้ายังมีคิว CONFIRMED อนาคตของบริการนี้ |
| `max_per_slot` / `max_per_day` | integer ≥ 1, `updateOrCreate` ค่าเดิมไว้ถ้าไม่ส่ง |
| `time_slots` | ถ้าส่ง = แทนที่ชุดเดิมทั้งหมด (ดู B3.5) |

`UpdateServiceRequest.php` · `StaffServiceManagementService.php`

### B3.4 เปิด/ปิดบริการ + ลบ

**`PATCH /v1/staff/services/{id}/toggle-status`** — สลับ `is_active` (ACTIVE/ปิดชั่วคราว)
**`DELETE /v1/staff/services/{id}`** — **HC_ADMIN/SUPER_ADMIN เท่านั้น** (STAFF → 403)
→ 422 ถ้ามีคิว CONFIRMED ตั้งแต่วันนี้ขึ้นไป: `"ไม่สามารถลบบริการได้ เนื่องจากมีคิวล่วงหน้าอยู่ N คิว"` (soft delete)

### B3.5 ตั้งวัน/เวลาเปิดให้บริการ

**`PUT /v1/staff/services/{id}/time-slot-days`** — HC_ADMIN / SUPER_ADMIN
```json
{
  "health_center_id": 1,          // required เฉพาะ SUPER_ADMIN
  "time_slots": [
    { "time_slot_id": 1, "days": [1,2,3,4,5] },
    { "time_slot_id": 2, "days": [1,2,3,4,5,6] }
  ]
}
```

**กติกาสำคัญ — full-set replacement:**
- ลบ `service_time_slot` ทั้งหมดของบริการ แล้ว insert ใหม่ทั้งชุด (ต้องส่งครบทุก slot ที่ต้องการ)
- ต้องมีอย่างน้อย 1 คู่ (slot, day) — ว่างเปล่า → 422
- **`days_mask` bitmask:** bit 0 = จันทร์ … bit 6 = อาทิตย์ (`1 << (day-1)`) — ค่า `127` = เปิดครบ 7 วัน
- **กันไม่ให้ "ปิดวัน" ที่มีคิว CONFIRMED ในอนาคต** → 422 `"ไม่สามารถลบวันให้บริการได้ เนื่องจากมีคิวที่ยืนยันแล้วในอนาคต กรุณาจัดการคิวก่อน"`
- ทำใน transaction + `lockForUpdate` + เขียน `AuditLog` (action `SERVICE_TIME_SLOT_DAYS_SYNC`)
- **ช่วงเวลาที่เลือกต้องเป็นของศูนย์เดียวกับบริการ** → ต่างศูนย์ = 422 `"ช่วงเวลาที่เลือกไม่ได้อยู่ในศูนย์สุขภาพเดียวกับบริการ"` บังคับไว้ 2 ชั้น คือตรวจในโค้ด และ foreign key แบบ composite ในฐานข้อมูล (ตารางเชื่อมมีคอลัมน์ศูนย์ของตัวเอง ต้องตรงกับทั้งบริการและช่วงเวลา) — เดิมไม่เคยตรวจข้อนี้ จึงผูกช่วงเวลาข้ามศูนย์ได้
`OperatingDayService.php` · `StaffServiceController.php`

---

## B4. เปิดเลือกหมอ (สำคัญ — ต้องทำตามลำดับ)

> ⚠️ **ลำดับห้ามสลับ** — ถ้าผูกหมอก่อนแล้วค่อยเปิด flag จะ 422

### ขั้นที่ 1 — ดูหมอที่จะผูก

**`GET /v1/staff/roster`** (B8.1) → ได้ `staff.id` พร้อม `category_id`
> หมอที่ผูกได้ต้องเป็น **หมวดหมู่เดียวกับบริการ** (เช่น บริการหมวด "แพทย์แผนไทย" ต้องผูกหมอหมวดนั้น)

### ขั้นที่ 2 — ผูกหมอกับบริการ

**`PUT /v1/staff/services/{id}/staff`** — HC_ADMIN / SUPER_ADMIN (STAFF → 403)
```json
{ "health_center_id": 1, "staff_ids": [1, 2] }   // health_center_id required เฉพาะ SUPER_ADMIN
```

| กติกา | ผลลัพธ์ |
|---|---|
| `staff_ids` ต้องมี key เสมอ | ไม่ส่ง → 422; ส่ง `[]` = ยกเลิกทั้งหมด |
| หมอต้องอยู่ศูนย์เดียวกัน | 422 `"เจ้าหน้าที่ X ไม่สังกัด ศูนย์สุขภาพนี้"` |
| หมอต้องอยู่หมวดหมู่เดียวกับบริการ | 422 `"หมอ X ไม่อยู่ในหมวดหมู่เดียวกับบริการนี้"` |
| **การถอดหมอออก** ที่มีคิว CONFIRMED อนาคต | 422 `"ไม่สามารถแกะเจ้าหน้าที่ออกจากบริการได้..."` |

เขียน `AuditLog` action `SERVICE_SELECTABLE_STAFF_SYNC` (before/after)
`StaffServiceController.php`

### ขั้นที่ 3 — เปิด flag

**`PUT /v1/staff/services/{id}`**
```json
{ "health_center_id": 1, "allow_staff_selection": true }
```

| เงื่อนไข | ผลลัพธ์ |
|---|---|
| `capacity_type ≠ PER_MASSEUSE` | 422 `"บริการที่เปิดให้เลือกเจ้าหน้าที่ต้องมีประเภทความจุเป็น PER_MASSEUSE"` |
| ยังไม่มีหมอใน `service_staff` | 422 `"กรุณาผูกเจ้าหน้าที่อย่างน้อย 1 คนก่อนเปิดให้เลือกเจ้าหน้าที่"` |
| มีคิว CONFIRMED อนาคต (กรณีปิด flag) | 422 |

**ตั้งแต่นี้** → `GET /v1/patient/available-slots` จะเริ่มคืน `staff[]` ให้ผู้รับบริการเลือก

### ขั้นที่ 4 (ทางเลือก) — ปิด flag กลับ

**`PUT /v1/staff/services/{id}`** ส่ง `allow_staff_selection: false`
→ 422 ถ้ายังมีคิว CONFIRMED อนาคต — **ต้องจัดการคิวให้หมดก่อน** (เช่น reassign หมอ หรือเปลี่ยนสถานะคิว)

> `allow_staff_selection` เป็น **column ของ service แต่ละแถว** — ไม่มี config กลาง ไม่มีการซิงก์ข้ามศูนย์ แต่ละศูนย์ตั้งอิสระ

---

## B5. มอบหมายผู้รับผิดชอบบริการ

**`PUT /v1/staff/services/{id}/assignees`** — HC_ADMIN / SUPER_ADMIN (STAFF → 403)
```json
{ "health_center_id": 1, "user_ids": [10, 11] }   // [] = ยกเลิกทั้งหมด
```

**นิยาม:** `user_ids` คือ **บัญชี User ที่มี role STAFF** ไม่ใช่ staff profile — ใช้คุมว่า STAFF คนนั้นเห็น/จัดการคิวของบริการนี้ได้

| กติกา | ผลลัพธ์ |
|---|---|
| ผู้ใช้ต้องอยู่ศูนย์เดียวกัน | 422 `"ผู้ใช้ X ไม่สังกัด ศูนย์สุขภาพนี้"` |
| ผู้ใช้ต้องมี role STAFF | 422 `"ผู้ใช้ X ไม่มีบทบาท STAFF"` |

**ผลข้างเคียง — นี่คือขอบเขตที่ใช้คุมทุกทางที่ดูหรือแตะคิว (ดู 0.3a):**

STAFF ที่ไม่ได้ถูก assign บริการนั้น → **403** ในทุกเส้นทางต่อไปนี้

| endpoint | ผล |
|---|---|
| `GET /staff/appointments` | เห็นเฉพาะคิวบริการที่ assign · filter `service_id` ที่ไม่ได้ assign → 403 |
| `GET /staff/dashboard/summary` | นับเฉพาะบริการที่ assign |
| `GET /staff/dashboard/bundle` | เหมือนข้างบน (แต่ `roster` ยังทั้งศูนย์) |
| `PATCH /staff/appointments/{id}/status` | 403 — เว้นแต่เขาเป็นผู้ถูกระบุในคิวนั้น |
| `PATCH /staff/appointments/{id}/reassign-staff` | 403 — เว้นแต่เป็นผู้ถูกระบุในคิวนั้น |
| `POST /staff/appointments/{id}/unmask` | 403 + **ไม่เขียน audit log** |
| `PATCH /staff/patients/{id}/contact` | 403 ถ้าผู้รับบริการไม่เคยมีคิวในบริการที่ assign |
| `POST /staff/appointments/walk-in` | 403 ทุกกรณี — เป็นหน้าที่ผู้ดูแลศูนย์อยู่แล้ว ไม่เกี่ยวกับการ assign |

`StaffServiceController::syncAssignees()` · `ResolvesHealthCenterScope::denyIfOutOfServiceScope()`

> **บริการที่ไม่ผูกชื่อบุคลากร** (นับตามช่วงเวลา/ทั้งวัน) คิวจะไม่มีผู้ถูกระบุ
> ทางที่สองจึงใช้ไม่ได้ คิวของบริการนั้นจะมีแต่ผู้ดูแลศูนย์ที่ปิดได้
> ถ้าในศูนย์ไม่มีใครถูกผูกบริการเลย คิวเหล่านั้นจะไม่มีใครทำงานต่อได้ — ต้องตรวจข้อมูลจริงก่อนใช้งาน

---

## B6. จัดการช่วงเวลา (Time Slots)

> ช่วงเวลาเป็นของ **ศูนย์** เสมอ — คอลัมน์ศูนย์เป็น NOT NULL และ unique ต่อ `(health_center_id, start_time, end_time)`
> การดูรายชื่อช่วงเวลาเป็น **ศูนย์เดียวต่อหนึ่งคำขอ** ไม่มีการดูข้ามศูนย์ ผู้ดูแลระบบเลือกศูนย์ได้จาก `GET /v1/staff/health-centers`

| Method | Endpoint | Role | หมายเหตุ |
|---|---|---|---|
| `GET` | `/v1/staff/admin/time-slots` | HC_ADMIN, SUPER_ADMIN | HC_ADMIN ไม่ต้องส่ง (ได้ศูนย์ตนเอง); SUPER_ADMIN **ต้องส่ง `health_center_id`** (ไม่ส่ง = 422, ไม่มีศูนย์นั้น = 404) |
| `PATCH` | `/v1/staff/admin/time-slots/{id}/toggle-status` | HC_ADMIN, SUPER_ADMIN | ข้ามศูนย์ไม่ได้ (404) |
| `POST` | `/v1/staff/admin/time-slots` | **SUPER_ADMIN เท่านั้น** | บังคับ `health_center_id` |
| `PUT` | `/v1/staff/admin/time-slots/{id}` | **SUPER_ADMIN เท่านั้น** | บังคับ `health_center_id`; slot ต้องเป็นของศูนย์นั้น |
| `DELETE` | `/v1/staff/admin/time-slots/{id}` | **SUPER_ADMIN เท่านั้น** | **ไม่รองรับ** — 422 ทุกกรณี |

**สร้าง:**
```json
{ "health_center_id": 1, "start_time": "09:00", "end_time": "09:30", "is_active": true }
```
- เวลา format `HH:MM`, `end_time` ต้องมากกว่า `start_time`
- **ช่วงเวลาที่ทับกับช่วงเดิมของศูนย์เดียวกัน → 422** `"มีช่วงเวลาที่ทับซ้อนกันอยู่แล้วในศูนย์นี้"` — ตรวจการทับจริง (ไม่ใช่เทียบเวลาให้ตรงกันทุกตัว) ช่วงที่ต่อกันพอดี เช่น 09:00-09:30 กับ 09:30-10:00 ไม่ถือว่าทับ
- ทับกับช่วงเวลาของศูนย์อื่น → สร้างได้ (แต่ไปผูกกับบริการข้ามศูนย์ไม่ได้ ดู B4/B7.2)
- `label` และ `sort_order` ถูกสร้างอัตโนมัติ; `is_active` ส่งมาได้ถ้าไม่ส่งค่าเริ่มต้นคือเปิดใช้งาน

**การตอบกลับ** — ทุกช่วงเวลาที่คืนมาระบุ `health_center` ของตัวเอง (เฉพาะ endpoint นี้ ที่อื่นที่ฝังช่วงเวลาไว้ เช่น นัดหมายของผู้รับบริการ จะไม่มีข้อมูลนี้)

**ลบ** → ไม่รองรับ 422 `"ไม่สามารถลบช่วงเวลาได้ กรุณาเปลี่ยนสถานะเป็นปิดใช้งานแทน"`
เหตุผลตาม ADR-0001: ช่วงเวลาถูกนัดหมายและถูกกำหนดวันให้บริการอ้างอิงอยู่ การลบจะทำให้ประวัตินัดหมายหายหรือวันให้บริการหายเงียบ ๆ

**ปิดใช้งานแทน** → เมื่อปิด ช่วงเวลาหายจากผู้รับบริการทันทีทุกบริการของศูนย์ แต่ **วันให้บริการที่ตั้งไว้ยังอยู่ครบ** และกลับมาใช้ทันทีเมื่อเปิดคืน

> ✅ **ข้อยกเว้นที่เคยเป็น bug แก้แล้ว:** controller นี้เคยไม่ใช้ `ResolvesHealthCenterScope` — SUPER_ADMIN ไม่ส่ง `health_center_id` ได้สร้างช่วงเวลาที่ไม่ผูกศูนย่อจริง และแก้/ลบข้ามศูนย์ได้ ตอนนี้ทุก endpoint ใช้ตัวแก้ scope กลางเหมือนที่เหลือ และฐานข้อมูลบังคับศูนย์ไม่ให้ว่างด้วย

---

## B7. โต๊ะเคาน์เตอร์คิว

### B7.1 ดูคิวของวัน

**`GET /v1/staff/appointments`**

| query | ผล |
|---|---|
| `date` | default วันนี้ |
| `service_id` | กรองรายบริการ |
| `status` | `CONFIRMED/COMPLETED/CANCELLED/NO_SHOW` |
| `health_center_id` | SUPER_ADMIN เท่านั้น |

**STAFF (ไม่ใช่ admin)** → เห็น**เฉพาะ**บริการที่ถูก assign ให้ตัวเอง และถ้า query `service_id` ที่ไม่ได้ assign → **403**
`StaffAppointmentController::index()` · `StaffAppointmentService::getDailyAppointments()`

> ข้อมูลผู้รับบริการใน response ถูก **masked** เสมอ — `patient.masked_name`, `patient.masked_phone` (AppointmentResource.php)

### B7.2 เปลี่ยนสถานะคิว

**`PATCH /v1/staff/appointments/{id}/status`**
```json
{ "status": "COMPLETED" }
```
```json
{ "status": "CANCELLED", "cancellation_reason": "ผู้รับบริการมาไม่ได้" }
```

- `status` ∈ `COMPLETED / CANCELLED / NO_SHOW` — **`CONFIRMED` ส่งไม่ได้**
- `cancellation_reason` **required** เมื่อ CANCELLED, max 500
- เปลี่ยนจากสถานะอื่นที่ไม่ใช่ CONFIRMED → 422 (แผนที่: `CONFIRMED → [COMPLETED, CANCELLED, NO_SHOW]` เท่านั้น)
- `NO_SHOW` ที่**ระบบเปลี่ยนเอง** แก้กลับเป็น COMPLETED ได้ ส่วนที่คนกดเองถือเป็นปลายทางถาวร
- ตั้งเป็น terminal + ศูนย์เป็น CLOSING → dispatch auto-close
- **ผู้เรียกได้**: `HEALTH_CENTER_ADMIN` (ศูนย์ตน) · `SUPER_ADMIN` (ต้องระบุ `health_center_id`) · `STAFF` (เฉพาะเงื่อนไขขอบเขตบริการ)
- **ขอบเขตบริการ (403)** — STAFF ต้องเข้าเงื่อนไขอย่างน้อยหนึ่งทางใน 0.3a
  1. ได้รับมอบหมายบริการนั้น **หรือ** 2. เป็นผู้ถูกระบุในคิวนั้น
- **การแก้ "ไม่มาตามนัด" ที่ระบบเปลี่ยนเอง ใช้เกณฑ์เดียวกัน** — ไม่มีช่องทางพิเศษ
- **ตรวจขอบเขตก่อนกฎการเปลี่ยนสถานะ** ดังนั้นผู้เรียกที่ไม่มีสิทธิ์จะได้ 403 แม้ส่งสถานะผิดวิธี
  ผลข้างเคียง: ไม่เปิดเผยว่าสถานะไหนถูก — ถือว่าถูกต้องเพราะเขายังไม่มีสิทธิ์แตะคิวนี้
`UpdateAppointmentStatusRequest.php` · `StaffAppointmentService.php`

### B7.3 นัดหมายเดินเข้ารับบริการที่หน้าเคาน์เตอร์ (Walk-in)

**`POST /v1/staff/appointments/walk-in`**

ผู้ดูแลศูนย์เป็นผู้เรียก — เขาถือตารางเวลาและบุคลากรของศูนย์ทั้งหมด
จึงออกนัดหมายได้ทุกบริการโดยไม่ต้องได้รับมอบหมาย
**เจ้าหน้าที่ปฏิบัติงาน (`STAFF`) เรียกไม่ได้ → 403** เพราะหน้าที่นี้เป็นของผู้ดูแลศูนย์ ไม่ใช่งานที่ทำตามที่ได้รับมอบหมาย
route อยู่ใน group `role:SUPER_ADMIN,HEALTH_CENTER_ADMIN` จึงตัดที่ระดับ route ไม่ใช่ระดับขอบเขตบริการ
ผู้ดูแลจึงออกคิวได้ทุกบริการในศูนย์โดยไม่ต้องได้รับมอบหมาย

หน้าเคาน์เตอร์เก็บข้อมูลผู้รับบริการ **ชุดเดียวกับที่การลงทะเบียนใช้**
เพื่อให้ผู้รับบริการเข้าสู่ระบบด้วย `phone_number` + `birth_date`
แล้วจองเองในครั้งถัดไปได้ โดยไม่ต้องลงทะเบียนซ้ำ

**ก่อนกรอก: ค้นด้วย `GET /v1/staff/admin/patients?cid=...`**
เพื่อเติมฟอร์มจากข้อมูลเดิม — คืนโปรไฟล์พร้อม `patient_rights`
การค้นนี้ไม่จำกัดศูนย์ เพราะการค้นด้วยเลขบัตรประชาชนคือการยืนยันตัวตน
ผู้รับบริการที่ลงทะเบียนเองที่ศูนย์อื่นต้องหาเจอที่หน้าเคาน์เตอร์นี้ได้
(การค้นแล้วพบถูกบันทึกลง `audit_logs` เป็น `LOOKUP_PATIENT_BY_CID` — ค้นแล้วไม่พบไม่บันทึก)

```json
{
  "cid": "5100000000011",
  "first_name": "สมชาย", "last_name": "ใจดี",
  "phone_number": "0899876543", "birth_date": "1990-01-01",
  "gender": "MALE",
  "subdistrict": "บ้านโคก", "district": "เมือง", "province": "ลำพูน",
  "full_address": "123 หมู่ 4 ตำบลบ้านโคก",
  "right_type_id": 1,
  "main_hospital_name": "โรงพยาบาลลำพูน",
  "health_center_id": 1,
  "service_id": 1, "time_slot_id": 1,
  "staff_id": 1,
  "patient_right_id": 1
}
```

| field | rule |
|---|---|
| `cid` | required, 13 หลัก (ตัวเลขล้วน) — ใช้เป็น key หา/สร้างผู้รับบริการ |
| `first_name` / `last_name` | required |
| `phone_number` | required, 10 หลัก — คู่กับวันเกิดใช้ยืนยันตัวตนตอนเข้าสู่ระบบ |
| `birth_date` | required — คู่กับเบอร์โทรศัพท์ |
| `gender` | optional — `MALE` / `FEMALE` / `OTHER` |
| `subdistrict` / `district` / `province` / `full_address` | optional — ที่อยู่ |
| `right_type_id` | **optional** — หน้าเคาน์เตอร์ไม่ต้องถามทุกครั้ง ต้องเป็นประเภทที่ `status = ACTIVE`; ไม่ส่ง = นัดหมายไม่มีสิทธิผูกไว้ |
| `main_hospital_name` | optional — สถานพยาบาลหลักตามสิทธิ |
| `health_center_id` | **required เฉพาะผู้ดูแลระบบ** — ไม่ส่ง = 422 `"ผู้ดูแลระบบต้องระบุ health_center_id"`, ส่งศูนย์ที่ไม่มี = 404; ผู้ดูแลศูนย์ไม่ต้องส่งและถูกบังคับเป็นศูนย์ตนเอง |
| `service_id` | required — **ต้องเป็นของศูนย์ที่ scope ไว้** |
| `time_slot_id` | required — ต้องเปิดใน**วันนี้** (ตรวจด้วย `days_mask` ของวันนี้) และต้องเป็นของศูนย์เดียวกัน |
| `staff_id` | optional — บังคับจริงถ้าบริการเปิดให้เลือกบุคลากร ต้องอยู่ใน `service_staff` ของบริการนั้น |
| `patient_right_id` | optional — ต้องเป็นสิทธิของผู้รับบริการที่ `cid` นั้น |

**ศูนย์ที่ถูกบันทึกมาจากบริการที่เลือกเสมอ** — ไม่ได้มาจากค่าที่ส่งมา ดังนั้นการส่ง `health_center_id` ของศูนย์อื่นโดยผู้ดูแลศูนย์จะไม่ทำให้นัดหมายไปตกศูนย์อื่น แต่จะถูกปฏิเสธเพราะบริการที่เลือกไม่ใช่ของศูนย์นั้น

**`appointment_date` ไม่ต้องส่ง** — ระบบใส่วันนี้ให้อัตโนมัติ

**ผู้รับบริการเดิม vs ใหม่**
| กรณี | ระบบทำอะไร |
|---|---|
| `cid` ยังไม่มี | สร้างผู้รับบริการใหม่พร้อมข้อมูลครบ และสร้างสิทธิหลักจาก `right_type_id` (ถ้าส่งมา) |
| `cid` มีอยู่แล้ว + เบอร์โทรและวันเกิดตรง | **เติมเฉพาะช่องที่ว่าง** ชื่อ/ที่อยู่/เพศที่มีอยู่แล้วจะไม่ถูกทับ |
| `cid` มีอยู่แล้ว + เบอร์โทรหรือวันเกิดไม่ตรง | **422** — แปลว่าไม่ใช่ผู้รับบริการคนเดียวกัน ให้เจ้าหน้าที่ตรวจสอบก่อน |
| `cid` ใหม่ + เบอร์โทรไปตรงกับผู้รับบริการคนอื่น | **422 พร้อมเลขบัตรประชาชนของเจ้าของเบอร์** — ไม่ใช่ 500 เพราะเจ้าหน้าที่ต้องรู้ว่าต้องตรวจสอบอะไร |
| ผู้รับบริการเดิมมีสิทธิหลักแล้ว | ใช้สิทธิหลักเดิม ไม่สร้างสิทธิซ้ำ และ `right_type_id` ที่ส่งมาจะไม่ถูกใช้ |
| ไม่ระบุ `patient_right_id` | ใช้สิทธิหลักของผู้รับบริการ (ถ้าไม่มีสิทธิเลย → นัดหมายไม่มีสิทธิผูกไว้) |

**ตรวจเพิ่มก่อนจอง (นอกจากชั้นของการจองปกติ):**
- บุคลากรทั้งหมดลาวันนี้ → **422** `"บริการนี้ปิดชั่วคราว เนื่องจากเจ้าหน้าที่ทั้งหมดลางานในวันนี้"`

> ❌ **ไม่มีการตรวจขอบเขตบริการที่นี่** ต่างจาก B7.2/B7.4/B7.5 ที่ตรวจ
> เพราะ route นี้เปิดให้เฉพาะผู้ดูแลอยู่แล้ว ซึ่งไม่ถูกจำกัดขอบเขตบริการ

> ข้อความ error ที่เกิดระหว่างสร้างนัดหมาย **ไม่ถูกกลืน** — `BookingException` ถูกจับแยกและส่งข้อความจริงออกไปพร้อมรหัสที่ถูกต้อง
> ต่างจาก `POST /v1/patient/book` ที่กลืนเฉพาะข้อผิดพลาดของระบบ (ดูหัวข้อ 0.6)

### B7.4 ย้ายคิวไปบุคลากรคนอื่น

**`PATCH /v1/staff/appointments/{id}/reassign-staff`**
```json
{ "health_center_id": 1, "staff_id": 5 }
```

**ขอบเขตบริการ (403)** — STAFF ต้องเข้าเงื่อนไขอย่างน้อยหนึ่งทางใน 0.3a เหมือน B7.2 และ B7.5
(ทางนี้เป็นที่ที่กติกา "กว้างขึ้น" จริง ๆ เพราะเดิมจำกัดแค่การถูกมอบหมาย ตอนนี้ผู้ที่เป็นผู้ถูกระบุในคิวก็ทำได้)

| เงื่อนไข | ผลลัพธ์ |
|---|---|
| คิวต้องเป็น `CONFIRMED` | 422 |
| บุคลากรคนใหม่ ≠ คนเดิม | 422 `"บุคลากรที่ระบุคือคนเดิมกับที่ยึดถืออยู่แล้ว"` |
| บริการต้องเปิดให้เลือกบุคลากร | 422 `"บริการนี้ไม่เปิดให้เลือกบุคลากร"` |
| คนใหม่ต้อง ACTIVE + ผูกบริการ + ไม่ลา | 422 `"บุคลากรที่เลือกไม่ว่างในวันที่ของคิวนี้"` |
| คนใหม่ต้องไม่เต็มรอบนั้น | 422 `"บุคลากรที่เลือกเต็มคิวในรอบเวลานี้แล้ว"` |

**ตรวจโควตาและบันทึกอยู่ในธุรกรรมเดียวกัน** พร้อมล็อกแถวศูนย์สุขภาพและแถวนัดหมาย
ถ้าโควตาไม่ผ่านจะ rollback ทั้งหมด ไม่เหลือ audit ค้าง
เหตุผล: ผู้ดูแลศูนย์สองคนกดย้ายพร้อมกันแล้วบุคลากรคนเดียวจะได้สองคิวในช่วงเวลาเดียว
ซึ่งแก้ทีหลังได้ยาก เพราะต้องเลื่อนนัดหมายใบหนึ่งออก

เขียน `AuditLog` action `APPOINTMENT_STAFF_REASSIGN` (before/after staff_id)
`StaffAppointmentController::reassignStaff()` · `StaffAppointmentService::reassignStaff()`

### B7.5 เปิดดูข้อมูลผู้รับบริการแบบไม่ปิดบัง (PDPA)

**`POST /v1/staff/appointments/{id}/unmask`** — ไม่ต้องส่ง body

- **ข้อมูลอ่อนไหวตาม PDPA** — เปิดข้อมูลตัวตนจริง
- **throttle `unmask` 10 ครั้ง/นาที** (ดู B2.3)
- **เขียน `AuditLog` action `UNMASK_PATIENT_DATA` ก่อนคืนข้อมูลทุกครั้ง**
- คืน `cid`, `full_name`, `phone_number`, `birth_date`, `gender`, `address`, `rights[]` แบบเต็ม
- **ขอบเขตบริการ (403)** — STAFF ต้องเข้าเงื่อนไขใดเงื่อนหนึ่งใน 0.3a
  คือ มีการมอบหมายบริการนั้น **หรือ** เป็นผู้ถูกระบุในคิวนี้
- **การถูกปฏิเสธจะไม่เขียน audit log** เพราะการเขียน log แปลว่ามีการเปิดข้อมูล ซึ่งไม่ได้เกิดขึ้น
  ⚠️ ต้องจัดการข้อความ 403 นี้ในหน้าจอ ไม่งั้นผู้ใช้จะไม่รู้ว่าทำไมถึงกดไม่ได้
- คิวของศูนย์อื่น → **404** (ไม่ใช่ 403 — ต่างศูนย์คือไม่มีสิทธิ์มอง ไม่ใช่มีแต่ไม่ได้ทำ)

**สองกรณีที่ตอบต่างกัน:**

| กรณี | สถานะ | เหตุผล |
|---|---|---|
| นัดหมายไม่ใช่ของศูนย์ที่เรียก | 404 | ไม่รู้จักว่ามีตัวตนนี้ในศูนย์ |
| เป็นของศูนย์ แต่ไม่ใช่บริการที่มอบหมาย และไม่ได้เป็นผู้ถูกระบุ | 403 | เห็นอยู่ในระบบ แต่แตะไม่ได้ |

`StaffAppointmentController::unmaskPatientData()` · `StaffAppointmentService::unmaskPatientInfo()`

---

## B8. Roster (ทะเบียนบุคลากร/หมอนวด)

> `staff` = **profile คนงาน** (มี `category_id`, `position`, `user_id`) — คนละชั้นกับ `users` = บัญชี login (B12)

### B8.1 ดูรายชื่

**`GET /v1/staff/roster`**

| query | ผล |
|---|---|
| — | HC_ADMIN/STAFF = ศูนย์ตัวเอง; SUPER_ADMIN = ทุกศูนย์ หรือ `?health_center_id=` |
| `?date=YYYY-MM-DD` | เพิ่ม field **`effective_status`** ที่รวม 2 กลไกเข้าด้วยกัน |

**ทุกรายการมี** `id`, `name`, `position`, `category`, `status`, `user`, **`health_center`**

**`effective_status` (มีเฉพาะตอนส่ง `?date=`):**
```
Staff.status = ACTIVE + มี StaffLeave วันนั้น  →  effective_status = 'LEAVE'
กรณีอื่น                                        →  effective_status = ค่าเดิมของ status
```
`StaffRosterService::index()` — logic อยู่ใน service ตอนนี้ `StaffRosterController` แค่เรียกใช้

> **สองกลไกของ "หมอไม่ว่าง" — อย่าสับสน**
> - `StaffLeave` row = **ลาเฉพาะวัน** (B9)
> - `Staff.status = 'LEAVE'` = **ปิดรับจองต่อเนื่อง** (B8.3)
> - roster คือจุดอ่านรวมทั้งสองแบบ

### B8.2 เพิ่มบุคลากร

**`POST /v1/staff/roster`**
```json
{ "name": "แม่หมอ...", "category_id": 1, "position": "หมอนวด", "status": "ACTIVE" }
```
- `name` required, max 255
- `status` **required** ∈ `ACTIVE / INACTIVE / LEAVE`
- `category_id`, `position` optional
- **STAFF ธรรมดาทำได้จริง** (ไม่จำกัดเฉพาะศูนย์ตัวเองถ้า scope ตรง)

### B8.3 สลับเวร (Active ↔ Leave)

**`PATCH /v1/staff/roster/{id}/toggle-duty`**
- `INACTIVE` → **422** `"ไม่สามารถสลับสถานะได้ เนื่องจากบุคลากรถูกระงับการใช้งาน"` (ต้องเปิดใช้งานก่อน)
- `ACTIVE → LEAVE` ที่มีคิว CONFIRMED ตั้งแต่วันนี้ → **422** `"ไม่สามารถปิดสถานะเจ้าหน้าที่ได้ เนื่องจากมีคิวที่จองไว้แล้ว N คิว"`
- `LEAVE → ACTIVE` = เปิดให้จองกลับทันที ไม่มีเงื่อนไขกิจกรรม

`StaffRosterController::toggleDuty()`

---

## B9. วันลา

### B9.1 ดูวันลา

**`GET /v1/staff/leaves?date=&staff_user_id=`**
- HC_ADMIN/STAFF = ศูนย์ตัวเอง; SUPER_ADMIN = ทุกศูนย์
- เรียงวันที่มาก→น้อย
- `staff_user_id` คือ **user id** (บัญชี) ไม่ใช่ staff id

### B9.2 ลงวันลา

**`POST /v1/staff/leaves`**
```json
{ "staff_user_id": 10, "leave_date": "2026-09-26", "reason": "ไปราชการ" }
```

| field | rule |
|---|---|
| `staff_user_id` | required, `exists:users` (ต้องยังไม่ถูกลบ) |
| `leave_date` | required, **≥ วันนี้** (ย้อนหลังไม่ได้) |
| `reason` | optional, max 500 |

**สิทธิ์:** STAFF ธรรมดา → ระบบบังคับ `staff_user_id` = ตัวเองเสมอ · HC_ADMIN → ลงให้ใครในศูนย์ตัวเองก็ได้ · SUPER_ADMIN → ทุกที่

**กติกาสำคัญ 4 ข้อ:**
1. **บล็อก 422** ถ้าหมอคนนั้นมีคิว `CONFIRMED` ในวันที่ลา → `"ไม่สามารถลงวันลาได้ เนื่องจากมีคิวที่จองไว้แล้ว N คิวในวันนี้"`
2. **Idempotent** — ลงซ้ำวันเดิมคืน row เดิม **HTTP 200** (ไม่ใช่ 201, ไม่ error)
3. **บัญชีที่ไม่มีโปรไฟล์บุคลากร ลาไม่ได้** → 422 `"บัญชีนี้ยังไม่มีโปรไฟล์บุคลากรในศูนย์สุขภาพนี้ กรุณาเพิ่มโปรไฟล์ก่อนลงวันลา"`
   ระบบไม่รู้ว่าคนนี้คือใคร การเดาจากหมวดหมู่เคยทำให้ผูกผิดคน จึงไม่เดาแล้ว
4. **วันลาเชื่อผูกกับ `staff.user_id`** — โปรไฟล์ที่ไม่ได้ผูกบัญชีถือว่า "ไม่ลาผ่านระบบนี้" เสมอ

`StaffLeaveController::store()` · `StoreLeaveRequest::rules()`

### B9.3 ยกเลิกวันลา

**`DELETE /v1/staff/leaves/{id}`**

| role | เงื่อนไข | ไม่ผ่าน |
|---|---|---|
| STAFF | ลบได้เฉพาะของตัวเอง | **404** (ไม่ใช่ 403 — ปิดบังการมีอยู่) |
| HC_ADMIN | ลบได้เฉพาะของศูนย์ตัวเอง | 404 |
| SUPER_ADMIN | ลบได้ทุกที่ | — |

### B9.4 เช็คบริการว่างให้บริการวันนั้นไหม

**`GET /v1/staff/leaves/availability?service_id=1&date=2026-09-26`**
```json
{ "success": true, "available": true }
```

**ใช้เมื่อไหร่:** อยากเช็คบริการเดียวโดยไม่ต้องดึงรายการทั้งหมด
**แหล่งข้อมูลหลัก** ของความพร้อมของบริการคือ `GET /v1/staff/services?date=` (B3.1) ซึ่งคืนรายการเฉพาะที่พร้อม — endpoint นี้เป็นตัวย่อยจาก **rule เดียวกัน** (`StaffLeave::isServiceAvailable`)

`StaffLeaveController::availability()`

---

## B10. ค้นหาและแก้ข้อมูลผู้รับบริการ

> **เงื่อนไขร่วมทุก endpoint ในหัวข้อนี้:** ผู้รับบริการต้อง**เคยมีประวัติคิวที่ศูนย์นั้น** — ผู้รับบริการที่ยังไม่เคยมาศูนย์จะไม่ปรากฏและแก้ไขไม่ได้ (tenant isolation ผ่าน queue history ไม่ใช่คอลัมน์บน patient)

### B10.1 ค้นหาผู้รับบริการ

**`GET /v1/staff/admin/patients`** — HC_ADMIN / SUPER_ADMIN (STAFF → 403)

| query | ผล |
|---|---|
| `q` | ค้นชื่อ/นามสกุล (max 100) |
| `cid` | 13 หลัก |
| `phone` | max 20 |
| `gender` | `MALE/FEMALE/OTHER` |
| `province` / `district` | max 100 |
| `health_center_id` | SUPER_ADMIN เท่านั้น; HC_ADMIN ถูกบังคับศูนย์ตัวเอง |
| `per_page` | 1-100, default 15 |
| `sort_by` | `created_at/first_name/last_name/id` |
| `order` | `asc/desc` |

`StaffPatientIndexRequest::rules()` · `StaffPatientController::index()`

### B10.2 แก้เบอร์โทร (STAFF ทำได้)

**`PATCH /v1/staff/patients/{id}/contact`**
```json
{ "phone_number": "0899999999" }
```
- 10 หลักตัวเลข, **unique ใน `patients`**
- ส่งเดิม → `"เบอร์โทรศัพท์ไม่มีการเปลี่ยนแปลง"` + **ไม่เขียน audit log**
- เขียน `AuditLog` (เบอร์ถูก mask ใน payload)

**ขอบเขตบริการ (403) — STAFF ทำได้เฉพาะผู้รับบริการที่ตนเคยให้บริการ:**

ผู้รับบริการ**ไม่ได้ผูกกับบริการโดยตรง** เขาผูกกับผ่านประวัติคิว
จึงต้องเคยมีนัดหมายอย่างน้อยหนึ่งครั้ง **ในบริการที่ถูกมอบหมาย** ของศูนย์นั้น

- **ไม่จำกัดวัน** — เบอร์ที่ผิดมักถูกแก้หลังการมาเยือนเสร็จไปแล้ว การจำกัดวันจะไปโดนเคสที่ต้องใช้จริงที่สุด
- **มีคิวหนึ่งครั้งก็พอ** — ต่อให้ครั้งล่าสุดจะไปบริการอื่น ผู้รับบริการคนเดิมยังแก้เบอร์ได้
- **STAFF ที่ยังไม่ได้รับมอบหมายบริการเลย จะแก้เบอร์ไม่ได้เลยแม้แต่คนเดียว** ต้องผูกบริการก่อน (ดู B5)
- การถูกปฏิเสธ → **ไม่เขียน audit log** เพราะไม่มีการแก้ข้อมูลเกิดขึ้น

**สองกรณีที่ตอบต่างกัน:**

| กรณี | สถานะ | เหตุผล |
|---|---|---|
| ผู้รับบริการไม่เคยมีคิวในศูนย์นี้เลย | 404 | ไม่รู้จักว่ามีตัวตนนี้ในศูนย์ |
| มีคิวในศูนย์ แต่ไม่เคยมีคิวในบริการที่มอบหมาย | 403 | เป็นผู้รับบริการของศูนย์ แต่ไม่ใช่ของฉัน |

> ⚠️ **ข้อจำกัดที่รับไว้โดยเจตนา:** การตรวจเบอร์ซ้ำเกิดที่ FormRequest **ก่อน** ตรวจขอบเขต
> จึงตอบ 422 `"เบอร์โทรโทรศัพท์นี้ถูกใช้งานโดยผู้รับบริการท่านอื่นแล้ว"` โดยไม่ตรวจว่าผู้เรียกอยู่ในขอบเขตหรือไม่
> ผลคือผู้ที่ทราบ patient id สามารถทดสอบว่าเบอร์ใดลงทะเบียนแล้วได้
> การปิดช่องนี้หมายถึงเลิกบอกว่าเบอร์ซ้ำ ซึ่งทำให้ผู้ใช้แก้ผิดแล้วไม่รู้ว่าผิดตรงไหน — เป็นการตัดสินใจเรื่องความเป็นประโยชน์ ไม่ใช่แก้ได้ด้วยการเพิ่ม scope

`UpdatePatientContactRequest::rules()` · `ResolvesHealthCenterScope::denyIfOutOfPatientScope()` · `StaffPatientService::updateContact()`

### B10.3 แก้ข้อมูลเต็มรูปแบบ (HC_ADMIN / SUPER_ADMIN)

**`PATCH /v1/staff/patients/{id}`**
```json
{ "health_center_id": 1, "cid": "5100000000011", "first_name": "...", "birth_date": "1990-01-01" }
```
- ทุก field optional; `cid` unique, `birth_date` ต้องเป็นวันที่ในอดีต
- เขียน `AuditLog` (mask ทั้ง cid และเบอร์)
- **STAFF ธรรมดา → 403** (ต้องใช้ B10.2 แทน)

`UpdatePatientProfileRequest::rules()` · `StaffPatientController::updateProfile()`

---

## B11. จัดการศูนย์สุขภาพ

### B11.1 รายชื่อศูนย์ (dropdown)

**`GET /v1/staff/health-centers`**

| role | คืน |
|---|---|
| SUPER_ADMIN | **ทุกศูนย์ ทุกสถานะ** ไม่ paginate — 12 field เต็ม (id, code, name, phone, subdistrict, district, province, full_address, latitude, longitude, status, has_webhook) |
| HC_ADMIN / STAFF | **ศูนย์ตัวเอง 1 แถว** เฉพาะ ACTIVE/CLOSING (INACTIVE ซ่อน) — เหลือ 4 field: id, code, name, has_webhook |

> สำหรับ HC_ADMIN/STAFF ให้ใช้ `GET /v1/staff/me` แทน — ได้ `health_center` เต็มรูปแบบและมีทุกวัน
> endpoint นี้มีไว้สำหรับ SUPER_ADMIN ที่ต้องเลือกข้ามศูนย์ (สร้างบริการ/ผู้ใช้) เพราะ admin list มี pagination

### B11.2 รายชื่อศูนย์สำหรับจัดการ (SUPER_ADMIN)

**`GET /v1/staff/admin/health-centers`**

| query | ผล |
|---|---|
| `q` | ค้นชื่อ (max 100) |
| `code` | ค้นรหัส (max 20) |
| `province` / `district` | max 100 |
| `status` | `ACTIVE/CLOSING/INACTIVE` |
| `per_page` | 1-100, default 20 |
| `sort_by` | `id/name/code/status/created_at` |
| `order` | `asc/desc` |

shape 12 field เดียวกับ dropdown ของ SUPER_ADMIN แต่ paginate + filter ได้
`StaffHealthCenterIndexRequest.php` · `StaffHealthCenterController.php`

### B11.3 เปิด/ปิดศูนย์ (SUPER_ADMIN)

**`PATCH /v1/staff/admin/health-centers/{id}/toggle-status`**
```json
{ "health_center_id": 1 }
```
- **`health_center_id` ต้องตรงกับ `{id}` ใน path** ไม่ตรง → **404**
- ใช้ `lockForUpdate` + เขียน `AuditLog`
- `ACTIVE ↔ CLOSING`, `INACTIVE → ACTIVE` (เปิดคืนได้)

`StaffHealthCenterController.php`

### B11.4 แก้ข้อมูลศูนย์

**`PATCH /v1/staff/health-centers/{id}`** — HC_ADMIN / SUPER_ADMIN
```json
{ "health_center_id": 1, "name": "...", "phone_number": "...", "full_address": "...",
  "latitude": 13.75, "longitude": 100.50 }
```

| field | rule |
|---|---|
| `code` | unique (max 20) — **HC_ADMIN ส่งมา → 403** `"ไม่มีสิทธิ์แก้ไขโค้ดสถานพยาบาล"`; SUPER_ADMIN แก้ได้ |
| `status` | **ถูกบังคับถอดออกเสมอ** — ต้องใช้ B11.3 แทน |
| `discord_webhook_url` | **ถูกถอดออกเสมอ** — ต้องใช้ B11.5 แทน |
| `latitude` / `longitude` | numeric, -90..90 / -180..180 |

- HC_ADMIN ส่ง `{id}` ของศูนย์อื่น → **404**
- ไม่มี field เปลี่ยน → ข้อความ `"ข้อมูลไม่มีการเปลี่ยนแปลง"` + **ไม่เขียน audit log**

`UpdateHealthCenterRequest.php` · `StaffHealthCenterController.php`

### B11.5 ตั้ง Discord Webhook (SUPER_ADMIN)

**`PATCH /v1/staff/discord/webhook`**
```json
{ "health_center_id": 1, "webhook_url": "https://discord.com/api/webhooks/123456/abcXYZ" }
```
- `webhook_url` ต้อง match regex `^https://discord\.com/api/webhooks/[0-9]+/[A-Za-z0-9_-]+$`, max 500
- ส่ง `null` = **ลบ webhook** (ปิดการแจ้งเตือนของศูนย์นั้น)
- `health_center_id` ไม่ส่ง → 422, ไม่มีจริง → 404
- ถ้าลบ/ตั้งแล้ว `GET /v1/staff/health-centers` จะสะท้อนค่าใหม่ใน `has_webhook`

`UpdateDiscordWebhookRequest.php` · `StaffDiscordController::updateWebhook()`

---

## B12. จัดการผู้ใช้ (บัญชี staff)

> `users` = บัญชี login (มี username/password) — ต่างจาก `staff` ใน roster (B8)
> **HC_ADMIN จัดการได้เฉพาะ role STAFF ในศูนย์ตัวเองเท่านั้น**

### B12.1 ดูรายชื่อผู้ใช้

**`GET /v1/staff/admin/users`** — HC_ADMIN / SUPER_ADMIN

| query | ผล |
|---|---|
| `search` | ค้นชื่อ / username |
| `health_center_id` | SUPER_ADMIN เท่านั้น; HC_ADMIN ถูกบังคับศูนย์ตัวเอง |
| `status` | `ACTIVE/INACTIVE` |
| `role_ids` | กรองตาม role (array) |
| `sort_by` | `id/name/username/status/created_at` |
| `per_page` | default 20 |

`StaffUserManagementController.php`

### B12.2 ดูรายชื่อบทบาทที่ตัวเองมีสิทธิ์มอบ

**`GET /v1/staff/admin/roles`**
- SUPER_ADMIN → ทุก role
- HC_ADMIN → **เฉพาะ `STAFF`** (ใช้เป็นค่าใน dropdown ตอนสร้างผู้ใช้)

### B12.3 สร้างผู้ใช้

**`POST /v1/staff/admin/users`**
```json
{ "name": "...", "username": "...", "password": "อย่างน้อย 8 ตัว",
  "role_ids": [3], "health_center_id": 1, "status": "ACTIVE" }
```

| field | rule |
|---|---|
| `username` | required, max 50, unique |
| `password` | required, **min 8** |
| `email` | optional, unique |
| `role_ids` | required array — **HC_ADMIN ส่ง role อื่น → 422** และ `health_center_id` ถูกบังคับเป็นศูนย์ตัวเอง |
| `health_center_id` | optional; SUPER_ADMIN ส่ง `null` ได้ (ผู้ใช้ไม่ผูกศูนย์) |

เขียน `AuditLog` action `USER_CREATED`
`StoreUserRequest.php` · `StaffUserManagementController.php`

### B12.4 แก้ผู้ใช้

**`PATCH /v1/staff/admin/users/{id}`**

| กรณี HC_ADMIN | ผล |
|---|---|
| แก้บัญชี SUPER_ADMIN | 403 `"ไม่สามารถแก้ไขบัญชีผู้ดูแลระบบได้"` |
| แก้ผู้ใช้ศูนย์อื่น | 404 |
| ส่ง `health_center_id` เพื่อย้ายศูนย์ | **ถูกเมินเสมอ** (`unset`) |
| ส่ง `role_ids` ที่มี role อื่นนอกจาก STAFF | 422 |
| ถอด role หรือปิดบัญชีของ**ผู้ดูแลศูนย์คนใด** | 403 `"การเปลี่ยนสิทธิ์ผู้ดูแลศูนย์ต้องให้ผู้ดูแลระบบเป็นผู้ดำเนินการ"` |

> **HC_ADMIN ถอดสิทธิ์ผู้ดูแลศูนย์ไม่ได้** แม้แต่ตัวเอง ถ้าได้ศูนย์จะไม่เหลือคนทำหน้าเคาน์เตอร์
> เฉพาะการแก้ชื่อหรือเบอร์โทรศัพท์เท่านั้นที่ HC_ADMIN ทำเองได้

**การป้องกัน "คนสุดท้าย" — ใช้เกณฑ์เดียวกันทั้งสองระดับ:**

| กรณี | ผล |
|---|---|
| SUPER_ADMIN คนสุดท้ายที่ยัง ACTIVE → ถอด role หรือตั้ง INACTIVE | 422 |
| HEALTH_CENTER_ADMIN คนสุดท้ายที่ยัง ACTIVE ของศูนย์นั้น → ถอด role หรือตั้ง INACTIVE | 422 |

ใช้นิยาม "คิวค้าง" ตัวเดียวกันคือ `status = CONFIRMED` **และ** `appointment_date >= วันนี้` (`Appointment::scopePending`)

**การตัด token ทันที** — การเปลี่ยนสิ่งที่ยืนยันตัวตนได้ ต้องล้าง session เดิมทิ้งทันที ไม่งั้นคนที่ถูกปิดบัญชีหรือเปลี่ยนรหัสผ่านแล้วยังทำงานต่อได้จนโทเคนหมดอายุ

| การแก้ไข | ตัด token ไหม |
|---|---|
| เปลี่ยน `password` | ✅ ตัดทั้งหมด |
| ตั้ง `status = INACTIVE` | ✅ ตัดทั้งหมด |
| แก้ชื่อ / เบอร์โทรศัพท์ | ❌ ไม่ตัด (ไม่กระทบการยืนยันตัวตน) |

`StaffUserManagementController::update()`

### B12.5 ลบผู้ใช้

**`DELETE /v1/staff/admin/users/{id}`**

| เงื่อนไข | ผลลัพธ์ |
|---|---|
| ลบบัญชีตัวเอง | 422 |
| HC_ADMIN ลบบัญชี SUPER_ADMIN | 403 |
| HC_ADMIN ลบผู้ใช้ศูนย์อื่น | 404 |
| ลบ SUPER_ADMIN คนสุดท้ายที่ยัง ACTIVE | 422 |
| ลบผู้ดูแลศูนย์คนสุดท้ายที่ยัง ACTIVE | 422 `"ไม่สามารถถอดหรือปิดผู้ดูแลศูนย์คนสุดท้ายของศูนย์สุขภาพนี้ได้"` |
| หมอใน roster ของบัญชีนี้มีคิว CONFIRMED อนาคต | 422 `"ไม่สามารถลบบัญชีผู้ใช้ได้ เนื่องจากมีคิวที่จองไว้แล้ว N คิว"` |

**ลบแล้วเกิดอะไรขึ้น (soft delete + cleanup):**
1. โปรไฟล์ `staff` ที่ผูก `user_id` → ตั้ง `status=INACTIVE` **และ `user_id=null`**
   ตัดออกจากระบบ เพื่อไม่ให้ผู้รับบริการเลือกจองบุคลากรคนนี้ได้อีก
   โปรไฟล์จึงกลายเป็น "คนงานที่ไม่มีบัญชี" แบบเดียวกับโปรไฟล์ที่สร้างมาก่อนมีระบบบัญชี
   — ยังให้บริการได้ถ้าไม่ปิดสถานะ แต่ลาไม่ได้
2. ถอดออกจาก `service_user` ทุกบริการ
3. **revoke token ทั้งหมด** → session ที่ยัง active จะใช้ไม่ได้ทันที
4. soft delete user → ซ่อนจาก admin list และ login ไม่ได้
5. เขียน `AuditLog` action `USER_DELETED`

`StaffUserManagementController::destroy()`

---

# Part C — Invariant และข้อจำกัดที่ต้องระวัง

## C.1 ✅ Time Slots บังคับ `health_center_id`

**อาการเดิม:** SUPER_ADMIN เรียกคำสั่งช่วงเวลาโดยไม่ส่ง `health_center_id` → ได้ช่วงเวลาที่ `health_center_id = null` และแก้/ลบข้ามศูนย์ได้

**สาเหตุ:** `StaffTimeSlotController` ไม่ได้ใช้ trait `ResolvesHealthCenterScope` แต่เขียน `healthCenterId` เองทุก method และ service ใช้ `if ($healthCenterId)` เป็นตัวตัด scope ซึ่งถ้าไม่ส่งค่าจะข้ามการกรองไปทั้งหมด

**สิ่งที่แก้ — บังคับ 3 ชั้น:**
1. **ชั้นบริการ** — ทุก method รับศูนย์แบบบังคับ (ไม่รับค่าว่าง) ตัดเส้นทาง "ไม่ส่งศูนย์ = ไม่จำกัดศูนย์" ทิ้งทั้งหมด
2. **ชั้น controller** — ใช้ตัวแก้ scope กลางทุก endpoint (ดู/เพิ่ม/แก้/สลับสถานะ) ผู้ดูแลระบบต้องระบุศูนย์ แตะข้ามศูนย์ไม่ได้
3. **ชั้นฐานข้อมูล** — คอลัมน์ศูนย์ของช่วงเวลาเป็น NOT NULL

**ผลข้างเคียงที่ต้องจำ:** `abort()` ในตัวแก้ scope ถูก `try/catch` ทั่วไปกินได้
ทำให้การไม่ระบุศูนย์ตอบ 422 ที่ข้อความว่าง — ✅ แก้แล้วทุกจุดที่พบ
แต่ **เป็นกับดักที่จะเกิดซ้ำได้** เมื่อเพิ่ม endpoint ใหม่

→ ต้องเรียก `resolveTargetHealthCenterId()` **ก่อนเข้า `try`** เสมอ (ดู 0.3)

**สิ่งที่เปลี่ยนตามมา:**
- การลบช่วงเวลาไม่รองรับอีกต่อไป ใช้ปิดใช้งานแทน (ADR-0001) — ดู B6
- การตรวจช่วงเวลาทับกันเปลี่ยนจาก "เทียบเวลาให้ตรงกันทุกตัว" เป็นการตรวจการทับจริง
- ตอนสร้างช่วงเวลา ส่ง `is_active: false` ได้ (เดิมไม่ได้ประกาศฟิลด์นี้ ทำให้ค่าถูกตัดทิ้ง)
- การตอบกลับระบุ `health_center` ของช่วงเวลาทุกรายการ

**บันทึกไว้ที่:** `Roles.md` หัวข้อ gap ข้อ 2 (ปิดแล้ว)

## C.1a ✅ ทุกทางที่แตะคิวถูกจำกัดด้วยขอบเขตบริการ

**เหตุที่ต้องบังคับ:** รายการคิวถูกจำกัดเฉพาะบริการที่มอบหมายอยู่แล้ว แต่ endpoint ที่รับเลขนัดหมายตรง ๆ เคยไม่ตรวจเลย
ผู้ที่ทราบเลขก็เปลี่ยนสถานะหรือเปิดข้อมูลผู้รับบริการของบริการอื่นในศูนย์ได้

**สาเหตุ:** กติกาถูกเขียนแยกจุด จึงมีเพียงทางเดียวที่ได้รับการป้องกัน
ระบบเชื่อว่าหน้าจอจะไม่ส่งเลขของบริการอื่นมาให้ ซึ่งเป็นการฝากความปลอดภัยไว้กับส่วนที่ควบคุมไม่ได้

**สิ่งที่แก้ — กติกาอยู่ที่จุดเดียว:**
1. กติกาใน trait `ResolvesHealthCenterScope` (`denyIfOutOfServiceScope`)
2. ใช้โดย `reassign-staff` · `updateStatus` · `unmask` · และการกรองรายการใน `index`
3. `User::assignedServiceIds()` เป็นที่เดียวที่นิยามความหมาย "บริการที่ได้รับมอบหมาย"

**หลักการที่ต้องรักษาเมื่อเพิ่มทางใหม่:** กติกาที่เขียนแยกจุดจะไม่ขยายไปทางที่เพิ่มในอนาคต ทางที่แตะคิวทุกทางต้องผ่านตัวตรวจเดียวกัน

**หลักการที่ต้องรักษาเมื่อแก้เอกสาร/ข้อความ:** ถอด audit log ไม่ได้ — ถ้าเขียน log แล้วค่อยปฏิเสธ จะได้หลักฐานว่าเปิดข้อมูล ทั้งที่ไม่ได้เปิด

**ผลข้างเคียง:** การตรวจขอบเขตมาก่อนกฎการเปลี่ยนสถานะ ผู้เรียกจะได้ 403 แทน 422 เมื่อส่งสถานะผิดวิธีไปยังบริการที่ตนแตะไม่ได้ ซึ่งถือว่าถูกต้อง เพราะไม่เปิดเผยว่าสถานะไหนถูก

ดูรายละเอียดที่ 0.3a

## C.2 Invariant "คิว CONFIRMED อนาคตกันการเปลี่ยน config"

ระบบจะปฏิเสธการเปลี่ยนแปลงเหล่านี้ ถ้ายังมีคิว `CONFIRMED` ตั้งแต่วันนี้ขึ้นไป:

| สิ่งที่พยายามทำ | ผล |
|---|---|
| ลบบริการ | 422 `"ไม่สามารถลบบริการได้ เนื่องจากมีคิวล่วงหน้าอยู่ N คิว"` |
| ปิด `allow_staff_selection` | 422 `"ไม่สามารถปิดการเลือกเจ้าหน้าที่ได้..."` |
| ถอดหมอออกจากบริการ | 422 `"ไม่สามารถแกะเจ้าหน้าที่ออกจากบริการได้..."` |
| ลบช่วงเวลา (time slot) | ไม่มีการลบเลย — ปิดใช้งานด้วย `toggle-status` แทน (ADR-0001) |
| ปิดวันให้บริการ (`time-slot-days`) | 422 `"ไม่สามารถลบวันให้บริการได้ เนื่องจากมีคิวที่ยืนยันแล้วในอนาคต..."` |
| ลงวันลาในวันที่มีคิว | 422 `"ไม่สามารถลงวันลาได้ เนื่องจากมีคิวที่จองไว้แล้ว N คิวในวันนี้"` |
| ปิดเวรหมอ (`ACTIVE → LEAVE`) | 422 `"ไม่สามารถปิดสถานะเจ้าหน้าที่ได้..."` |
| ลบผู้ใช้ที่มีคิวผูกกับหมอ | 422 `"ไม่สามารถลบบัญชีผู้ใช้ได้..."` |

รวมตัวกันแล้วใน `StaffServiceManagementService::assertNoPendingStaffQueues()` (app/Services/Staff/StaffServiceManagementService.php) และ `OperatingDayService::hasFutureConfirmedBookingsOnDays()` (app/Services/OperatingDayService.php)

**หมายเหตุ:** เกณฑ์คือ "ตั้งแต่วันนี้ขึ้นไป" — **คิวในอดีตที่ยัง CONFIRMED อยู่ ไม่กันการเปลี่ยน config** (เช่น ย้อนหลังมาปิดบริการได้)

## C.3 สิทธิ์ PDPA

**ข้อมูลถูก mask เสมอใน response** ยกเว้นเรียก unmask:
- `masked_full_name` = 2 ตัวแรก + `*** ` + 2 ตัวแรกนามสกุล + `***`
- `masked_phone` = 3 ตัวแรก + `-***-` + 4 ตัวท้าย
- `masked_cid` = 1 ตัวแรก + `-****-*****-` + 2 ตัวท้าย

เป็น accessor ระดับ model จึงใช้กับทุกทางโดยอัตโนมัติ — รวมถึง `appointments` ใน `/dashboard/bundle`

> ⚠️ **ชื่อ field ใน response ของรายการคิวคือ `masked_name` / `masked_phone`** (ไม่ใช่ `masked_full_name`)
> ส่วน accessor ชื่อ `masked_full_name` — ชื่อใน JSON ถูกตั้งใน resource
> ดู `AppointmentResource::patient()`

**หน้าที่ที่เขียน `AuditLog` (PDPA):**

| action | เกิดเมื่อ |
|---|---|
| `UNMASK_PATIENT_DATA` | ทุกครั้งที่เปิดข้อมูลจริง (มี throttle 10/นาที) — **ไม่เขียนเมื่อถูกปฏิเสธ** |
| `LOOKUP_PATIENT_BY_CID` | ค้นผู้รับบริการด้วยเลขบัตรประชาชนแล้วพบ (ค้นไม่พบไม่เขียน เพราะไม่ได้เปิดข้อมูลอะไรออกไป) |
| `USER_CREATED` / `USER_UPDATED` / `USER_DELETED` | จัดการบัญชี |
| `UPDATE_HEALTH_CENTER` | แก้ข้อมูลศูนย์ (เฉพาะค่าที่เปลี่ยนจริง) |
| `UPDATE_HEALTH_CENTER_STATUS` | เปิด/ปิดศูนย์ |
| `UPDATE_DISCORD_WEBHOOK` | ตั้งหรือลบ webhook |
| `UPDATE_PATIENT_CONTACT` / `UPDATE_PATIENT_PROFILE` | แก้ข้อมูลผู้รับบริการ (mask cid/phone ใน payload) |
| `SERVICE_SELECTABLE_STAFF_SYNC` / `SERVICE_TIME_SLOT_DAYS_SYNC` | ผูกหมอ / ตั้งวันเวลา |
| `APPOINTMENT_STAFF_REASSIGN` | ย้ายคิวหมอ |
| `AUTO_CLOSE_HEALTH_CENTER` | ระบบปิดศูนย์อัตโนมัติ (`user_id = null`) |

> ชื่อ action ทั้งหมด grep ได้จาก `'action' => '...'` ทั่ว `app/`
> ตรวจจากตรงนั้นก่อนใช้ query audit เสมอ ชื่อไม่ได้ถูกรวบรวมไว้ที่จุดเดียว

> หมายเหตุ: `USER_*` และ `UPDATE_HEALTH_CENTER*` **ไม่ mask** เพราะไม่ใช่ข้อมูลส่วนบุคคลของผู้รับบริการ
> `audit_logs` มีเพียงคอลัมน์ `payload` (json) — ไม่มีคอลัมน์ `masked_data` แยก
> การ mask จึงเกิดในแต่ละ service ที่สร้าง payload ไม่ใช่ที่ชั้นฐานข้อมูล

## C.4 Discord notification

- ถูก dispatch **หลัง commit** (นอก transaction) — ไม่ rollback การจองเมื่อส่ง Discord ไม่สำเร็จ
- queue = `discord`, retry 3 ครั้ง (backoff 30 / 120 / 600 วินาที)
- ศูนย์ไม่มี webhook → `Log::warning` แล้วจบ ไม่ error (ไม่กระทบการจอง)
- ข้อความใน embed ใช้ชื่อผู้รับบริการแบบ **masked**
- **ต้องมี `queue_worker` รันอยู่** ถึงจะส่งจริง — ถ้า worker ไม่ทำงาน job จะสะสมใน Redis
`BookingService.php` · `SendDiscordNotificationJob.php`

## C.5 ข้อจำกัดอื่นที่ควรรู้

| ข้อจำกัด | รายละเอียด |
|---|---|
| **เลขคิวรีเซ็ตทุกวัน + ใช้ร่วมทุกบริการ** | `appointment_number` scope ที่ `(health_center_id, appointment_date)` — ไม่ใช่รายบริการ ถ้าต้องการแยกลำดับต่อบริการต้องเปลี่ยน scope |
| **ไม่มี seeder ข้อมูลจริง** | `database/seeders/` มีแต่ตัวอย่างสำหรับ dev — ข้อมูลจริงเป็นของที่ import มา **จำนวนศูนย์/บริการเปลี่ยนไปเรื่อย ๆ จึงไม่บันทึกไว้ในเอกสารนี้ ให้นับจากฐานข้อมูลที่ใช้งานจริงเสมอ** |
| **ไม่มี API จัดการ `categories` / `right_types`** | ตารางอ้างอิงเหล่านี้ไม่มี CRUD endpoint — ต้องแก้ผ่าน DB โดยตรง |
| **`service_user` อาจยังว่างทั้งหมด** | ตรวจจากฐานข้อมูลที่ใช้งานจริง — ถ้ายังไม่มีการ assign บริการให้ STAFF เจ้าหน้าที่จะเห็นคิวว่างและแตะคิวไหนก็ได้ 403 (ดู 0.3a) ต้องผูกบริการก่อนใช้งานจริง บริการที่ไม่ผูกชื่อบุคลากรจะถูกคุมความพร้อมด้วย capacity ล้วน |
| **มี scheduler แล้ว แต่ auto-close ยังทำงานต่อเมื่อมี action ต่อคิวเท่านั้น** | ปิดคิวของวันที่ผ่านมารันทุกวัน 00:05 แต่การปิดศูนย์เมื่อคิวหมดยังรอ action จากเจ้าหน้าที่ (ดู 0.5) |
| **ตัวตรวจวันให้บริการรับวันที่ที่ส่งเข้ามา** | ใช้วันในพารามิเตอร์ ไม่ใช่วันของเครื่อง จึงข้ามเที่ยงคืนได้โดยไม่ตรวจผิดวัน |
| **สิทธิ์เจ้าหน้าที่มาจากฐานข้อมูลอย่างเดียว** | token ไม่ได้ใช้ตัดสิทธิ์ ถ้าไม่มีบทบาทในฐานข้อมูลจะเข้าฝั่งเจ้าหน้าที่ไม่ได้ |
| **ตารางเชื่อมบริการ↔ช่วงเวลามีคอลัมน์ศูนย์ที่ต้องเขียนค่าเองทุกจุดเขียน** | ค่าผิดจะถูกฐานข้อมูลปฏิเสธ แต่ถ้าเพิ่มจุดเขียนใหม่ต้องส่งศูนย์มาด้วย ไม่งั้นจะ NOT NULL |
| **`RoleSeeder` รันซ้ำได้ปลอดภัย แต่ seeder อื่นยัง truncate** | `RoleSeeder` ไม่ truncate แล้ว (อ้างอิงด้วย `name`) · `HealthCenterSeeder` / `ServiceSeeder` / `StaffSeeder` / `UserSeeder` ยัง truncate ตาราง — **ห้ามรันบนฐานข้อมูลที่มีข้อมูลจริง** |
| **ดัชนีฐานข้อมูลที่เกี่ยวกับคิวมีอยู่แล้ว** | `idx_appointments_capacity_lookup (health_center_id, service_id, appointment_date, time_slot_id, status)` ครอบการนับโควตาและ `lockForUpdate` · `idx_appointments_staff_lookup (staff_id, appointment_date, status)` ครอบการนับคิวรายหมอ · `idx_appointments_status_date` ครอบ scheduler — **อย่าเพิ่มดัชนีซ้ำซ้อนกับหัวคอลัมน์เหล่านี้** เพราะจะทำให้ช้าลง |
| **ไม่มี ETag / 304** | ระบบไม่มี cache ระดับ HTTP — ทุกคำขอได้ payload เต็ม ถ้าจะเพิ่มต้องระวังเรื่องข้อมูลข้ามผู้ใช้ เพราะคิวมีข้อมูลผู้รับบริการ |
| **ตัวเลขนับผู้รับบริการในหน้าจอต้องตรงกับรายการที่เห็น** | ถ้าเรียก `/dashboard/summary` แยกจาก `/appointments` แล้ววันที่ของทั้งสองไม่ตรงกัน ตัวเลขจะไม่ตรง — ใช้ `/dashboard/bundle` แก้ (ดู B2.2) |

---

## ภาคผนวก — ตารางอ้างอิงเร็ว

### สถานะ (enum ที่ใช้ทั้งระบบ)

| ค่า | ของ | หมายเหตุ |
|---|---|---|
| `ACTIVE` / `CLOSING` / `INACTIVE` | `health_centers.status` | ดู 0.5 |
| `ACTIVE` / `INACTIVE` | `users.status` | soft delete = INACTIVE |
| `ACTIVE` / `LEAVE` / `INACTIVE` | `staff.status` | `INACTIVE` สลับ duty ไม่ได้ |
| `CONFIRMED` / `COMPLETED` / `CANCELLED` / `NO_SHOW` | `appointments.status` | เปลี่ยนได้เฉพาะจาก CONFIRMED |
| `PER_MASSEUSE` / `PER_SLOT` / `PER_DAY` | `services.capacity_type` | ดู 0.4 |
| `SELF` / `STAFF` | `appointments.booked_by_type` | จองเอง / walk-in |
| `MALE` / `FEMALE` / `OTHER` | `patients.gender` | |

### ตาราง pivot ที่ต้องรู้

| pivot | ความสัมพันธ์ | ใช้ทำอะไร |
|---|---|---|
| `service_staff` | service ↔ staff (หมอที่เลือกได้) | **เลือกหมอ** + กรองความพร้อม |
| `service_user` | service ↔ user (role STAFF) | **ทางที่ 1 ของขอบเขตบริการ** — จำกัดว่า STAFF เห็น/แตะคิวของบริการไหนได้ |
| `service_time_slot` | service ↔ time_slot + `days_mask` + `health_center_id` | **วัน/เวลาที่เปิดให้บริการ** — คอลัมน์ศูนย์ต้องตรงกับทั้งบริการและช่วงเวลา (composite FK) |
| `user_roles` | user ↔ role | ตรวจสิทธิ์ทุก request |
| `patient_rights` | patient ↔ right_type | สิทธิการรักษา (มี `is_primary` — บังคับให้มีได้เพียงหนึ่งรายการต่อคน) |

### ไฟล์ที่ต้องเปิดดูเมื่อ debug

| ปัญหา | ไฟล์ |
|---|---|
| จองไม่ผ่าน / โควตาเต็ม | `BookingService.php` + `storage/logs/laravel.log` |
| คิวหมอไม่โผล่ใน `available-slots` | `StaffLeave.php` (`getSelectableStaff`) |
| slot ไม่โผล่ในวันที่เลือก | `OperatingDayService.php` (`isSlotAvailableOnDate`) + `service_time_slot.days_mask` |
| เห็นข้อมูลศูนย์อื่น | `ResolvesHealthCenterScope.php` (`resolveFilterHealthCenterId` / `resolveTargetHealthCenterId`) + `routes/api.php` (middleware) |
| ได้ 403 ทั้งที่ควรได้ | `ResolvesHealthCenterScope.php` (`denyIfOutOfServiceScope` / `denyIfOutOfPatientScope`) + ตรวจ `service_user` และ `appointments.staff_id` ในฐานข้อมูล |
| เห็นคิวว่างแต่ควรมี | ตรวจว่าบัญชีนั้นถูก assign บริการแล้วหรือยัง (`service_user`) |
| โดน 429 | `AppServiceProvider::rateLimitFor()` + ดู B2.3 |
| auto-close ไม่ทำงาน | `CloseHealthCenterJob.php` + สถานะ `queue_worker` |
| validation message ไม่ตรงที่คาด | `app/Http/Requests/Api/**` (messages อยู่ในไฟล์เดียวกัน) |

### วิธีตรวจว่าเอกสารนี้ยังตรงกับโค้ด

เอกสารนี้อ้างอิงตำแหน่งด้วย **ชื่อฟังก์ชันและชื่อไฟล์** ไม่ใช้เลขบรรทัด
เพราะเลขบรรทัดหมดอายุทันทีที่มีการแก้ไฟล์นั้น และเอกสารนี้เคยตกเพราะเหตุนี้

เมื่อสงสัยว่าข้อไหนยังจริง ให้ grep ตามชื่อฟังก์ชันที่ระบุ เช่น

```
grep -rn "mayActOnService" app/
```

สิ่งที่ควรตรวจเป็นพิเศษเมื่อมีการเปลี่ยนแปลง:
- **ตัวเลขสรุปในเอกสาร** — นับจากฐานข้อมูลจริงเสมอ อย่าใส่ตัวเลข snapshot
- **ชื่อ action ของ audit log** — grep `'action' => '` ทั่ว `app/`
- **ชื่อ field ใน response** — ดู resource ที่เกี่ยวข้อง เพราะ accessor กับชื่อใน JSON ไม่จำเป็นต้องตรงกัน
- **เงื่อนไขการจำกัดสิทธิ์** — ดู route group ใน `routes/api.php` ก่อนสรุปว่าใครเข้าถึงอะไรได้
