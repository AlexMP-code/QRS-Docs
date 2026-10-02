# คู่มือลำดับการเรียก API ทุก Feature (QRS)

> **อ้างอิงเอกสาร:** `Roles.md` (สิทธิ์/ขอบเขต) · `openapi.yaml` (schema) · `Feature-First.yaml` (flow ตามหน้าจอ)
>
> เอกสารนี้เขียนให้ตรงกับ **พฤติกรรมจริงของ code**
>
> **การอ้างอิงตำแหน่งในโค้ด** ใช้รูปแบบ `ไฟล์.php` หรือ `ClassName::method()` เท่านั้น
> ไม่บันทึกเลขบรรทัดหรือ commit เพราะหมดอายุทันที่มีคนแก้ไฟล์นั้น — ซึ่งเกิดขึ้นเสมอ
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
    - [B7.3 จองคิวให้คนไข้ (walk-in / ล่วงหน้า)](#b73-นัดหมายเดินเข้ารับบริการที่หน้าเคาน์เตอร์-walk-in)
    - [B7.6 เลื่อนคิวไปวันอื่น](#b76-เลื่อนคิวไปวันอื่น)
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
| 4 | ตัวเลขสรุปต้องตรงกับรายการที่เห็น — ใช้ bundle เพื่อให้แน่ใจ | [B2.2](#b22-หน้าจอหนึ่งจอ--สามชิ้นในคำขอเดียว-แนะนำ) |

> หมายเหตุเรื่องตัวเลข: เอกสารนี้จงใจ**ไม่บันทึกตัวเลขจำนวนศูนย์ จำนวนบริการ หรือจำนวนบทบาท**
> เพราะเป็นค่าจากฐานข้อมูลจริงที่เปลี่ยนไปเรื่อย ๆ และเคยทำให้คนอ่านเชื่อว่าเป็นค่าปัจจุบัน
> ให้นับจากฐานข้อมูลที่ใช้งานจริงเสมอ

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

**ล้มเหลวจาก `BusinessRuleException`** — มี `success: false` เหมือน controller ที่ทำเอง
แต่ข้อความมาจาก service ที่บังคับกติกาทางธุรกิจ ไม่ใช่จาก validation

> ⚠️ `errors` จะมีเฉพาะ validation ระดับ Request — ดูหัวข้อ 0.6 เรื่อง error ที่ถูกกลืน
> **client ต้องอ่าน `message` เป็นหลัก** เพราะทั้งสามแบบมี `message` แต่มีเฉพาะบางแบบที่มี `errors`
> ถ้าอาศัยแค่ `errors` จะพลาดข้อความจาก service ทั้งหมด

**404 จาก `findOrFail` มีข้อความกลางเดียว** — `"ไม่พบข้อมูลที่ระบุ"`
ไม่ได้บอกว่าอะไรไม่พบ จึงอย่าคาดหวังข้อความเฉพาะจาก `findOrFail`
(ส่วนที่ปฏิเสธเองด้วย `response()->json()` จะมีข้อความเฉพาะ)

**500 ที่ไม่คาดคิดถูกปิดรายละเอียด** — ข้อความรวมคือ `"เกิดข้อผิดพลาดของระบบ กรุณาลองใหม่อีกครั้ง"`
ไม่เผยรายละเอียดภายใน เพราะอาจมีโครงสร้างฐานข้อมูลติดไป
ยกเว้น validation / authentication / authorization / throttle ที่ปล่อยให้ Laravel จัดการ
ตั้งค่าที่ `bootstrap/app.php` (`withExceptions`)

## 0.2 Token

| ฝั่ง | ได้ token จาก | Token ability | Middleware |
|---|---|---|---|
| Staff | `POST /v1/staff/login` | `role:staff` | `auth:sanctum` + `role:STAFF,HEALTH_CENTER_ADMIN,SUPER_ADMIN` |
| Patient | `POST /v1/patient/login` หรือ `/register` | `role:patient` | `auth:sanctum` + `abilities:role:patient` |

ส่งทุก request หลัง login: `Authorization: Bearer <token>`

- **Patient token → staff route = 403** (ไม่มี role ใน `users`)
- **Staff token → patient route = 403** (ability ไม่ใช่ `role:patient`)
- **ไม่มี token / token หมดอายุ → 401** ทุกฝั่ง

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
| `PATCH .../reschedule` | เหมือนกัน |
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
| มี `health_center_id` | ✅ บังคับ NOT NULL (รวมผู้ดูแลระบบ — ดู B12.3) | ✅ |
| ความสัมพันธ์ | `users.id` ↔ `staff.user_id` | **1 ต่อ 1 ฝั่ง `users`** (`uq_staff_user_id` unique) |
| ใช้ทำอะไร | login, สิทธิ์, โควตา `PER_MASSEUSE` | เข้าเวร, วันลา, โควตาต่อหมอ, ผูกบริการ |

> ⚠️ **1 ต่อ 1 ฝั่ง `users`** — `users` เดียวผูกได้โปรไฟล์เดียว (unique บังคับ)
> แต่ `staff` เดียวผูก `users` ได้หลายแถวในทางทฤษฎี (คอลัมน์ nullable ไม่มี unique ฝั่ง `staff`)
> ระบบปกติจะไม่เกิดกรณีนี้ เพราะทุกคำขอสร้างผู้ใช้สร้างโปรไฟล์ใหม่เสมอ (B12.3) — ถ้าเจอ แปลว่ามีจุดเขียนที่หลุดธรรมชาติ

- **ทุกบัญชีที่สร้างต้องมีโปรไฟล์ `staff`** — สร้างในคำขอเดียวกัน (ดู B12.3)
- **โปรไฟล์เก่าที่ไม่มีบัญชียังทำงานได้** — `staff.user_id` เป็น nullable จึงยังให้บริการและนับโควตาได้ เขาแค่ลาไม่ได้
- **บัญชีที่ไม่มีโปรไฟล์ ลงวันลาไม่ได้** (422) เพราะระบบไม่รู้ว่าคนนี้คือใคร การเดาจากหมวดหมู่เคยทำให้ผูกผิดคน
- **สร้างโปรไฟล์เดิมซ้ำไม่ได้** — `uq_staff_user_id` บังคับ ถ้าบัญชีนั้นมีโปรไฟล์แล้วส่งมาใหม่ = 422 (ดู B8.2)
- **ลบบัญชี** → โปรไฟล์ถูกตั้ง `user_id = null` + `status = INACTIVE` (ดู B12.5) ทำให้ผู้รับบริการเลือกจองเขาไม่ได้อีก
- **เจ้าหน้าที่สร้างโปรไฟล์ผ่าน `POST /roster` จะรับ `user_id` ที่ยังไม่มีโปรไฟล์** — ทางนี้ไม่ได้สร้างบัญชีใหม่

## 0.4 `capacity_type` — 3 แบบ (กำหนดว่านับโควตายังไง)

`CapacityService::getMaxCapacity()` (app/Services/CapacityService.php)

| ค่า | โควตาต่อรอบเวลามาจาก | ต้องส่งตอนสร้าง | ลักษณะ |
|---|---|---|---|
| `PER_MASSEUSE` | นับบัญชีที่ **ถูกมอบหมายบริการนี้** (`service_user`) และยังไม่ลาวันนั้น ไม่ใช่ทั้งหมวดหมู่ | — | 1 คิว/หมอ/รอบ |
| `PER_SLOT` | `service_capacity.max_per_slot` (ค่า default 1) | `max_per_slot` | ตายตัวต่อรอบเวลา |
| `PER_DAY` | `service_capacity.max_per_day` (ค่า default 10) | `max_per_day` | **นับรวมทั้งวัน ไม่ล้างทุกรอบ** |

> ⚠️ `PER_MASSEUSE` นับจาก **`service_user` (บัญชีที่ถูกมอบหมาย)** ไม่ใช่ `service_staff` (หมอที่ผู้รับบริการเลือกได้)
> เป็นคนละความหมายกัน — ดู 0.3b และ B5 · ถ้ายังไม่มีการมอบหมายบริการให้ใคร
> โควตาของ `PER_MASSEUSE` จะเป็น **0 ทั้งที่มีหมอผูกไว้แล้ว**
>
> **วันลาถูกหักออกจากโควตา** — ผู้ที่ลาวันนั้นไม่นับ (นับผ่าน `users.id`)
> ผู้ที่ผูกกับบริการแต่ **ไม่มีบัญชี** จะไม่มีวันลาในระบบ จึงไม่ถูกหัก — เขายังนับโควตาตามปกติ

**สองอันดับที่คนสับสนบ่อย:**
1. **`allow_staff_selection=true` override ทุกอย่าง** — ข้าม `capacity_type` ไปนับ `selectableStaff()` แทน โดยนับเฉพาะ `status = ACTIVE` และไม่ลาวันนั้น
2. **โควตา "ต่อหมอ = 1 คิว/รอบ"** (`CapacityService::PER_STAFF_PER_SLOT_LIMIT`) บังคับ**เฉพาะตอนเลือกหมอ** ไม่เกี่ยวกับ `capacity_type` อื่น

`CapacityService::getMaxCapacity()` · `CapacityService::countAssignedStaff()`

## 0.5 วงจรชีวิตศูนย์สุขภาพ

```
ACTIVE ──toggle (ยังมีคิวค้าง)──> CLOSING ──คิว CONFIRMED หมด──> INACTIVE
   │                                │
   │                                └── toggle ──> ACTIVE (เปิดใหม่ได้)
   └── toggle (ไม่มีคิวค้าง)──> INACTIVE  (ข้าม CLOSING)
```

> ระบบไม่มีสถานะระหว่างที่ต้องรอเสมอ — ถ้าไม่มีคิวค้างตอนกดปิด ก็เข้า `INACTIVE` เลย

| สถานะ | รับจองใหม่ | staff login | เห็นใน patient API |
|---|---|---|---|
| `ACTIVE` | ✅ | ✅ | ✅ |
| `CLOSING` | ❌ | ✅ (ยังจัดการคิวค้างได้) | ✅ |
| `INACTIVE` | ❌ | ❌ 403 | ❌ 404 |

- `HealthCenter::isBookable()` = `status === 'ACTIVE'` (app/Models/HealthCenter.php)
- Login บล็อกเฉพาะ INACTIVE (app/Http/Controllers/Api/Staff/StaffAuthController.php)

**ปิดศูนย์แบบไม่รอคิวหมด** — ดูแผนที่สถานะที่ B11.3
กด toggle จาก `ACTIVE` ตอนที่ยังไม่มีคิวค้างเลย → เป็น `INACTIVE` ทันที ข้ามขั้น `CLOSING`

**Auto-close ไม่มี cron** — ถูก dispatch เมื่อ staff เปลี่ยนสถานะคิวเป็น terminal
หรือผู้รับบริการกดยกเลิก ขณะศูนย์เป็น `CLOSING` (`StaffAppointmentService.php`, `PatientHistoryController.php`)
→ `CloseHealthCenterJob` เช็คว่าไม่มีคิว `CONFIRMED` แล้วเปลี่ยนเป็น `INACTIVE` + audit

## 0.6 error สองชนิดที่ต้องแยกให้ออก

`POST /v1/patient/book` แยกข้อผิดพลาดออกเป็นสองชนิด

| ชนิด | สถานะ | ผู้รับบริการเห็น |
|---|---|---|
| **เงื่อนไขทางธุรกิจ** (`BookingException`) | ตามรหัสที่กำหนด เช่น 422 / 409 | ข้อความจริง เช่น `"ท่านมีรายการจองบริการนี้ในวันที่เลือกเรียบร้อยแล้ว"` |
| **สิทธิการรักษาไม่ตรงกับผู้รับบริการ** | 422 | `"สิทธิการรักษาที่ระบุไม่พบในข้อมูลของผู้รับบริการคนนี้"` — ปฏิเสธแทนที่จะเงียบแล้วสร้างนัดหมายที่ไม่มีสิทธิผูกไว้ |
| **ข้อผิดพลาดของระบบ** (อย่างอื่น) | 500 | `"ไม่สามารถจองคิวได้ กรุณาลองใหม่อีกครั้ง"` — ไม่เผยรายละเอียดภายใน |

> ถ้า debug แล้วจองไม่ผ่าน แต่ข้อความเป็นกลาง แปลว่าตกไปที่ชั้นระบบ
> → ดู `storage/logs/laravel.log` (มี `Booking failed: <ข้อความจริง>`)
>
> validation ของ FormRequest (เช่น `staff_id` ไม่ผูกกับบริการ) **ไม่ผ่านเส้นทางนี้**
> จึงตอบ 422 พร้อม `errors` ชัดเจนเสมอ
>
> ฝั่งหน้าเคาน์เตอร์ (`POST /walk-in`) **ไม่กลืนข้อผิดพลาดของระบบ** — ดู B7.3

## 0.7 เลขคิว (queue number)

- `appointment_number` = `max(appointment_number)` ของ `(health_center_id, appointment_date)` + 1
- `queue_no` = `"Q-"` + เติมศูนย์ข้างหน้า 3 หลัก → `Q-001`, `Q-002`
- **รีเซ็ตทุกวัน และใช้ร่วมกันทุกบริการของศูนย์นั้น** (ไม่ใช่เลขแยกรายบริการ)
- **ล็อกที่แถวศูนย์สุขภาพ ไม่ใช่แถวนัดหมาย** เพราะการหาค่าสูงสุดเป็นการรวมค่า
  จึงไม่ล็อกแถวใดแม้เขียน `lockForUpdate` — ล็อกแถวนัดหมายจะไม่กันการจองพร้อมกัน
- **ดัชนีไม่ซ้ำ `uq_appointments_daily_queue` บน `(ศูนย์, วันที่, เลขคิว)** เป็นการป้องกันชั้นสุดท้าย
  ถ้าชน → 409 `"มีการจองพร้อมกัน กรุณาลองใหม่อีกครั้ง"` ให้ผู้ใช้ลองใหม่ ไม่ใช่บอกว่าระบบเสีย

`BookingService::createAppointment()` · `BookingService::lockDailyQueue()`

---

# Part A — ฝั่งผู้รับบริการ

## A1. ค้นหาศูนย์ + ดูบริการ (public)

> ไม่ต้อง login ทุก endpoint ในหัวข้อนี้

### A1.1 ได้หมวดหมู่บริการ (ต้องมีก่อนค้นหา)

**`GET /v1/patient/categories`**
คืนหมวดหมู่ที่ `status=ACTIVE` เรียงตาม `sort_order` — field: `id`, `name`, `icon`
ไม่ paginate · ไม่มีการกรองเพิ่ม

`PatientLocationController::getCategories()`

### A1.2 ได้สิทธิการรักษา (ต้องมีก่อนสมัครสมาชิก)

**`GET /v1/patient/right-types`**
คืนสิทธิที่ `status=ACTIVE` — field: `id`, `code`, `name`, `description`
ไม่ paginate · **ไม่เรียงตามอะไร** (ไม่มี `sort_order`)

> การลงทะเบียนของผู้รับบริการบังคับ `right_type_id` แต่**ไม่ตรวจ `status`**
> จึงส่งสิทธิที่ปิดอยู่ก็ได้ ไม่ผิด 422 (ต่างจากหน้าเคาน์เตอร์ที่บังคับเป็น `ACTIVE`)

`PatientLocationController::getRightTypes()`

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

คืนเฉพาะศูนย์ **`ACTIVE`** ที่ **มีบริการของหมวดหมู้นั้นและ `is_active=true`** และไม่ถูก soft delete
เรียงตาม `distance_km` จริง (สูตร Haversine) คินละ 2 ตำแหน่งทศนิยม
พร้อมบริการของหมวดหมู้นั้นฝังมาให้ → ใช้เลือกศูนย์ได้เลยโดยไม่ต้องเปิด detail

envelope มี `count` ด้วย · บริการที่ฝังมาไม่มี `allow_staff_selection` — ถ้าต้องการรู้ต้องเรียก A1.5

`NearbyCentersRequest::rules()` · `LocationService::getNearbyHealthCenters()`

### A1.4 ดูรายละเอียดศูนย์

**`GET /v1/patient/health-centers/{id}`**
เห็นได้ทั้ง `ACTIVE` และ `CLOSING`; `INACTIVE` → **404**
คืน 11 field: `id`, `code`, `name`, `phone_number`, `subdistrict`, `district`,
`province`, `full_address`, `latitude`, `longitude`, `status`

> endpoint นี้ไม่มี `health_center_status` แยก — ดู `status` ใน `data` ได้เลย
> ต่างจาก A1.5 ที่ยกขึ้นมาไว้ที่ระดับ envelope

`PatientLocationController::getCenterDetail()`

### A1.5 ดูบริการของศูนย์

**`GET /v1/patient/health-centers/{id}/services`**
เห็นได้เฉพาะบริการที่ผ่าน **4 เงื่อนไขพร้อมกัน**:
1. `services.is_active = true`
2. มี pivot `service_time_slot` ≥ 1 แถว (คือมีการตั้งวันเปิดให้บริการแล้ว)
3. `category.status = ACTIVE`
4. ศูนย์เป็น ACTIVE/CLOSING

ศูนย์ `INACTIVE` → **404** (ทั้งรายละเอียดศูนย์และรายการบริการ)
แต่ละรายการคืน: `id`, `name`, `description`, `capacity_type`, **`allow_staff_selection`**, `category`, `capacity`, `time_slots[]`
envelope มี `health_center_status` ด้วย → **เช็คสถานะศูนย์ได้จากที่นี่ที่เดียว**

`time_slots[]` แต่ละรายการคืน `time_slot_id` + `days[]` (1-7)
โดย `days` ถอดจาก bitmask `days_mask` — ไม่ใช่ตารางแยก

> หมายเหตุ: endpoint นี้**ไม่**กรองบริการที่หมอลาหมด (ต่างจาก staff `services?date=`) — ผู้รับบริการจะเห็นบริการแล้วค่อยพบว่าเต็มตอนกดดูช่วงเวลา
>
> หมายเหตุ: บริการที่ soft delete แล้วจะไม่ปรากฏ

`PatientLocationController::getCenterServices()` · `Service::slotsWithDays()`

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

> `right_type_id` ไม่ตรวจ `status` — ต่างจากหน้าเคาน์เตอร์ที่บังคับให้เป็นประเภทที่ยังเปิดใช้
> ถ้าต้องการให้เหมือนกันต้องแก้ที่ `PatientRegisterRequest`

`PatientRegisterRequest::rules()` · `PatientAuthController::register()`

### A2.2 เข้าสู่ระบบ

**`POST /v1/patient/login`** (throttle `patient-login`)
```json
{ "phone_number": "0899876543", "birth_date": "1990-01-01" }
```

**4 ผลลัพธ์ที่ต้องแยกให้ออก:**

| สถานการณ์ | HTTP | คือ | client ต้องทำ |
|---|---|---|---|
| เบอร์ไม่มีในระบบ | **200** | `is_registered: false, token: null` | พาไปหน้าสมัคร |
| บัญชียังถูกล็อกอยู่ | **423** | `"บัญชีถูกระงับชั่วคราว... กรุณารออีก N นาที"` | ปิดปุ่ม login และนับถอยหลัง |
| วันเกิดผิด (ครบ 5 ครั้ง) | **423** | `"กรอกวันเดือนปีเกิดไม่ถูกต้องครบ 5 ครั้ง..."` | ปิดปุ่ม login |
| สำเร็จ | **200** | `is_registered: true, token, patient{id, masked_name, masked_phone}` | เก็บ token |

- วันเกิดผิดครั้งที่ 1-4 → **401** พร้อม `"เหลือโอกาสอีก N ครั้ง"` และ `failed_login_attempts++`
- ครบ 5 ครั้ง → ตั้ง `locked_until = now()+30min` (ทันทีที่ครบครั้งที่ 5 ไม่ต้องรอครั้งที่ 6)
- login สำเร็จ → reset `failed_login_attempts = 0`, `locked_until = null`
- **ตรวจการล็อกก่อนตรวจวันเกิด แต่หลังจากหาเจอผู้รับบริการแล้ว**
  เบอร์ที่ไม่มีในระบบจะไม่มี `locked_until` ให้ตรวจ จึงตอบ 200 `is_registered: false` เสมอ
- **มี 2 ชั้นที่คนละเรื่องกัน**: `throttle:patient-login` 10/นาที (ต่อเบอร์ หรือต่อ IP ถ้าไม่ส่งมา)
  กับตัวนับที่ระดับ `patients` 5 ครั้ง/30 นาที — ชั้นแรกกันการยิงรัว ชั้นหลังกันการเดาเบอร์ทีละเบอร์

`PatientAuthService::authenticate()`

### A2.3 ดูโปรไฟล์ตัวเอง

**`GET /v1/patient/me`** (ต้อง token)
คืนข้อมูลแบบ **masked** (ชื่อ/เบอร์/เลขบัตรถูกปิดบัง) + `rights[]`
อยู่ใน key `data` · ไม่มี endpoint ฝั่งผู้รับบริการที่ unmask ได้

`PatientAuthController::me()` · `PatientResource`

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

> `service_id` ที่ไม่ผ่านเงื่อนไข → **422** พร้อม `errors` ไม่ใช่ 404
> เพราะเป็น validation ระดับ Request ไม่ใช่การค้นแล้วไม่เจอ

`AvailableSlotsRequest::rules()`

**response แบบเปิดเลือกหมอ** — แต่ละ slot มี `staff[]`:
```json
{
  "success": true,
  "service_name": "นวดแผนไทย...",
  "capacity_type": "PER_MASSEUSE",
  "date": "2026-09-25",
  "slots": [{
    "time_slot_id": 1, "time_label": "09:00 - 09:30 น.",
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
| `service_id` | required, `exists:services,id` — **ไม่กรอง `is_active` ที่นี่** เช็คทีหลังใน service (ข้อ 1) |
| `time_slot_id` | required — **ต้องเป็น slot ที่เปิดในวันที่เลือกจริง** (เช็คจาก `service_time_slot` + `days_mask` + `time_slots.is_active`) |
| `appointment_date` | required, `Y-m-d`, **≥ วันนี้** |
| `staff_id` | nullable ใน Request — แต่ถ้าส่งต้องอยู่ใน `service_staff` ของบริการนั้น และ**บังคับต้องมีเมื่อเปิดเลือกหมอ** (ตอน business) |
| `patient_right_id` | optional — ต้องเป็นสิทธิของผู้รับบริการ**คนนี้** (เช็คใน Request แล้ว) |

> ต่างจาก `POST /walk-in`: ฝั่งผู้รับบริการ `patient_right_id` ผูกกับ `patient_id = user.id`
> ส่วนหน้าเคาน์เตอร์ผูกกับ `cid` ที่ส่งมา (ดู B7.3)
>
> `service_id` ที่ **soft delete แล้ว** ผ่าน validation นี้ (rule ตรวจตารางตรง ๆ ไม่เห็น `deleted_at`)
> แต่ `findOrFail` ใน service จะไม่เจอ → **404** ก่อนถึงข้อ 1
> ส่วนบริการที่แค่ `is_active = false` → ผ่าน validation แล้วไปตกที่ข้อ 1 ตามปกติ
>
> ต่างจาก `GET /available-slots` ที่กรอง `is_active` + `deleted_at` ตั้งแต่ใน Request
> ทำให้สอง endpoint นี้ตอบคนละแบบสำหรับ `service_id` ที่ถูกปิดหรือลบแล้ว

`BookAppointmentRequest::rules()`

**ตรวจสอบตามลำดับ** — ทั้งหมดอยู่ใน transaction เดียว

| # | ตรวจอะไร | เมื่อไม่ผ่าน |
|---|---|---|
| 1 | `service.is_active` | `"ขออภัย บริการนี้ปิดให้บริการชั่วคราว"` |
| 2 | ศูนย์ `isBookable()` = ACTIVE | `"ศูนย์ปิดรับการจองใหม่"` |
| 3 | `time_slot` เปิดในวันนั้น | `"บริการนี้ไม่เปิดให้บริการในวันที่เลือก"` |
| 4 | ช่วงเวลาสิ้นสุดยังไม่ผ่านมาแล้ว (**เฉพาะการจองเอง**) | `"ช่วงเวลานี้เลยกำหนดแล้ว กรุณาเลือกช่วงเวลาอื่น"` |
| 5 | `StaffLeave::isServiceAvailable` (บุคลากรไม่ลาครบทั้งหมด) | `"บริการนี้ปิดชั่วคราว เนื่องจากเจ้าหน้าที่ทั้งหมดลางานในวันนี้"` |
| 6 | ไม่จอง service เดิมซ้ำวันเดิม (CONFIRMED) | `"ท่านมีรายการจองบริการนี้ในวันที่เลือกเรียบร้อยแล้ว"` |
| 7 | **เปิดให้เลือกบุคลากร** → `staff_id` ต้องมี | `"กรุณาเลือกเจ้าหน้าที่/หมอที่ต้องการรับบริการ"` |
| 8 | `StaffLeave::isStaffAvailableForService` | `"ไม่สามารถเลือกเจ้าหน้าที่นี้ได้ในวันที่เลือก (ไม่ว่างหรือไม่ได้ให้บริการนี้)"` |
| 9 | `CapacityService::isStaffSlotAvailable` | `"เจ้าหน้าที่ที่ท่านเลือกเต็มคิวในรอบเวลานี้แล้ว..."` |
| — | *กรณีไม่เปิดให้เลือกบุคลากร* → นับคิวแล้วเทียบโควตา | `"ขออภัย คิวในรอบเวลานี้เต็มแล้ว กรุณาเลือกรอบเวลาอื่น"` |
| 10 | สิทธิการรักษาที่แนบยังไม่หมดอายุ | สิทธิหมดอายุแล้ว (ดูหัวข้อ 0.6) |
| 11 | ออกเลขคิว = `max+1` ของ (ศูนย์, วันนั้น) + ดัชนีกันซ้ำ | ชนกัน → 409 ให้ลองใหม่ |

- **ข้อ 5 ใช้กับทุกทางเข้า** ทั้งจองเองและหน้าเคาน์เตอร์ — เป็นกติกาเดียวกัน
- ข้อ 4 เป็นของ `bookedByType === 'SELF'` เท่านั้น (ดู B7.3)
- ข้อ 9 ใช้ `CapacityService::PER_STAFF_PER_SLOT_LIMIT` (ดู 0.4)
- ข้อ 10 ใช้กับทุกทางเข้า ทั้งจองเองและหน้าเคาน์เตอร์
- **ข้อความ error ของ `POST /v1/patient/book` ถูกส่งตรงให้ผู้รับบริการ** (ดูหัวข้อ 0.6)
- ⚠️ ข้อ 11 ล็อกที่แถวศูนย์สุขภาพ ไม่ใช่แถวนัดหมาย (ดู 0.7) — เพราะการหา `max()` ไม่ล็อกแถวใด
- ⚠️ กรณีไม่เปิดให้เลือกบุคลากร: `lockForUpdate` อยู่บน query ที่นับคิว
  ซึ่งเป็นการรวมค่าเหมือนกัน จึงไม่ได้ล็อกแถวใดเช่นกัน (ต่างจาก `reassign-staff` ที่ล็อกทั้งสองแถว)

**หมายเหตุเรื่องลำดับ:** ข้อ 5 มาก่อนข้อ 6
ถ้าบุคลากรลาครบแต่ผู้รับบริการเคยจองบริการนั้นวันเดียวกัน จะได้ข้อความ "บริการปิดชั่วคราว"
ไม่ใช่ "ท่านมีรายการซ้ำแล้ว" — ตรวจจากข้อความที่ได้จริง ไม่ควรเดาจากข้อมูลที่ส่งมา

**หมายเหตุเรื่องลำดับ:** ข้อ 5 (`isServiceAvailable`) มาก่อนข้อ 6 (กันจองซ้ำ)
ถ้าหมอลาครบแต่ผู้รับบริการเคยจองบริการนั้นวันเดียวกัน จะได้ข้อความ "บริการปิดชั่วคราว" ไม่ใช่ "ท่านมีรายการซ้ำแล้ว"
ตรวจจากข้อความที่ได้จริง ไม่ควรเดาจากข้อมูลที่ส่งมา

**Response 201:**
```json
{
  "success": true, "message": "จองคิวสำเร็จ",
  "data": {
    "appointment_id": 1, "queue_no": "Q-001",
    "health_center": "...", "service_name": "...",
    "appointment_date": "2026-09-25", "time_slot": "09:00 - 09:30 น.",
    "staff": { "id": 1, "name": "...", "position": "หมอนวด" }
  }
}
```
`time_slot` คือค่า `label` ของช่วงเวลา · `staff` เป็น `null` เมื่อบริการไม่เปิดให้เลือกหมอ (A4)
`PatientBookingController::book()`

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
| `max_capacity` ที่ขั้น 3 | จำนวนหมอที่เลือกได้และยังให้บริการ | ตาม `capacity_type` (ดู 0.4) |
| `available_count` ที่ขั้น 3 | จำนวนหมอที่ **ยังว่าง** ในรอบนั้น | โควตาที่เหลือของรอบ (หรือทั้งวันถ้า `PER_DAY`) |
| การนับตอนจอง | ต่อหมอ (1 คิว/รอบ/คน) | `lockForUpdate` แล้วเทียบกับโควตา |
| response `staff` | คืนหมอที่เลือก | คืน `null` → `"เจ้าหน้าที่": "ระบบจัดสรร"` ใน Discord |

**ขั้นตอนเดียวกันเป๊ะ** — A1.1 → A1.5 → ขั้น 3 → login/register → `POST /book` → A5

ผลลัพธ์ขั้น 3 แบบไม่เลือกหมอ — envelope เหมือนกัน มี `service_name` / `capacity_type` / `date` ด้วย
```json
{
  "slots": [{
    "time_slot_id": 1, "time_label": "09:00 - 09:30 น.",
    "start_time": "09:00", "end_time": "09:30",
    "max_capacity": 1, "available_count": 1, "is_full": false
  }]
}
```

**`PER_DAY` ต่างอีก:** ค่า `available_count` คือ **ทั้งวันรวมกัน** ไม่ใช่รายรอบ — ถ้าเต็ม 1 คิว รอบอื่นของวันเดียวกันจะเป็น `available_count: 0` ด้วย

`CapacityService::countBookedByTimeSlot()`

---

## A5. ประวัติการจอง + ยกเลิก

### A5.1 ดูประวัติ

**`GET /v1/patient/appointments?per_page=20`**
- คืนเฉพาะของตัวเอง เรียงวันที่มาก→น้อย แล้ว id มาก→น้อย
- `per_page` ถูก clamp ระหว่าง 1-100 (ส่งเกินจะถูกตัด ไม่ error)
- envelope มี `meta: { current_page, last_page, per_page, total }`
- **ไม่มีตัวกรอง** — ดูทุกสถานะรวมทั้งที่ยกเลิกแล้ว

`PatientHistoryController::index()`

### A5.2 ดูรายละเอียด 1 ใบ

**`GET /v1/patient/appointments/{id}`**
คืน key `ticket` (ไม่ใช่ `data`) — ของผู้รับบริการอื่น → 404
`PatientHistoryController::show()`

### A5.3 ยกเลิก

**`PATCH /v1/patient/appointments/{id}/cancel`**
```json
{ "reason": "ติดธุระ" }   // optional, max 500
```

**ยกเลิกได้เฉพาะเมื่อ 2 เงื่อนไขพร้อมกัน:**
1. `status = CONFIRMED`
2. เป็นวันอนาคต **หรือ** เป็นวันนี้แต่ยังไม่เลยเวลาสิ้นสุดของรอบ (`timeSlot.end_time > เวลาปัจจุบัน`)

ไม่ผ่านเงื่อนไขใดข้อ → **404** (ไม่ใช่ 422 — เพราะ query ถูกจำกัดเงื่อนไขไว้ก่อน `findOrFail`)

- `reason` ไม่ส่ง → default `"ผู้รับบริการขอยกเลิกเองผ่านระบบ"`
- ถ้าศูนย์เป็น CLOSING → dispatch `CloseHealthCenterJob` ลองปิดศูนย์อัตโนมัติ
- **ยกเลิกแล้วย้อนกลับไม่ได้** — `VALID_TRANSITIONS` ไม่มีทางออกจาก CANCELLED

`PatientHistoryController::cancel()` · `CancelAppointmentRequest::rules()`

---

# Part B — ฝั่งเจ้าหน้าที่

> ทุก endpoint ใน Part B ต้องมี Staff token และผ่าน role gate `STAFF, HEALTH_CENTER_ADMIN, SUPER_ADMIN`
> **ระลึกกฎ `health_center_id` ในหัวข้อ 0.3 ทุกครั้ง**
>
> ⚠️ **กลุ่มที่ต้องเลื่อนไปดู `routes/api.php` เสมอ** — ไม่ใช่ทุก endpoint อยู่ใน group เดียวกัน:
>
> | group | ใครเข้าถึงได้ | มีอะไรบ้าง |
> |---|---|---|
> | `role:STAFF,HC_ADMIN,SUPER_ADMIN` | ทุกบทบาท | dashboard, appointments, roster, leaves, services CRUD, **patients/{id}/contact** |
> | `role:SUPER_ADMIN,HC_ADMIN` | ผู้ดูแลขึ้นไป | admin/users, admin/roles, **walk-in**, **patients/{id}** (แก้เต็ม), health-centers, time-slot-days, assignees, service/staff |
> | `role:SUPER_ADMIN` | ผู้ดูแลระบบเท่านั้น | discord webhook, admin/health-centers, สร้าง/แก้/ลบ time slot |
> | ต่อ route | ระบุเอง | `GET /health-centers` (dropdown), `POST /leaves` เป็น STAFF ได้ |
>
> เอกสารนี้ระบุ role ไว้ทุก endpoint แต่ถ้าขัดแย้ง ให้เชื่อ `routes/api.php`

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
- ชื่อผู้ใช้/รหัสผิด → **422** (ไม่ใช่ 401) ด้วยข้อความเดียวกันทั้งสองกรณี เพื่อไม่ให้บอกว่าชื่อถูกแต่รหัสผิด
- **`status != ACTIVE` ถูกมองเหมือนไม่มีบัญชี** → 422 ข้อความเดียวกัน ไม่ใช่ 403 (สอดคล้องกับหัวข้อ 0.6 ตรงกันว่าไม่เผยว่าชื่อผู้ใช้นี้มีจริงหรือไม่)
- บัญชีที่ไม่มีศูนย์ (แต่ไม่ใช่ SUPER_ADMIN) → **403** `"ไม่พบข้อมูลศูนย์สุขภาพ ต้นสังกัด กรุณาติดต่อผู้ดูแลระบบ"`
- ศูนย์ `INACTIVE` → **403** `"ศูนย์สุขภาพ ต้นสังกัดถูกระงับการใช้งานชั่วคราว"`
- ศูนย์ `CLOSING` → **ยัง login ได้** เพราะต้องจัดการคิวที่เหลือ (เช็คเป็นรายการ `ACTIVE`/`CLOSING` ไม่ใช่แค่ `ACTIVE`)
- `health_center` ใน response เป็น `null` สำหรับ SUPER_ADMIN — ต้องเลือกศูนย์เองทุกครั้งที่ส่งคำขอแบบเจาะจง
- `role` ใน response คือ role แรกเท่านั้น (`roles->first()`) ถ้ามีหลาย role จะเห็นแค่ตัวเดียว
- token อายุ 24 ชม. (Sanctum `expiration => 1440`)

**`throttle.staff` — ต่างจาก limiter ปกติ:**

| จุด | พฤติกรรม |
|---|---|
| นับทีละเลข 5 ครั้ง / 30 นาที | ต่อ **คู่ (username + IP)** — ไม่ใช่แค่ IP อย่างเดียว |
| นับ **ทุกครั้ง** ที่มีการพยายาม | รวมถึงกรณีรหัสผิดที่ตอบ 422 (เดิมนับเฉพาะ 401/403 ทำให้ตัวนับถูกล้างทุกครั้ง) |
| ล้างตัวนับเมื่อ login สำเร็จ | `RateLimiter::clear` — ล็อกจะหายทันทีไม่ต้องรอ 30 นาที |
| เกินลิมิต → **429** | พร้อม `"ล็อกอินหลายครั้งเกินไป กรุณารอ N วินาที"` |

> คู่ (username + IP) หมายถึงผู้โจมตีที่เปลี่ยน IP ได้ลิมิตใหม่ต่อชื่อผู้ใช้เดิม
> ชั้นนี้จึงกันการยิงรัวจากเครื่องเดียว ไม่ใช่การกันการเดารหัสผ่านแบบกระจาย

`ThrottleStaffLogin::handle()` · ดู B2.3 เรื่อง limiter รวม

`StaffAuthController::login()` · `StaffLoginRequest::rules()`

**2. `GET /v1/staff/me`** — ข้อมูลตัวเอง + `roles[]` + `health_center` เต็มรูปแบบ
> สำหรับ STAFF/HC_ADMIN ค่านี้แทน `GET /v1/staff/health-centers` ได้ (ซึ่งคืนแค่ศูนย์ตัวเอง 1 แถว)

**3. `PATCH /v1/staff/me`** — แก้ชื่อ / เปลี่ยนรหัสผ่าน
```json
{ "name": "ชื่อใหม่" }
```
```json
{ "current_password": "เดิม", "password": "ใหม่อย่างน้อย 6 ตัว", "password_confirmation": "ใหม่อย่างน้อย 6 ตัว" }
```

เปลี่ยนรหัสผ่าน **ต้องส่ง `current_password` ที่ถูกต้อง** เสมอ

> ⚠️ **การเปลี่ยนรหัสผ่านตัวเองไม่ตัด token เดิมทิ้ง** — session อื่นยังใช้ได้
> ต่างจาก `PATCH /admin/users/{id}` ที่ตัดทั้งหมด (ดู B12.4)
> ถ้าต้องการให้ถอด session เมื่อเปลี่ยนรหัสผ่านต้องเพิ่มเอง

`UpdateProfileRequest::rules()` · `StaffAuthController::updateProfile()`

**4. `POST /v1/staff/logout`** — ลบ token ปัจจุบันเท่านั้น (token อื่นยังใช้ได้)
> ถ้าต้องการถอดทุก session ต้องเรียก revoke ที่อื่น

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
- **`roster` มี `effective_status` เสมอ** — ต่างจาก `GET /staff/roster` ที่ต้องส่ง `?date=` จึงจะมี
  เพราะ bundle ใส่วันให้เสมอ (ถ้าไม่ส่ง = วันนี้)
- **endpoint เดิมทั้งสามยังทำงานเหมือนเดิม** — ย้ายมาใช้ bundle ทีละหน้าได้

`StaffDashboardController::bundle()`

### B2.3 โควตาการเรียก API (Rate Limit)

**ทุก endpoint ใน `routes/api.php` ถูกจำกัดอัตราโดยอัตโนมัติ** ผ่าน middleware group `api`

| ผู้เรียก | ต่อนาที |
|---|---|
| เจ้าหน้าที่ปฏิบัติงาน · ผู้ดูแลศูนย์ · ผู้ดูแลระบบ | **240** |
| ผู้รับบริการ · ยังไม่ยืนยันตัวตน | **60** |

- **นับแยกตามบัญชี ไม่ใช่ตาม IP** — ผู้เรียกคนเดียวที่ยิงหนักไม่กระทบคนอื่น
  อ่านเจ้าของโทเคนจาก header โดยตรง เพราะตัว limiter ทำงานก่อน auth
  และแยกชนิดบัญชีด้วย morph class เพราะเจ้าหน้าที่กับผู้รับบริการเก็บคนละตาราง เลขบัญชีจึงซ้ำกันได้
- **โทเคนที่ไม่รู้จักหรือหมดอายุ → นับเป็น `ip:` ที่โควตา 60** เหมือนยังไม่ยืนยันตัวตน
  ตัวจำกัดอัตราไม่ตรวจวันหมดอายุซ้ำ เพราะการยืนยันตัวตนจะตรวจอีกครั้งอยู่แล้ว
- **เจ้าหน้าที่ที่ถูกถอด role ออกแล้ว → ตกไปใช้โควตา 60** แม้ยังมีโทเคนที่ยังไม่หมดอายุ
  เพราะโควตาขึ้นกับ role ในฐานข้อมูลขณะนั้น ไม่ใช่ตาม ability ที่แนบในโทเคน
- **เก็บตัวนับใน Redis** (`CACHE_STORE=redis`) จึงนับรวมทุก instance
- เป็น **fixed window** — ขอบเขตแรงสุดคือโควตาเต็มในช่วงเวลาใดช่วงหนึ่ง แล้วเป็นศูนย์ทันทีเมื่อข้ามเส้น
  ถ้าหน้าจอยิงแน่นตอนเปิดจนชนเพดาน ผู้ใช้จะเจอ 429 ทันทีและต้องรอข้ามเส้น
- **ไม่มีโควตาแยกตามบทบาท** — ผู้ดูแลระบบที่ยิงรายงานหนักกินโควตาเดียวกับเจ้าหน้าที่

**ลิมิตเฉพาะทางที่เข้มกว่านี้** (ทำงานทับโควตารวม):

| ลิมิต | ค่า | ใช้กับ |
|---|---|---|
| `throttle:unmask` | 10/นาที | `POST /staff/appointments/{id}/unmask` (PDPA) |
| `throttle:patient-login` | 10/นาที **ต่อเบอร์โทร** | `POST /patient/login` |
| `throttle:patient-register` | 5/นาที ต่อ IP | `POST /patient/register` |
| `throttle.staff` | 5 ครั้ง / 30 นาที | `POST /staff/login` (ดู B1) |

> `patient-login` นับต่อเบอร์โทร ไม่ใช่ต่อ IP เพราะต้องกันการเดารหัสผ่านการลองเบอร์ทีละเบอร์
> ถ้าใช้ IP ผู้โจมตีเปลี่ยน IP แล้วเดาได้ไม่จำกัด
> ไม่ส่ง `phone_number` มาเลย → ตกไปนับที่ IP แทน
>
> ⚠️ **`throttle:unmask` (10/นาที) เข้มกว่าโควตารวมของเจ้าหน้าที่ (240/นาที) เสมอ**
> ถ้าหน้าจอเปิดดูข้อมูลผู้รับบริการถี่จะโดน 429 ที่นี่ก่อนโควตารวมเสมอ

`AppServiceProvider::boot()` — ค่าอยู่ในค่าคงที่ `STAFF_RATE_LIMIT` / `PUBLIC_RATE_LIMIT` ต้องแก้ที่นี่ที่เดียว

---

## B3. จัดการบริการ

### B3.1 ดูรายการบริการ

**`GET /v1/staff/services`**

| query | ผล |
|---|---|
| — | HC_ADMIN/STAFF = ศูนย์ตัวเอง; SUPER_ADMIN = ทุกศูนย์ |
| `?health_center_id=` | SUPER_ADMIN กรองศูนย์ |
| `?date=YYYY-MM-DD` | **ซ่อนบริการที่หมอทั้งหมดลาในวันนั้น** (เฉพาะเมื่อมี center scope ชัดเจน) |

**เปิดให้ STAFF ทั่วไปเรียกได้** — ไม่ถูกจำกัดที่บทบาทบริหาร

`StaffServiceController::index()` — การกรองใช้ `StaffLeave::isServiceAvailable()` ตัวเดียวกับ booking

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
| `allow_staff_selection` | **ส่งได้แค่ `false`/`0`** — ส่ง `true` → 422 พร้อมข้อความให้ไปเปิดทีหลัง |

**ผลลัพธ์:** สร้าง `services` (บังคับ `allow_staff_selection=false`, `is_active=true`) + `service_capacity` (default 1/10) + `service_time_slot` ใน transaction เดียว

> ⚠️ `POST /services` เปิดให้ **STAFF ทั่วไป** สร้างบริการได้
> ถ้าต้องการจำกัดเฉพาะผู้ดูแล ต้องย้าย route ไป group `role:SUPER_ADMIN,HEALTH_CENTER_ADMIN`
> ตอนนี้ STAFF สร้างบริการในศูนย์ตัวเองได้โดยไม่ต้องได้รับมอบหมาย

`CreateServiceRequest::rules()` · `StaffServiceManagementService::createService()`

### B3.3 แก้ไขบริการ

**`PUT /v1/staff/services/{id}`** — ส่งเฉพาะ field ที่ต้องการแก้

| field | rule |
|---|---|
| `allow_staff_selection: true` | ได้เฉพาะ `PER_MASSEUSE` **และ** มีหมอใน `service_staff` ≥ 1 คน — ไม่งั้น 422 |
| `allow_staff_selection: false` | 422 ถ้ายังมีคิว CONFIRMED อนาคตของบริการนี้ |
| `max_per_slot` / `max_per_day` | integer ≥ 1, `updateOrCreate` ค่าเดิมไว้ถ้าไม่ส่ง |
| `time_slots` | ถ้าส่ง = แทนที่ชุดเดิมทั้งหมด (ดู B3.5) |

> การปิด `allow_staff_selection` เช็คคิวค้างของ **ทั้งบริการ** ไม่ใช่เฉพาะคิวที่ผูกหมอไว้

`UpdateServiceRequest::rules()` · `StaffServiceManagementService::updateService()`

### B3.4 เปิด/ปิดบริการ + ลบ

**`PATCH /v1/staff/services/{id}/toggle-status`** — สลับ `is_active` (ACTIVE/ปิดชั่วคราว)
**STAFF ทั่วไปเรียกได้** — ปิดบริการที่ศูนย์ตัวเองได้

**`DELETE /v1/staff/services/{id}`** — **HC_ADMIN/SUPER_ADMIN เท่านั้น** (STAFF → 403 `"ไม่มีสิทธิ์ลบบริการ"`)
→ 422 ถ้ามีคิว CONFIRMED ตั้งแต่วันนี้ขึ้นไป: `"ไม่สามารถลบบริการได้ เนื่องจากมีคิวล่วงหน้าอยู่ N คิว กรุณาจัดการคิวก่อน"`
→ **soft delete** — บริการหายจาก `GET /staff/services` และจาก `A1.5` ทันที ทั้งที่ยังมีนัดหมายเก่าอยู่
→ มีนิยามคิวค้างตัวเดียวกันทั้งระบบ ดู C.2

**ข้อควรระวังเรื่อง scope ของการลบ:** ตรวจบทบาทก่อนแตะศูนย์
STAFF ที่ไม่มีสิทธิ์จะได้ 403 แม้ส่ง `health_center_id` ผิดศูนย์มาด้วย
(ต่างจาก endpoint ที่ตรวจศูนย์ก่อน แล้วจึงตรวจสิทธิ์ทีหลัง)

`StaffServiceController::destroy()` · `StaffServiceManagementService::deleteService()`

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
- **ช่วงเวลาที่เลือกต้องเป็นของศูนย์เดียวกับบริการ** → ต่างศูนย์ = 422 `"ช่วงเวลาที่เลือกไม่ได้อยู่ในศูนย์สุขภาพเดียวกับบริการ"` บังคับไว้ 2 ชั้น คือตรวจในโค้ด และ foreign key แบบ composite ในฐานข้อมูล
  ตารางเชื่อมมีคอลัมน์ศูนย์ของตัวเอง ต้องตรงกับทั้งบริการและช่วงเวลา — ถ้าเพิ่มจุดเขียนใหม่ต้องส่งศูนย์มาด้วย ไม่งั้นจะถูกฐานข้อมูลปฏิเสธ
- **ช่วงเวลาต้องเปิดใช้งานอยู่** → ช่วงที่ถูกปิดใช้งานอยู่ → 422 `"ช่วงเวลาไม่ถูกต้องหรือไม่เปิดใช้งาน"`
  ต้องเปิดช่วงเวลาให้ใช้งานก่อนจึงจะตั้งวันให้บริการได้ (ลำดับคือ สร้าง/เปิดช่วงเวลา → ตั้งวันให้บริการ → ผูกหมอ)

`OperatingDayService::syncDays()` · `OperatingDayService::validateSlotDays()` · `StaffServiceController::syncTimeSlotDays()`

---

## B4. เปิดเลือกหมอ (สำคัญ — ต้องทำตามลำดับ)

> ⚠️ **ลำดับห้ามสลับ** — ถ้าเปิด flag ก่อนผูกหมอจะ 422
> และ **สร้างบริการตอน `POST /services` เปิด flag ไม่ได้เลย** — ต้องแก้ทีหลังเท่านั้น

### ขั้นที่ 1 — ดูหมอที่จะผูก

**`GET /v1/staff/roster`** (B8.1) → ได้ `staff.id` พร้อมหมวดหมู่
> หมอที่ผูกได้ต้องเป็น **หมวดหมู่เดียวกับบริการ** (เช่น บริการหมวด "แพทย์แผนไทย" ต้องผูกหมอหมวดนั้น)
> **บริการที่ไม่มี `category_id` จะไม่ตรวจหมวดหมู่** — ผูกหมออะไรก็ผ่าน

### ขั้นที่ 2 — ผูกหมอกับบริการ

**`PUT /v1/staff/services/{id}/staff`** — HC_ADMIN / SUPER_ADMIN (STAFF → 403)
```json
{ "health_center_id": 1, "staff_ids": [1, 2] }   // health_center_id required เฉพาะ SUPER_ADMIN
```

| กติกา | ผลลัพธ์ |
|---|---|
| `staff_ids` ต้องมี key เสมอ (`present`) | ไม่ส่ง → 422 `"กรุณาระบุ staff_ids (ส่ง [] เพื่อยกเลิกทั้งหมด)"` · ส่ง `[]` = ยกเลิกทั้งหมด |
| หมอต้องมีอยู่จริง | 422 `"ไม่พบเจ้าหน้าที่ที่ระบุในระบบ"` |
| หมอต้องอยู่ศูนย์เดียวกัน | 422 `"เจ้าหน้าที่ X ไม่สังกัด ศูนย์สุขภาพนี้"` |
| หมอต้องอยู่หมวดหมู่เดียวกับบริการ | 422 `"หมอ X ไม่อยู่ในหมวดหมู่เดียวกับบริการนี้"` |
| **การถอดหมอออก** ที่มีคิว CONFIRMED อนาคต | 422 `"ไม่สามารถแกะเจ้าหน้าที่ออกจากบริการได้..."` |

- **ตรวจการถอดทีละคน** เฉพาะคนที่กำลังจะถูกถอด — คนที่ยังอยู่ไม่ถูกตรวจเรื่องคิวค้าง
- ทั้งหมดอยู่ในธุรกรรมเดียว · เขียน `AuditLog` action `SERVICE_SELECTABLE_STAFF_SYNC` (before/after)

`SyncServiceStaffRequest::rules()` · `StaffServiceController::syncStaff()`

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

**สิ่งที่มักเข้าใจผิดในหัวข้อนี้:**

- การผูกหมอ = `service_staff` → **หมอที่ผู้รับบริการเลือกได้**
- การมอบหมายให้ดูแลคิว = `service_user` → **ใครมีสิทธิ์เห็น/แตะคิว** (B5)
  **เป็นคนละเรื่องกัน** — การเปิด flag ไม่ได้ทำให้ได้ `service_user` โดยอัตโนมัติ
- เปิด flag แล้วโควตานับจาก `service_staff` ที่ยังให้บริการอยู่
  ต่างจาก `PER_MASSEUSE` แบบไม่เปิด flag ที่นับจาก `service_user` (ดู 0.4) — **สองคนละชุดนับ**

---

## B5. มอบหมายผู้รับผิดชอบบริการ

**`PUT /v1/staff/services/{id}/assignees`** — HC_ADMIN / SUPER_ADMIN (STAFF → 403)
```json
{ "health_center_id": 1, "user_ids": [10, 11] }   // [] = ยกเลิกทั้งหมด
```

**นิยาม:** `user_ids` คือ **บัญชี User ที่มี role STAFF** ไม่ใช่ staff profile — ใช้คุมว่า STAFF คนนั้นเห็น/จัดการคิวของบริการนี้ได้

| กติกา | ผลลัพธ์ |
|---|---|
| `user_ids` ต้องมี key เสมอ (`present`) | ไม่ส่ง → 422 `"กรุณาระบุ user_ids (ส่ง [] เพื่อยกเลิกทั้งหมด)"` · ส่ง `[]` = ยกเลิกทั้งหมด |
| ผู้ใช้ต้องมีอยู่จริง | 422 `"ไม่พบผู้ใช้ที่ระบุ"` |
| ผู้ใช้ต้องอยู่ศูนย์เดียวกัน | 422 `"ผู้ใช้ X ไม่สังกัด ศูนย์สุขภาพนี้"` |
| ผู้ใช้ต้องมี role STAFF | 422 `"ผู้ใช้ X ไม่มีบทบาท STAFF"` |

> **ตรวจคนที่ถูกระบุทุกคนก่อนเขียน** — ถ้าคนที่ 5 จาก 10 ผิดเงื่อนไข ก็ไม่มีการเขียนค่าใด ๆ
> ต่างจาก B4 ที่ตรวจการถอดหมอทีละคน

**ไม่มีเงื่อนไขคิวค้าง** — ถอดผู้รับผิดชอบออกได้เลยแม้ยังมีคิวยืนยันอยู่
เพราะการถอดผู้มอบหมายไม่ทำให้คิวหาย แต่เป็นการถอด "สิทธิ์เข้าถึง" ไม่ใช่การเปลี่ยนนัดหมาย
(ต่างจากการถอดหมอใน B4 ที่กระทบการเลือกหมอของผู้รับบริการ)

**ไม่เขียน `AuditLog`** — เป็นการเปลี่ยนการตั้งค่าความปลอดภัยระดับบริการ ไม่ใช่ข้อมูลผู้รับบริการ

**ผลข้างเคียง — นี่คือขอบเขตที่ใช้คุมทุกทางที่ดูหรือแตะคิว (ดู 0.3a):**

STAFF ที่ไม่ได้ถูก assign บริการนั้น → **403** ในทุกเส้นทางต่อไปนี้

| endpoint | ผล |
|---|---|
| `GET /staff/appointments` | เห็นเฉพาะคิวบริการที่ assign · filter `service_id` ที่ไม่ได้ assign → 403 |
| `GET /staff/dashboard/summary` | นับเฉพาะบริการที่ assign |
| `GET /staff/dashboard/bundle` | เหมือนข้างบน (แต่ `roster` ยังทั้งศูนย์) |
| `PATCH /staff/appointments/{id}/status` | 403 — เว้นแต่เขาเป็นผู้ถูกระบุในคิวนั้น |
| `PATCH /staff/appointments/{id}/reassign-staff` | 403 — เว้นแต่เป็นผู้ถูกระบุในคิวนั้น |
| `PATCH /staff/appointments/{id}/reschedule` | เหมือนข้างบน (ดู B7.6) |
| `POST /staff/appointments/{id}/unmask` | 403 + **ไม่เขียน audit log** |
| `PATCH /staff/patients/{id}/contact` | 403 ถ้าผู้รับบริการไม่เคยมีคิวในบริการที่ assign |
| `POST /staff/appointments/walk-in` | 403 ทุกกรณี — เป็นหน้าที่ผู้ดูแลศูนย์อยู่แล้ว ไม่เกี่ยวกับการ assign |

`SyncAssigneesRequest::rules()` · `StaffServiceController::syncAssignees()` · `ResolvesHealthCenterScope::denyIfOutOfServiceScope()`

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
| `PATCH` | `/v1/staff/admin/time-slots/{id}/toggle-status` | HC_ADMIN, SUPER_ADMIN | ข้ามศูนย์ไม่ได้ (404) · **ปิดไม่ได้ถ้ายังมีคิวค้าง → 422** |
| `POST` | `/v1/staff/admin/time-slots` | **SUPER_ADMIN เท่านั้น** | บังคับ `health_center_id` |
| `PUT` | `/v1/staff/admin/time-slots/{id}` | **SUPER_ADMIN เท่านั้น** | บังคับ `health_center_id`; slot ต้องเป็นของศูนย์นั้น · ปิด (`is_active: false`) ไม่ได้ถ้ายังมีคิวค้าง |
| `DELETE` | `/v1/staff/admin/time-slots/{id}` | **SUPER_ADMIN เท่านั้น** | **ไม่รองรับ** — 422 ทุกกรณี แม้จะไม่มีอะไรให้ลบ |

**สร้าง:**
```json
{ "health_center_id": 1, "start_time": "09:00", "end_time": "09:30", "is_active": true }
```
- เวลา format `HH:MM` (`H:i`), `end_time` ต้องมากกว่า `start_time`
- **ช่วงเวลาที่ทับกับช่วงเดิมของศูนย์เดียวกัน → 422** `"มีช่วงเวลาที่ทับซ้อนกันอยู่แล้วในศูนย์นี้"`
  ตรวจการทับจริง (ไม่ใช่เทียบเวลาให้ตรงกันทุกตัว) ช่วงที่ต่อกันพอดี เช่น 09:00-09:30 กับ 09:30-10:00 ไม่ถือว่าทับ
- ทับกับช่วงเวลาของศูนย์อื่น → สร้างได้ (แต่ไปผูกกับบริการข้ามศูนย์ไม่ได้ ดู B3.5)
- **`label` และ `sort_order` ถูกสร้างอัตโนมัติ ส่งมาเองไม่ได้**
  - `label` = `"09:00 - 09:30 น."` (มีช่องว่างรอบขีดและต่อท้ายด้วย `น.`)
  - `sort_order` = นาทีนับจากเที่ยงคืน ทำให้เรียงตามเวลาเริ่มอัตโนมัติ
- `is_active` ส่งมาได้ ถ้าไม่ส่งค่าเริ่มต้นคือเปิดใช้งาน
- **แก้เวลาแล้ว `label` / `sort_order` ถูกสร้างใหม่ให้อัตโนมัติเช่นกัน**

> `time_label` ใน response ที่ผู้รับบริการเห็นคือค่า `label` นี้ จึงมีรูปแบบ `"09:00 - 09:30 น."`
> ไม่ใช่ `"09:00-09:30"` ตามที่มักเขียนในตัวอย่าง

**การตอบกลับ** — ทุกช่วงเวลาที่คืนมาระบุ `health_center` ของตัวเอง (เฉพาะ endpoint นี้ ที่อื่นที่ฝังช่วงเวลาไว้ เช่น นัดหมายของผู้รับบริการ จะไม่มีข้อมูลนี้)

**ลบ** → ไม่รองรับ 422 `"ไม่สามารถลบช่วงเวลาได้ กรุณาเปลี่ยนสถานะเป็นปิดใช้งานแทน"`
เหตุผลตาม ADR-0001: ช่วงเวลาถูกนัดหมายและถูกกำหนดวันให้บริการอ้างอิงอยู่ การลบจะทำให้ประวัตินัดหมายหายหรือวันให้บริการหายเงียบ ๆ

**ปิดใช้งานแทน** → เมื่อปิด ช่วงเวลาหายจากผู้รับบริการทันทีทุกบริการของศูนย์
แต่ **วันให้บริการที่ตั้งไว้ (`service_time_slot`) ยังอยู่ครบ** และกลับมาใช้ทันทีเมื่อเปิดคืน

> ⚠️ **ปิดช่วงเวลาที่ยังมีคิวค้างไม่ได้ → 422**
> ใช้เกณฑ์คิวค้างตัวเดียวกับทั้งระบบ (`status = CONFIRMED` และ `appointment_date >= วันนี้`)
> เหตุผลเดียวกับที่ลบบริการไม่ได้ — ปิดแล้วนัดหมายที่ผู้รับบริการถือไว้จะถูกทิ้งโดยไม่มีคำเตือน

`StaffTimeSlotController` · `StaffTimeSlotManagementService`

> ✅ **ข้อยกเว้นที่เคยเป็น bug แก้แล้ว:** controller นี้เคยไม่ใช้ `ResolvesHealthCenterScope` — SUPER_ADMIN ไม่ส่ง `health_center_id` ได้สร้างช่วงเวลาที่ไม่ผูกศูนย่อจริง และแก้/ลบข้ามศูนย์ได้ ตอนนี้ทุก endpoint ใช้ตัวแก้ scope กลางเหมือนที่เหลือ และฐานข้อมูลบังคับศูนย์ไม่ให้ว่างด้วย

ดูรายละเอียดของการแก้ครั้งนั้นได้ที่หัวข้อ **C.1** ท้ายเอกสาร

---

## B7. โต๊ะเคาน์เตอร์คิว

### B7.1 ดูคิวของวัน

**`GET /v1/staff/appointments`**

| query | ผล |
|---|---|
| `date` | `Y-m-d`, default วันนี้ · **ไม่จำกัดวัน** — ดูคิวล่วงหน้าได้ (คิวที่หมอจองให้ไว้ล่วงหน้า) |
| `service_id` | กรองรายบริการ (`exists:services,id`) |
| `status` | `CONFIRMED`/`COMPLETED`/`CANCELLED`/`NO_SHOW` |
| `health_center_id` | SUPER_ADMIN เท่านั้น; ไม่ส่ง = ทุกศูนย์ |

- **STAFF (ไม่ใช่ admin)** → เห็น**เฉพาะ**บริการที่ถูก assign ให้ตัวเอง และถ้า query `service_id` ที่ไม่ได้ assign → **403**
- ไม่ paginate — คืนทั้งวันที่เป็นรายการ (`data` + `date`)
- ข้อมูลผู้รับบริการถูก **masked** เสมอ — `patient.masked_name`, `patient.masked_phone`

`StaffAppointmentController::index()` · `StaffAppointmentFilterRequest::rules()` · `StaffAppointmentService::getDailyAppointments()`

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
- **ไม่เขียน audit log** — เป็นการเปลี่ยนสถานะคิว ไม่ใช่การเปิดข้อมูลผู้รับบริการ
- ตั้งเป็น `CANCELLED` จะเขียน `cancellation_reason` + `cancelled_at` ลงแถว
- **`status_auto_changed_at` ถูกเก็บไว้เสมอ** แม้เจ้าหน้าที่จะเปลี่ยนสถานะทับ
  จึงยังบอกได้ว่า "ครั้งนี้ระบบเป็นคนเปลี่ยนเองเมื่อไหร่" แม้สถานะปัจจุบันถูกแก้แล้ว

> **คิวอนาคตก็ยกเลิกได้** เช่นเดียวกับคิววันนี้ และยังบังคับเหตุผลเหมือนกัน
> แต่คิวอนาคตคือคนไข้ที่กำลังจะมา — ถ้ามาทันก็ควรใช้การเลื่อนวัน (B7.6) แทนการยกเลิก

`UpdateAppointmentStatusRequest::rules()` · `StaffAppointmentService::updateAppointmentStatus()`

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
  "appointment_date": "2026-10-05",
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
| `appointment_date` | **optional** · ไม่ส่ง = วันนี้ (คนมาถึงหน้าเคาน์เตอร์) · ส่ง = จองล่วงหน้าให้คนไข้ที่ไม่มีอุปกรณ์ · `Y-m-d` · **≥ วันนี้** |
| `time_slot_id` | required — ต้องเปิดใน**วันที่จอง** (ตรวจด้วย `days_mask` ของวันนั้น) และต้องเป็นของศูนย์เดียวกัน |
| `staff_id` | optional — บังคับจริงถ้าบริการเปิดให้เลือกบุคลากร ต้องอยู่ใน `service_staff` ของบริการนั้น |
| `patient_right_id` | optional — ต้องเป็นสิทธิของผู้รับบริการที่ `cid` นั้น (**ผูกกับ cid ไม่ใช่ user**) |

**ศูนย์ที่ถูกบันทึกมาจากบริการที่เลือกเสมอ** — ไม่ได้มาจากค่าที่ส่งมา ดังนั้นการส่ง `health_center_id` ของศูนย์อื่นโดยผู้ดูแลศูนย์จะไม่ทำให้นัดหมายไปตกศูนย์อื่น แต่จะถูกปฏิเสธเพราะบริการที่เลือกไม่ใช่ของศูนย์นั้น

**จองล่วงหน้าให้คนไข้ที่ไม่มีอุปกรณ์** — ส่ง `appointment_date` เป็นวันที่ต้องการ
คนไข้ไม่มีอินเทอร์เน็ตหรือสมาร์ทโฟน จึงต้องพิมพ์ใบนัดให้จากหน้าจอนี้
ครั้งหน้าคนไข้เข้าสู่ระบบเองด้วยเบอร์ + วันเกิด และเห็นคิวนี้ในระบบได้

> ⚠️ **ทุกอย่างที่เกี่ยวกับวันที่ ต้องใช้วันที่จอง ไม่ใช่วันที่กด**
> - `days_mask` ของช่วงเวลา → ตรวจด้วยวันในสัปดาห์ของวันที่จอง
> - `isServiceAvailable` → ตรวจวันลาของวันที่จอง
> ถ้าใช้วันที่กด การจองวันอนาคตจะตรวจผิดวันและผ่านทั้งที่บริการไม่เปิดวันนั้น

**ที่ต่างจาก `POST /patient/book`:**

| | patient book | walk-in |
|---|---|---|
| `appointment_date` | required | optional (default วันนี้) |
| เงื่อนไขเวลาสิ้นสุดของรอบ | **ตรวจ** ย้อนหลังไม่ได้ | **ไม่ตรวจ** เพราะมาถึงหน้าเคาน์เตอร์แล้ว |
| `patient_right_id` ผูกกับ | user ของผู้เรียก | `cid` ที่ส่งมา |
| สร้างผู้รับบริการใหม่ | ไม่ได้ (ต้อง login) | ได้ ถ้า `cid` ยังไม่มี |

**ผู้รับบริการเดิม vs ใหม่**

| กรณี | ระบบทำอะไร |
|---|---|
| `cid` ยังไม่มี | สร้างผู้รับบริการใหม่พร้อมข้อมูลครบ และสร้างสิทธิหลักจาก `right_type_id` (ถ้าส่งมา) |
| `cid` มีอยู่แล้ว + เบอร์โทรและวันเกิดตรง | **เติมเฉพาะช่องที่ว่าง** ชื่อ/ที่อยู่/เพศที่มีอยู่แล้วจะไม่ถูกทับ |
| `cid` มีอยู่แล้ว + เบอร์โทรหรือวันเกิดไม่ตรง | **422** — แปลว่าไม่ใช่ผู้รับบริการคนเดียวกัน ให้เจ้าหน้าที่ตรวจสอบก่อน |
| `cid` ใหม่ + เบอร์โทรไปตรงกับผู้รับบริการคนอื่น | **422 พร้อมเลขบัตรประชาชนของเจ้าของเบอร์** — ไม่ใช่ 500 เพราะเจ้าหน้าที่ต้องรู้ว่าต้องตรวจสอบอะไร |
| ผู้รับบริการเดิมมีสิทธิหลักแล้ว | ใช้สิทธิหลักเดิม ไม่สร้างสิทธิซ้ำ และ `right_type_id` ที่ส่งมาจะไม่ถูกใช้ |
| ไม่ระบุ `patient_right_id` | ใช้สิทธิหลักของผู้รับบริการ (ถ้าไม่มีสิทธิเลย → นัดหมายไม่มีสิทธิผูกไว้) |

> เหตุผลที่ยืนยันตัวตนด้วยเบอร์+วันเกิด: ถ้าเก็บข้อมูลใหม่ทับของเดิม ตัวตนที่บันทึกไว้จะไม่ตรงกับที่ผู้รับบริการใช้ยืนยัน

**ตรวจเพิ่มก่อนจอง (นอกจากชั้นของการจองปกติ):**
- บุคลากรทั้งหมดลา**วันที่จอง** → **422** `"บริการนี้ปิดชั่วคราว เนื่องจากเจ้าหน้าที่ทั้งหมดลางานในวันนี้"`

> ❌ **ไม่มีการตรวจขอบเขตบริการที่นี่** ต่างจาก B7.2/B7.4/B7.5 ที่ตรวจ
> เพราะ route นี้เปิดให้เฉพาะผู้ดูแลอยู่แล้ว ซึ่งไม่ถูกจำกัดขอบเขตบริการ

> ข้อความ error ที่เกิดระหว่างสร้างนัดหมาย **ไม่ถูกกลืน** — `BookingException` ถูกจับแยกและส่งข้อความจริงออกไปพร้อมรหัสที่ถูกต้อง
> ต่างจาก `POST /v1/patient/book` ที่กลืนเฉพาะข้อผิดพลาดของระบบ (ดูหัวข้อ 0.6)

`WalkInBookingRequest::rules()` · `StaffAppointmentController::walkInBooking()` · `StaffAppointmentService::createWalkInAppointment()`

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

**`POST /v1/staff/appointments/{id}/unmask`**

```json
{ "health_center_id": 1 }   // required เฉพาะ SUPER_ADMIN
```

- **ข้อมูลอ่อนไหวตาม PDPA** — เปิดข้อมูลตัวตนจริง
- **ไม่ต้องส่ง body สำหรับ STAFF / HC_ADMIN** เพราะศูนย์ถูกบังคับเป็นศูนย์ตนเอง
  แต่ **SUPER_ADMIN ต้องส่ง `health_center_id`** ไม่ส่ง = 422 (ไม่ใช่ endpoint ไม่ต้องใช้ศูนย์)
- **throttle `unmask` 10 ครั้ง/นาที** (ดู B2.3) — เข้มกว่าโควตารวมของเจ้าหน้าที่เสมอ
- **เขียน `AuditLog` action `UNMASK_PATIENT_DATA` ก่อนคืนข้อมูลทุกครั้ง**
- คืน `id`, `cid`, `full_name` (ชื่อ + นามสกุลเชิงเส้น), `phone_number`, `birth_date`,
  `gender`, `address` (เฉพาะ `full_address`), `rights[]` — **ทั้งหมดเป็นค่าจริง ไม่ mask**
- audit payload มีแค่ `appointment_id` — ไม่บันทึกข้อมูลตัวตนลง log
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

### B7.6 เลื่อนคิวไปวันอื่น

**`PATCH /v1/staff/appointments/{id}/reschedule`**
```json
{ "health_center_id": 1, "appointment_date": "2026-10-05", "time_slot_id": 2, "staff_id": 5 }
```

ใช้เมื่อหมอติดงานด่วนจนให้บริการไม่ได้ จึงโทรคุยกับผู้รับบริการแล้วย้ายวันมา
**แก้คิวเดิม** — ไม่สร้างคิวใหม่ ไม่ยกเลิกคิวเดิม

| field | rule |
|---|---|
| `health_center_id` | **required เฉพาะผู้ดูแลระบบ** (เหมือนทุกทางที่เจาะจงศูนย์) |
| `appointment_date` | required, `Y-m-d`, **≥ วันนี้** |
| `time_slot_id` | required — ต้องเป็นของศูนย์เดิม และต้องเปิดในวันใหม่ |
| `staff_id` | optional — ไม่ส่ง = ใช้หมอคนเดิม · ส่ง = ต้องผูกบริการ + ไม่ลาวันใหม่ + ไม่เต็มรอบ |

**ตรวจตามลำดับ (ทั้งหมดในธุรกรรมเดียว พร้อมล็อกแถวศูนย์และแถวนัดหมาย):**
1. `status = CONFIRMED` — คิวที่เสร็จ/ยกเลิก/ไม่มาแล้วเลื่อนไม่ได้ → 422
2. ช่วงเวลาเปิดในวันใหม่ → 422
3. ไม่มีอะไรเปลี่ยนเลย (วันเดิม + รอบเดิม + หมอเดิม) → 422
4. หมอผูกบริการ + ไม่ลาวันใหม่ → 422
5. หมอไม่เต็มรอบวันใหม่ → 422
6. ผู้รับบริการไม่มีคิวอื่นของบริการนี้ในวันใหม่ → 422

**เลขคิวคำนวณใหม่จากวันใหม่ทุกครั้ง** — เลขเดิมนับรวมคิวของวันที่ย้ายออกไปแล้ว
เก็บไว้จะชนกับคิวอื่นของวันใหม่ ทำให้ลำดับเลขของวันนั้นไม่เรียง

> ⚠️ **ผลที่ต้องยอมรับ: ใบนัดที่พิมพ์ไปแล้วจะบอกเลขคิวผิด**
> เป็นผลจากการเลือก "แก้คิวเดิม + ไม่เก็บประวัติการเลื่อน"
> ไม่มีการเก็บว่าคิวเคยถูกเลื่อนจากวันไหน จึงย้อนกลับไม่ได้
> หน้าจอควรพิมพ์ใบนัดใหม่ทุกครั้งหลังเลื่อน

**ผู้เรียกได้:** `HEALTH_CENTER_ADMIN` · `SUPER_ADMIN` · `STAFF` (เฉพาะเงื่อนไขขอบเขตบริการใน 0.3a)
**ไม่เขียน audit log** — เป็นการแก้ตารางนัดหมาย ไม่ใช่การเปิดข้อมูลผู้รับบริการ

`RescheduleAppointmentRequest::rules()` · `StaffAppointmentController::reschedule()` · `StaffAppointmentService::rescheduleAppointment()`

---

## B8. Roster (ทะเบียนบุคลากร/หมอนวด)

> `staff` = **profile คนงาน** (มี `category_id`, `position`, `user_id`) — ดู 0.3b และ B12

### B8.1 ดูรายชื่

**`GET /v1/staff/roster`**

| query | ผล |
|---|---|
| — | HC_ADMIN/STAFF = ศูนย์ตัวเอง; SUPER_ADMIN = ทุกศูนย์ หรือ `?health_center_id=` |
| `?date=YYYY-MM-DD` | เพิ่ม field **`effective_status`** ที่รวม 2 กลไกเข้าด้วยกัน |

**ทุกรายการมี** `id`, `name`, `position`, `category`, `status`, `user`, **`health_center`**

`user` คืน `{ id, username }` — โปรไฟล์ที่ยังไม่มีบัญชีจะได้ `id: null`
เพื่อให้ผู้ดูแลเห็นว่าใครเข้าใช้ระบบได้แล้ว

**`effective_status` (มีเฉพาะตอนส่ง `?date=`):**
```
Staff.status = ACTIVE + มี StaffLeave วันนั้น  →  effective_status = 'LEAVE'
กรณีอื่น                                        →  effective_status = ค่าเดิมของ status
```

**วันลากรองจากศูนย์เดียวกันเท่านั้น** — ถ้าไม่กรองศูนย์ บุคลากรของศูนย์อื่นที่ลาวันเดียวกันจะโผล่มา

`StaffRosterService::index()` — logic อยู่ใน service ตอนนี้ `StaffRosterController` แค่เรียกใช้

> ⚠️ **roster ไม่ถูกจำกัดด้วยขอบเขตบริการ** — STAFF เห็นทั้งศูนย์เสมอ (ดู 0.3a)
> เพราะใครเข้าเวรหรือลาวันนี้เป็นเรื่องของศูนย์ ไม่ใช่ของบริการเดียว
> แตกต่างจากรายการคิวที่ถูกจำกัดตามบริการ

> **สองกลไกของ "หมอไม่ว่าง" — อย่าสับสน**
> - `StaffLeave` row = **ลาเฉพาะวัน** (B9)
> - `Staff.status = 'LEAVE'` = **ปิดรับจองต่อเนื่อง** (B8.3)
> - roster คือจุดอ่านรวมทั้งสองแบบ

### B8.2 เพิ่มบุคลากร

**`POST /v1/staff/roster`** — **HC_ADMIN / SUPER_ADMIN เท่านั้น** (STAFF → 403)
```json
{ "user_id": 10, "name": "แม่หมอ...", "category_id": 1, "position": "หมอนวด", "status": "ACTIVE" }
```

| field | rule |
|---|---|
| **`user_id`** | **required** — ต้องเป็นบัญชีที่**ยังไม่มีโปรไฟล์บุคลากร** และยังไม่ถูกลบ มิฉะนั้น 422 |
| `name` | required, max 255 |
| `status` | **required** ∈ `ACTIVE / INACTIVE / LEAVE` |
| `category_id` | optional, `exists:categories,id` |
| `position` | optional, max 100 |

**`health_center_id` ไม่ต้องส่ง** — นำจากศูนย์ของ `user_id` เสมอ ไม่ใช่ศูนย์ของผู้เรียก
ไม่งั้นผู้ดูแลศูนย์หนึ่งจะย้ายคนของอีกศูนย์มาไว้ใต้ศูนย์ตนได้
→ HC_ADMIN ส่ง `user_id` ของศูนย์อื่น → **404** `"ไม่พบบัญชีเจ้าของที่ระบุ"`

> เอกสารเดิมไม่มี `user_id` เลย และเขียนว่า "STAFF ธรรมดาทำได้จริง" — ผิดทั้งสองจุด
> route อยู่ใน group `role:SUPER_ADMIN,HEALTH_CENTER_ADMIN`

`StaffRosterController::store()` · `CreateStaffRequest::rules()`

### B8.3 สลับเวร (Active ↔ Leave)

**`PATCH /v1/staff/roster/{id}/toggle-duty`**

| เงื่อนไข | ผล |
|---|---|
| `status = INACTIVE` | **422** `"ไม่สามารถสลับสถานะได้ เนื่องจากบุคลากรถูกระงับการใช้งาน"` (ต้องเปิดใช้งานก่อน — ทำได้ที่ B8.2 ตอนสร้าง) |
| `ACTIVE → LEAVE` ที่มีคิว CONFIRMED ตั้งแต่วันนี้ | **422** `"ไม่สามารถปิดสถานะเจ้าหน้าที่ได้ เนื่องจากมีคิวที่จองไว้แล้ว N คิว กรุณาจัดการคิวก่อน"` |
| `LEAVE → ACTIVE` | เปิดให้จองกลับทันที ไม่มีเงื่อนไขกิจกรรม |

- **เปิดให้ STAFF ทั่วไปเรียกได้** — route อยู่ใน group ทุกบทบาท (ศูนย์ถูกบังคับเป็นศูนย์ตนเอง)
- โปรไฟล์ของศูนย์อื่น → **404**
- **ไม่มีการเขียน audit log** — เป็นการสลับสถานะการให้บริการ ไม่ใช่การแตะข้อมูลผู้รับบริการ

`StaffRosterController::toggleDuty()`

---

## B9. วันลา

### B9.1 ดูวันลา

**`GET /v1/staff/leaves?date=&staff_user_id=`** — ไม่ paginate คืนทั้งหมด

| query | ผล |
|---|---|
| — | HC_ADMIN/STAFF = ศูนย์ตัวเอง; SUPER_ADMIN ไม่ส่ง = **ทุกศูนย์**; ส่ง = กรองศูนย์นั้น |
| `?date=` | เฉพาะวันนั้น |
| `?staff_user_id=` | **user id** (บัญชี) ไม่ใช่ staff id |

- เรียงวันที่มาก→น้อย · คืนพร้อมชื่อบัญชีของคนที่ลา (`id`, `name`, `username`)
- **ไม่จำกัดด้วยขอบเขตบริการ** — เช่นเดียวกับ roster เพราะวันลาเป็นเรื่องของศูนย์

`StaffLeaveController::index()`

### B9.2 ลงวันลา

**`POST /v1/staff/leaves`**
```json
{ "staff_user_id": 10, "leave_date": "2026-09-26", "reason": "ไปราชการ" }
```

| field | rule |
|---|---|
| `staff_user_id` | **STAFF ส่งไม่ต้อง** (ระบบใส่ให้) · ผู้ดูแลขึ้นไป **required** · ต้องเป็น user ที่ยังไม่ถูกลบ |
| `leave_date` | required, **≥ วันนี้** (ย้อนหลังไม่ได้) |
| `reason` | optional, max 500 |

**สิทธิ์:**

| role | ลงให้ใครได้ | ไม่ผ่าน |
|---|---|---|
| STAFF | บังคับเป็นตัวเองเสมอ (ส่งมาหรือไม่ส่งก็ได้) | ระบุคนอื่น → 422 `"ลาด้วยตนเองได้ ระบุเจ้าหน้าที่ท่านลาด้วยตนเองไม่ได้"` |
| HC_ADMIN | ใครในศูนย์ตัวเองก็ได้ | คนนอกศูนย์ → 422 `"ไม่พบเจ้าหน้าที่ที่ระบุในศูนย์ของคุณ"` |
| SUPER_ADMIN | ทุกที่ | — |

**กติกาสำคัญ 4 ข้อ:**
1. **บล็อก 422** ถ้าหมอคนนั้นมีคิว `CONFIRMED` ในวันที่ลา → `"ไม่สามารถลงวันลาได้ เนื่องจากมีคิวที่จองไว้แล้ว N คิวในวันนี้"`
2. **Idempotent** — ลงซ้ำวันเดิมคืน row เดิม **HTTP 200** (ไม่ใช่ 201, ไม่ error)
3. **บัญชีที่ไม่มีโปรไฟล์บุคลากร ลาไม่ได้** → 422 `"บัญชีนี้ยังไม่มีโปรไฟล์บุคลากรในศูนย์สุขภาพนี้ กรุณาเพิ่มโปรไฟล์ก่อนลงวันลา"`
   ระบบไม่รู้ว่าคนนี้คือใคร การเดาจากหมวดหมู่เคยทำให้ผูกผิดคน จึงไม่เดาแล้ว
4. **วันลาเชื่อผูกกับ `staff.user_id`** — โปรไฟล์ที่ไม่ได้ผูกบัญชีถือว่า "ไม่ลาผ่านระบบนี้" เสมอ
   → ต้องมีโปรไฟล์ **ของศูนย์นั้น** ไม่ใช่แค่มีที่ไหนบ้าง

> เอกสารเดิมเขียนว่าระบบเดาโปรไฟล์ให้เมื่อเจอคนที่ตรงเงื่อนไขเดียว — ผิด ปัจจุบันไม่เดาแล้ว

`StaffLeaveController::store()` · `StoreLeaveRequest::rules()`

### B9.3 ยกเลิกวันลา

**`DELETE /v1/staff/leaves/{id}`** — hard delete (ไม่ใช่ soft delete)

| role | เงื่อนไข | ไม่ผ่าน |
|---|---|---|
| STAFF | ลบได้เฉพาะของตัวเอง | **404** (ไม่ใช่ 403 — ปิดบังการมีอยู่) |
| HC_ADMIN | ลบได้เฉพาะของศูนย์ตัวเอง | 404 |
| SUPER_ADMIN | ลบได้ทุกที่ | — |

- **ไม่มีการตรวจคิวค้าง** ต่างจากตอนลงวันลา — ยกเลิกวันลาไม่กระทบคิวที่จองไว้แล้ว
- ไม่มีการเขียน audit log — เป็นการแก้ตารางวันลา ไม่ใช่การเปิดข้อมูลผู้รับบริการ

`StaffLeaveController::destroy()`

### B9.4 เช็คบริการว่างให้บริการวันนั้นไหม

**`GET /v1/staff/leaves/availability?service_id=1&date=2026-09-26`**
```json
{ "success": true, "available": true }
```

**ใช้เมื่อไหร่:** อยากเช็คบริการเดียวโดยไม่ต้องดึงรายการทั้งหมด
**แหล่งข้อมูลหลัก** ของความพร้อมของบริการคือ `GET /v1/staff/services?date=` (B3.1) ซึ่งคืนรายการเฉพาะที่พร้อม — endpoint นี้เป็นตัวย่อยจาก **กติกาเดียวกัน** (`StaffLeave::isServiceAvailable`)

**ความหมายของ `available` — อ่านให้ถูก ไม่ใช่แค่ "มีคิวว่าง":**

| สถานการณ์ | ผล |
|---|---|
| มี pool ของคนงาน แล้วลาครบ | `false` |
| มี pool แต่ยังมีคนไม่ลา | `true` |
| **ไม่มี pool เลย** | **`true` เสมอ** — ความพร้อมถูกควบคุมโดย `capacity` แทน |

- pool ขึ้นกับ flag: `allow_staff_selection=true` → `service_staff` · `false` → `service_user`
- **เช็คแค่วันลา ไม่เช็คโควตา** — บริการที่ยังมีโควตาว่างแต่หมอลาครบก็จะได้ `false`
- วันลาที่นับถูกกรองด้วย `health_center_id` ด้วยเสมอ

`StaffLeaveController::availability()` · `StaffLeave::isServiceAvailable()`

---

## B10. ค้นหาและแก้ข้อมูลผู้รับบริการ

> **เงื่อนไขร่วมของการแก้ไข (B10.2 / B10.3):** ผู้รับบริการต้อง**เคยมีประวัติคิวที่ศูนย์นั้น** — ผู้รับบริการที่ยังไม่เคยมาศูนย์จะแก้ไม่ได้
> (tenant isolation ผ่าน queue history ไม่ใช่คอลัมน์บน `patients`)
> **แต่การค้นหา (B10.1) ผ่อนข้อยกเว้นนี้เมื่อค้นด้วย `cid`** — ดูรายละเอียดด้านล่าง

### B10.1 ค้นหาผู้รับบริการ

**`GET /v1/staff/admin/patients`** — HC_ADMIN / SUPER_ADMIN (STAFF → 403)

| query | ผล |
|---|---|
| `q` | ค้นชื่อ/นามสกุล (max 100) — ค้นทั้ง `first_name` และ `last_name` |
| `cid` | 13 หลัก — **ข้ามการกรองด้วยศูนย์** (ดูข้อความล่าง) |
| `phone` | max 20 — ค้นแบบ contains ไม่ใช่ตรงทั้งหมด |
| `gender` | `MALE/FEMALE/OTHER` |
| `province` / `district` | max 100 — ค้นแบบ contains |
| `health_center_id` | SUPER_ADMIN เท่านั้น; HC_ADMIN ถูกบังคับศูนย์ตัวเอง |
| `per_page` | 1-100, default 15 |
| `sort_by` | `created_at`/`first_name`/`last_name`/`id` (default `created_at`) |
| `order` | `asc`/`desc` (default `desc` — ใหม่สุดก่อน) |

> ⚠️ **ค้นด้วย `cid` = การยืนยันตัวตน ไม่ใช่การค้นในข้อมูลของศูนย์**
> พิมพ์ `cid` แล้วจะ**ข้ามเงื่อนไข "ต้องเคยมาศูนย์นี้"** ไปเลย เห็นผู้รับบริการจากศูนย์อื่นด้วย
> เหตุผล: ผู้รับบริการที่ลงทะเบียนเองที่ศูนย์อื่นต้องมาหาเจอที่หน้าเคาน์เตอร์ได้
> ถ้ากรองด้วยศูนย์ตรงนี้ ระบบจะเข้าใจว่าไม่เคยลงทะเบียน แล้วลงทะเบียนซ้ำจนชนเบอร์
>
> การค้นที่**พบผู้รับบริการและค้นด้วย `cid`** → เขียน `AuditLog` action `LOOKUP_PATIENT_BY_CID` (ดู C.3)

`StaffPatientIndexRequest::rules()` · `StaffPatientController::index()` · `StaffPatientService::index()`

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

| field | rule |
|---|---|
| `health_center_id` | SUPER_ADMIN ต้องส่ง · HC_ADMIN ถูกบังคับเป็นศูนย์ตัวเอง |
| `cid` | optional, 13 หลัก, unique |
| `first_name` / `last_name` | optional, max 100 |
| `birth_date` | optional, **ต้องเป็นวันที่ในอดีต** (`before:today`) |
| `gender` | optional, `MALE/FEMALE/OTHER` |
| `subdistrict` / `district` / `province` | optional, max 100 |
| `full_address` | optional, ไม่จำกัดความยาว |

- ทุก field optional · ไม่มี field เปลี่ยนจริง → `"ข้อมูลไม่มีการเปลี่ยนแปลง"` + **ไม่เขียน audit log**
- เขียน `AuditLog` action `UPDATE_PATIENT_PROFILE` (mask ทั้ง cid และเบอร์)
- **ไม่แตะ `phone_number`** — ต้องใช้ B10.2 (ซึ่งมีกติกาขอบเขตบริการเข้มกว่า)
- **STAFF ธรรมดา → 403** (route อยู่ใน group ผู้ดูแลขึ้นไป) ต้องใช้ B10.2 แทน
- response คืนข้อมูลจริงทั้งหมด **ไม่ mask** เพราะเป็นการแก้ไขข้อมูลของผู้รับบริการโดยตรง

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
- **`health_center_id` ต้องตรงกับ `{id}` ใน path** ไม่ตรง → **404** ตรวจก่อนแตะข้อมูล
- ใช้ `lockForUpdate` + เขียน `AuditLog` (payload มี `from`/`to`)

**แผนที่สถานะ:**

| สถานะปัจจุบัน | ผลลัพธ์ |
|---|---|
| `ACTIVE` + ยังมีคิวค้าง | → `CLOSING` |
| `ACTIVE` + **ไม่มีคิวค้างเลย** | → `INACTIVE` ทันที (ข้ามขั้น CLOSING) |
| `CLOSING` | → `ACTIVE` (เปิดใหม่ได้) |
| `INACTIVE` | → `ACTIVE` (เปิดคืนได้) |

> ⚠️ **สถานะ INACTIVE เกิดได้ทันทีจากการสลับครั้งเดียว** ถ้าตอนกดปิดไม่มีคิวค้างเลย
> ไม่ได้ต้องรอให้คิวหมดเสมอไป — จึงไม่มีสถานะระหว่างที่ต้องรอ
>
> เกณฑ์ "คิวค้าง" คือ `status = CONFIRMED` และ `appointment_date >= วันนี้` ตัวเดียวกับ C.2
> และเป็นเกณฑ์เดียวกับที่ `CloseHealthCenterJob` ใช้ จึงไม่มีกติกาสองชุดขัดกันในสถานการณ์เดียวกัน

`StaffHealthCenterController::toggleStatus()` · `StaffHealthCenterService::toggleStatus()`

### B11.4 แก้ข้อมูลศูนย์

**`PATCH /v1/staff/health-centers/{id}`** — HC_ADMIN / SUPER_ADMIN
```json
{ "health_center_id": 1, "name": "...", "phone_number": "...", "full_address": "...",
  "latitude": 13.75, "longitude": 100.50 }
```

| field | rule |
|---|---|
| `code` | unique (max 20) — **HC_ADMIN ส่งมา → 403** `"ไม่มีสิทธิ์แก้ไขรหัสศูนย์สุขภาพ"`; SUPER_ADMIN แก้ได้ |
| `name` / `phone_number` / `subdistrict` / `district` / `province` / `full_address` | optional, ทุก field nullable |
| `status` | **ถูกบังคับถอดออกเสมอ** — ต้องใช้ B11.3 แทน (แม้จะผ่าน validation มาก่อนก็ถูกตัดทิ้ง) |
| `discord_webhook_url` | **ถูกถอดออกเสมอ** — ต้องใช้ B11.5 แทน |
| `latitude` / `longitude` | numeric, -90..90 / -180..180 |

- HC_ADMIN ส่ง `{id}` ของศูนย์อื่น → **404** (ตรวจก่อนแตะข้อมูล)
- **`code` ซ้ำ → 422** ผ่าน 2 ชั้น: `Rule::unique` ใน validation และจับ `QueryException` ใน service เผื่อชนกันพร้อมกัน
- **ค่าที่ส่งมาเป็น `null` จะถูกกรองทิ้ง** — ไม่สามารถล้างค่าของศูนย์ด้วยการส่ง `null`
- ไม่มี field เปลี่ยนจริง → ข้อความ `"ข้อมูลไม่มีการเปลี่ยนแปลง"` + **ไม่เขียน audit log**
- response คืน 10 field ของศูนย์ (ไม่ใช่ `has_webhook` แบบของ B11.1)

`UpdateHealthCenterRequest::rules()` · `StaffHealthCenterController::update()` · `StaffHealthCenterService::updateProfile()`

### B11.5 ตั้ง Discord Webhook (SUPER_ADMIN)

**`PATCH /v1/staff/discord/webhook`**
```json
{ "health_center_id": 1, "webhook_url": "https://discord.com/api/webhooks/123456/abcXYZ" }
```
- `webhook_url` ต้อง match regex `^https://discord\.com/api/webhooks/[0-9]+/[A-Za-z0-9_-]+$`, max 500
  ปฏิเสธทุกโดเมนอื่นรวมถึง `discordapp.com` และ `ptb./canary.`
- **ส่ง `null` = ลบ webhook** (ปิดการแจ้งเตือนของศูนย์นั้น) — ข้อความตอบกลับต่างกันตามเคส
- `health_center_id` ไม่ส่ง → 422 (ผู้ดูแลระบบต้องระบุศูนย์ เพราะ webhook เป็นของศูนย์), ไม่มีจริง → 404
- **audit log เก็บแค่ `set: true/false` ไม่เก็บ URL** เพราะ webhook คือค่าลับ — ตรวจย้อนหลังได้ว่าเคยตั้งหรือลบ แต่ย้อนค่าเดิมไม่ได้
- ถ้าลบ/ตั้งแล้ว `GET /v1/staff/health-centers` จะสะท้อนค่าใหม่ใน `has_webhook`

`UpdateDiscordWebhookRequest::rules()` · `StaffDiscordController::updateWebhook()`

---

## B12. จัดการผู้ใช้ (บัญชี staff)

> `users` = บัญชี login (มี username/password) — ต่างจาก `staff` ใน roster (B8)
> **HC_ADMIN จัดการได้เฉพาะ role STAFF ในศูนย์ตัวเองเท่านั้น**

### B12.1 ดูรายชื่อผู้ใช้

**`GET /v1/staff/admin/users`** — HC_ADMIN / SUPER_ADMIN

| query | ผล |
|---|---|
| `search` | ค้นชื่อ / username (max 100) |
| `health_center_id` | SUPER_ADMIN เท่านั้น; HC_ADMIN ถูกบังคับศูนย์ตัวเอง |
| `status` | `ACTIVE/INACTIVE` |
| `role_ids` | กรองตาม role (array) |
| `sort_by` | `id/name/username/status/created_at` |
| `order` | `asc/desc` (default `asc`) |
| `per_page` | 1-100, default 20 |

> `search` ครอบคลุมทั้ง `name` และ `username` ในเงื่อนไขเดียวกัน
> ถ้าใส่เงื่อนไขศูนย์ไว้ข้างใน ต้องจับกลุ่มไว้ ไม่งั้น OR จะหลุดเงื่อนไขศูนย์ออกไป

`StaffUserManagementController::index()` · `AdminUserIndexRequest::rules()`

### B12.2 ดูรายชื่อบทบาทที่ตัวเองมีสิทธิ์มอบ

**`GET /v1/staff/admin/roles`**
- SUPER_ADMIN → ทุก role
- HC_ADMIN → **เฉพาะ `STAFF`** (ใช้เป็นค่าใน dropdown ตอนสร้างผู้ใช้)

### B12.3 สร้างผู้ใช้

**`POST /v1/staff/admin/users`** — สร้างบัญชี + โปรไฟล์บุคลากรในคำขอเดียว
```json
{ "name": "...", "username": "...", "password": "อย่างน้อย 8 ตัว",
  "role_ids": [3], "health_center_id": 1, "status": "ACTIVE",
  "category_id": 1, "position": "หมอนวด" }
```

| field | rule |
|---|---|
| `name` | required, max 255 — ใช้เป็นชื่อทั้งบัญชีและโปรไฟล์ |
| `username` | required, max 50, unique |
| `password` | required, **min 8** |
| `email` / `phone_number` | optional (max 255 / 20), `email` unique |
| `role_ids` | required array ≥ 1 — **HC_ADMIN ส่ง role อื่น → 422** และ `health_center_id` ถูกบังคับเป็นศูนย์ตัวเอง |
| `health_center_id` | optional ใน Request แต่**ทุกคนต้องมีศูนย์** — ไม่ส่งและไม่มีศูนย์ตัวเอง → 422 `"กรุณาระบุศูนย์สุขภาพที่สังกัดของบัญชีนี้"` (ดู 0.3) |
| `status` | optional, `ACTIVE/INACTIVE` (default `ACTIVE`) — สถานะของ**บัญชี** |
| `category_id` / `position` | optional — ของ**โปรไฟล์บุคลากร** (ดู 0.3b) |

**โปรไฟล์บุคลากรที่ถูกสร้างพร้อมกัน:**
- `name` = ชื่อเดียวกับบัญชี · `user_id` ผูกเข้าหาใหม่ · `health_center_id` = ของบัญชี
- **`status = ACTIVE` เสมอ ไม่ยกมาจาก `status` ของบัญชี** — การหยุดให้บริการเกิดจากวันลา ไม่ใช่จากสถานะบัญชี
  ตั้งบัญชีเป็น `INACTIVE` จึงไม่ทำให้หมอหายจากหน้าจอผู้รับบริการ
- ทั้งสองอย่างอยู่ใน transaction เดียว — ถ้าตัวไหนพังจะไม่เหลือบัญชีค้าง

**ผลลัพธ์** → **201** พร้อม `AdminUserResource` · เขียน `AuditLog` action `USER_CREATED` (มี `staff_id` ใน payload)

`StoreUserRequest::rules()` · `StaffUserManagementController::store()`

### B12.4 แก้ผู้ใช้

**`PATCH /v1/staff/admin/users/{id}`** — ส่งเฉพาะ field ที่ต้องการแก้

| กรณี HC_ADMIN | ผล |
|---|---|
| แก้บัญชี SUPER_ADMIN | 403 `"ไม่สามารถแก้ไขบัญชี Super Admin ได้"` |
| แก้ผู้ใช้ศูนย์อื่น | 404 |
| ส่ง `health_center_id` เพื่อย้ายศูนย์ | **ถูกเมินเสมอ** (`unset`) — SUPER_ADMIN ที่ย้ายศูนย์ได้ |
| ส่ง `role_ids` ที่มี role อื่นนอกจาก STAFF | 422 |
| ถอด role หรือปิดบัญชีของ**ผู้ดูแลศูนย์คนใด** | 403 `"การเปลี่ยนสิทธิ์ผู้ดูแลศูนย์ต้องให้ผู้ดูแลระบบเป็นผู้ดำเนินการ"` |

> **HC_ADMIN ถอดสิทธิ์ผู้ดูแลศูนย์ไม่ได้** แม้แต่ตัวเอง ถ้าได้ศูนย์จะไม่เหลือคนทำหน้าเคาน์เตอร์
> เฉพาะการแก้ชื่อหรือเบอร์โทรศัพท์เท่านั้นที่ HC_ADMIN ทำเองได้
>
> ⚠️ **"ไม่ส่ง `role_ids`" ≠ "ถอดบทบาท"** — ถ้าตีความผิด ผู้ดูแลศูนย์จะแก้ชื่อหรือเบอร์ของตัวเองไม่ได้เลย

**สิ่งที่แก้ไม่ได้จากที่นี่:**
- **ชื่อในโปรไฟล์บุคลากรไม่ซิงก์กับบัญชี** — แก้ `name` ที่นี่เปลี่ยนเฉพาะฝั่ง `users`
  ถ้าต้องเปลี่ยนชื่อที่ผู้รับบริการเห็น ต้องแก้ที่ roster (B8.2)
- `category_id` / `position` ของโปรไฟล์ · `username` ซ้ำ → 422 · ย้าศูนย์ (HC_ADMIN) · เปลี่ยนสิทธิ์ผู้ดูแลศูนย์

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

**ข้อมูลถูก mask เสมอใน response** ยกเว้นเรียก unmask

| accessor | รูปแบบ |
|---|---|
| `masked_full_name` | 2 ตัวแรก + `*** ` + 2 ตัวแรกนามสกุล + `***` |
| `masked_phone` | 3 ตัวแรก + `-***-` + 4 ตัวท้าย |
| `masked_cid` | 1 ตัวแรก + `-****-*****-` + 2 ตัวท้าย |

เป็น accessor ระดับ model จึงใช้กับทุกทางโดยอัตโนมัติ — รวมถึง `appointments` ใน `/dashboard/bundle`
แต่ **ไม่ใช่ทุก response จะ mask** — `updateContact` และ `updateProfile` คืนค่าจริงโดยตรง
เพราะเป็นการแก้ไขข้อมูลของผู้รับบริการรายนั้นโดยตรง

> ⚠️ **ชื่อ field ใน JSON ไม่ตรงกับชื่อ accessor**
> accessor ชื่อ `masked_full_name` แต่ `AppointmentResource::patient()` ตั้งชื่อใน JSON เป็น `masked_name`
> เจ้าหน้าที่หน้าค่าน์ต้องอ่าน `masked_name` — อย่าไปหา `masked_full_name` ใน response

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
>
> **หน้าที่ที่ไม่เขียน audit log** (ไม่ใช่ข้อมูลส่วนบุคคลของผู้รับบริการ):
> เปลี่ยนสถานะคิว · สลับเวรหมอ · ลง/ยกเลิกวันลา · สร้าง-แก้-ลบบริการ · ตั้งวันให้บริการ · มอบหมายผู้รับผิดชอบ

> หมายเหตุ: `USER_*` และ `UPDATE_HEALTH_CENTER*` **ไม่ mask** เพราะไม่ใช่ข้อมูลส่วนบุคคลของผู้รับบริการ
> `audit_logs` มีเพียงคอลัมน์ `payload` (json) — ไม่มีคอลัมน์ `masked_data` แยก
> การ mask จึงเกิดในแต่ละ service ที่สร้าง payload ไม่ใช่ที่ชั้นฐานข้อมูล

## C.4 Discord notification

- ถูก dispatch **หลัง commit** (นอก transaction) — ไม่ rollback การจองเมื่อส่ง Discord ไม่สำเร็จ
- queue = `discord`, retry 3 ครั้ง (backoff 30 / 120 / 600 วินาที) · timeout ต่อครั้ง 30 วินาที
- ศูนย์ไม่มี webhook → `Log::warning` แล้วจบ ไม่ error (ไม่กระทบการจอง)
- ข้อความใน embed ใช้ชื่อผู้รับบริการแบบ **masked**
- retry ครบแล้วยังไม่สำเร็จ → `failed()` บันทึก log แล้วจบ ไม่ throw ต่อ
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
| **สิทธิ์เจ้าหน้าที่มาจากฐานข้อมูลอย่างเดียว** | token ไม่ได้ใช้ตัดสิทธิ์ ถ้าไม่มีบทบาทในฐานข้อมูลจะเข้าฝั่งเจ้าหน้าที่ไม่ได้ — รวมถึงโควตา rate limit ด้วย (ดู B2.3) |
| **`roles` คืนเฉพาะ role แรก** | response ของ `login` ใช้ `roles->first()` ถ้าบัญชีมีหลาย role จะเห็นแค่ตัวเดียว — ต้องดู `GET /v1/staff/me` เมื่อต้องการทั้งหมด |
| **ตารางเชื่อมบริการ↔ช่วงเวลามีคอลัมน์ศูนย์ที่ต้องเขียนค่าเองทุกจุดเขียน** | ค่าผิดจะถูกฐานข้อมูลปฏิเสธ แต่ถ้าเพิ่มจุดเขียนใหม่ต้องส่งศูนย์มาด้วย ไม่งั้นจะ NOT NULL |
| **`RoleSeeder` รันซ้ำได้ปลอดภัย แต่ seeder อื่นยัง truncate** | `RoleSeeder` ไม่ truncate แล้ว (อ้างอิงด้วย `name`) · seeder อื่นยัง truncate ตารางทั้งหมด: `CategorySeeder` · `HealthCenterSeeder` · `RightTypeSeeder` · `ServiceSeeder` (รวม `service_capacity` และ `service_time_slot`) · `StaffSeeder` · `TimeSlotSeeder` · `UserSeeder` (รวม `user_roles`) — **ห้ามรันบนฐานข้อมูลที่มีข้อมูลจริง** |
| **ดัชนีฐานข้อมูลที่เกี่ยวกับคิวมีอยู่แล้ว** | `idx_appointments_capacity_lookup (health_center_id, service_id, appointment_date, time_slot_id, status)` ครอบการนับโควตาและ `lockForUpdate` · `idx_appointments_staff_lookup (staff_id, appointment_date, status)` ครอบการนับคิวรายหมอ · `idx_appointments_duplicate_check (patient_id, service_id, appointment_date, status)` ครอบกันจองซ้ำ · `idx_appointments_status_date (status, appointment_date)` ครอบ scheduler · `uq_appointments_daily_queue` กันเลขคิวซ้ำ — **อย่าเพิ่มดัชนีซ้ำซ้อนกับหัวคอลัมน์เหล่านี้** เพราะจะทำให้ช้าลง |
| **ไม่มี ETag / 304** | ระบบไม่มี cache ระดับ HTTP — ทุกคำขอได้ payload เต็ม ถ้าจะเพิ่มต้องระวังเรื่องข้อมูลข้ามผู้ใช้ เพราะคิวมีข้อมูลผู้รับบริการ |
| **ตัวเลขนับผู้รับบริการในหน้าจอต้องตรงกับรายการที่เห็น** | ถ้าเรียก `/dashboard/summary` แยกจาก `/appointments` แล้ววันที่ของทั้งสองไม่ตรงกัน ตัวเลขจะไม่ตรง — ใช้ `/dashboard/bundle` แก้ (ดู B2.2) |

---

## ภาคผนวก — ตารางอ้างอิงเร็ว

### สถานะ (enum ที่ใช้ทั้งระบบ)

| ค่า | ของ | หมายเหตุ |
|---|---|---|
| `ACTIVE` / `CLOSING` / `INACTIVE` | `health_centers.status` | ดู 0.5 · `INACTIVE` เกิดจากระบบเท่านั้น |
| `ACTIVE` / `INACTIVE` | `users.status` | ลบบัญชี = soft delete (`deleted_at`) + ตั้ง `INACTIVE` |
| `ACTIVE` / `LEAVE` / `INACTIVE` | `staff.status` | `INACTIVE` สลับ duty ไม่ได้ (ดู B8.3) |
| `CONFIRMED` / `COMPLETED` / `CANCELLED` / `NO_SHOW` | `appointments.status` | แผนที่การเปลี่ยนสถานะดู B7.2 |
| `PER_MASSEUSE` / `PER_SLOT` / `PER_DAY` | `services.capacity_type` | ดู 0.4 |
| `SELF` / `STAFF` | `appointments.booked_by_type` | จองเอง / walk-in |
| `MALE` / `FEMALE` / `OTHER` | `patients.gender` | |
| `ACTIVE` / `INACTIVE` | `categories.status` · `right_types.status` | ไม่มี API แก้ (ดู C.5) |

> ตารางอ้างอิงที่ไม่มี enum แต่ต้องรู้: `roles.name` = `STAFF` / `HEALTH_CENTER_ADMIN` / `SUPER_ADMIN`
> ชื่อ role ถูก**อ้างอิงเป็นข้อความ** ไม่ใช่ id — ถ้าเปลี่ยนชื่อในฐานข้อมูล ระบบทั้งหมดจะพัง

### ตาราง pivot ที่ต้องรู้

| pivot | ความสัมพันธ์ | ใช้ทำอะไร |
|---|---|---|
| `service_staff` | service ↔ staff (หมอที่เลือกได้) | **เลือกหมอ** + กรองความพร้อม |
| `service_user` | service ↔ user (role STAFF) | **ทางที่ 1 ของขอบเขตบริการ** — จำกัดว่า STAFF เห็น/แตะคิวของบริการไหนได้ |
| `service_time_slot` | service ↔ time_slot + `days_mask` + `health_center_id` | **วัน/เวลาที่เปิดให้บริการ** — คอลัมน์ศูนย์ต้องตรงกับทั้งบริการและช่วงเวลา (composite FK) |
| `user_roles` | user ↔ role | ตรวจสิทธิ์ทุก request |
| `patient_rights` | patient ↔ right_type | สิทธิการรักษา (มี `is_primary` — บังคับให้มีได้เพียงหนึ่งรายการต่อคน บังคับที่ระดับฐานข้อมูล) |
| `staff_leaves` | staff_user_id + leave_date | วันลา — **ผูกกับ user ไม่ใช่ staff profile** (ดู B9) |

> ⚠️ **ตาราง `service_time_slot_days` ถูกลบทิ้งแล้ว** — เคยเป็นตารางแยกต่อ (slot, day)
> ปัจจุบันรวมเป็น bitmask ใน `service_time_slot.days_mask` แทน
> ชื่อ `Service::timeSlotDays()` ที่ยังใช้อยู่เป็น **alias** ของ `timeSlots()` ไม่ใช่คนละตาราง

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
| auto-close ไม่ทำงาน | `CloseHealthCenterJob.php` + สถานะ `queue_worker` — ต้องมี action ต่อคิวก่อน (ดู C.5) |
| Discord ไม่ส่ง | `SendDiscordNotificationJob.php` + `queue_worker` + ค่า `discord_webhook_url` ของศูนย์ |
| validation message ไม่ตรงที่คาด | `app/Http/Requests/Api/**` (messages อยู่ในไฟล์เดียวกัน) |
| ตอบ 404 ทั้งที่ควรเจอ | ตรวจ `deleted_at` (soft delete) และ `is_active` ของศูนย์/บริการ/ช่วงเวลาที่เกี่ยว |
| ตอบ 422 ที่คาดว่า 403 | ตรวจว่ามี FormRequest ที่ validate เงื่อนไขเดียวกันอยู่ก่อนถึงชั้น controller หรือไม่ |

### วิธีตรวจว่าเอกสารนี้ยังตรงกับโค้ด

เอกสารนี้อ้างอิงตำแหน่งด้วย **ชื่อฟังก์ชันและชื่อไฟล์** ไม่ใช้เลขบรรทัดหรือ commit
เพราะทั้งสองอย่างหมดอายุทันที่มีการแก้ไฟล์นั้น และเอกสารนี้เคยตกเพราะเหตุนี้
ถ้าเอกสารกับโค้ดไม่ตรงกัน ให้เชื่อโค้ดเสมอ

เมื่อสงสัยว่าข้อไหนยังจริง ให้ grep ตามชื่อฟังก์ชันที่ระบุ เช่น

```
grep -rn "mayActOnService" app/
```

สิ่งที่ควรตรวจเป็นพิเศษเมื่อมีการเปลี่ยนแปลง:
- **ตัวเลขสรุปในเอกสาร** — นับจากฐานข้อมูลจริงเสมอ อย่าใส่ตัวเลข snapshot
- **ชื่อ action ของ audit log** — grep `'action' => '` ทั่ว `app/`
- **ชื่อ field ใน response** — ดู resource ที่เกี่ยวข้อง เพราะ accessor กับชื่อใน JSON ไม่จำเป็นต้องตรงกัน
- **เงื่อนไขการจำกัดสิทธิ์** — ดู route group ใน `routes/api.php` ก่อนสรุปว่าใครเข้าถึงอะไรได้
