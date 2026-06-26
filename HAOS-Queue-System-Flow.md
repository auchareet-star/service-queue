# HAOS Queue System — Flow การทำงานทั้งหมด

> เอกสารสรุป Flow การทำงานของระบบจัดการคิว (Queue Management System)
> Tech Stack: .NET 8.0 (WebApi) + Angular 18 (WebClient) + SignalR (Real-time)
> อัปเดต: 2026-06-26

---

## สารบัญ

1. [ภาพรวมระบบ (System Overview)](#1-ภาพรวมระบบ)
2. [ส่วนประกอบหลัก (Components)](#2-ส่วนประกอบหลัก)
3. [สถานะคิว (Queue Status)](#3-สถานะคิว-queue-status)
4. [Flow หลัก: วงจรชีวิตของคิว (Queue Lifecycle)](#4-flow-หลัก-วงจรชีวิตของคิว)
5. [Flow การออกบัตรคิว (Issue Ticket)](#5-flow-การออกบัตรคิว)
6. [Flow การเช็คอิน (Check-in)](#6-flow-การเช็คอิน)
7. [Flow การเรียกคิว (Call Next)](#7-flow-การเรียกคิว)
8. [Flow ระหว่างให้บริการ (Recall / Pause / Forward / Complete)](#8-flow-ระหว่างให้บริการ)
9. [Flow Real-time ผ่าน SignalR](#9-flow-real-time-ผ่าน-signalr)
10. [Flow จอแสดงผล TV Display](#10-flow-จอแสดงผล-tv-display)
11. [Flow ฝั่งลูกค้า Mobile/Kiosk](#11-flow-ฝั่งลูกค้า-mobilekiosk)
12. [Flow Authentication & Authorization](#12-flow-authentication--authorization)
13. [Flow ข้อมูลหลัก & รายงาน (Master Data & Report)](#13-flow-ข้อมูลหลัก--รายงาน)
14. [สรุป API Endpoints](#14-สรุป-api-endpoints)
15. [ตาราง Database หลัก](#15-ตาราง-database-หลัก)

---

## 1. ภาพรวมระบบ

ระบบ HAOS Queue เป็นระบบจัดการคิวสำหรับเคาน์เตอร์บริการและห้องรอ มีความสามารถหลัก:

- **ออกบัตรคิว** ผ่าน Kiosk / Mobile / เจ้าหน้าที่
- **เรียกคิว** ที่เคาน์เตอร์ (Service Point) แบบ Real-time
- **ส่งต่อคิว (Forward)** ไปยังห้อง/บริการอื่น
- **พักคิว (Hold/Break)** พร้อมระบุเหตุผล
- **จอแสดงผล TV** แสดงคิวที่กำลังเรียกแบบสด
- **รายงาน & Dashboard** วิเคราะห์เวลารอ/เวลาให้บริการ

```
┌─────────────┐   ออกบัตร   ┌──────────────┐  เรียกคิว  ┌────────────────┐
│  ลูกค้า/Kiosk │ ─────────► │  ระบบคิว (API) │ ◄──────── │ เจ้าหน้าที่/เคาน์เตอร์ │
└─────────────┘             └──────┬───────┘           └────────────────┘
                                   │ SignalR (Real-time)
                       ┌───────────┴───────────┐
                       ▼                       ▼
                ┌────────────┐         ┌──────────────┐
                │ จอ TV Display │         │ Mobile ติดตามคิว │
                └────────────┘         └──────────────┘
```

---

## 2. ส่วนประกอบหลัก

| Layer | Project | หน้าที่ |
|-------|---------|--------|
| **WebApi** | `HAOS.Queue.WebApi` | Controllers, Middleware, Authentication |
| **Business** | `HAOS.Queue.Business` | Service / Business Logic |
| **Core** | `HAOS.Queue.Core` | DTO, Constants, SignalR Hub (`EventHub`) |
| **DAL** | `HAOS.Queue.DAL` | Entity Framework, Entities, Migrations |
| **WebClient** | Angular App | หน้าจอเจ้าหน้าที่ / Kiosk / Mobile / Admin |
| **QueueTV** | Angular App แยก | จอแสดงผลในห้องรอ |

---

## 3. สถานะคิว (Queue Status)

กำหนดใน `constServiceQueueStatus.cs` — เก็บใน `TbQueueService.QueueStatus`

| ค่า (Value) | ความหมาย (ไทย) | คำอธิบาย |
|-------------|----------------|----------|
| `None` | รอยืนยันเข้าคิว | เพิ่งสร้างบัตร ยังไม่เช็คอิน |
| `CheckIn` | เข้าคิวแล้ว | เช็คอินแล้ว รอเรียก |
| `Inprogress` | รอดำเนินการ / กำลังให้บริการ | ถูกเรียกที่เคาน์เตอร์แล้ว |
| `Holding` | พักคิว | ถูกพักไว้ (มีเหตุผล) |
| `Completed` | เสร็จสิ้น | ให้บริการเสร็จ |
| `Cancelled` | ยกเลิกคิว | ถูกยกเลิก |

**แผนผังการเปลี่ยนสถานะ (State Transition):**

```
                    ┌──────────────────────────────────┐
                    │                                  │
  [None] ──checkin──► [CheckIn] ──next──► [Inprogress] ─┼─complete─► [Completed]
    │                    ▲                    │         │
    │ (AutoCheckin=true) │                    ├─pause──► [Holding] ──next──┐
    └────────────────────┘                    │              ▲            │
                                              │              └────────────┘
                                              └─forward─► [Completed] + สร้าง service path ใหม่ ([None])
```

---

## 4. Flow หลัก: วงจรชีวิตของคิว

ภาพรวมตั้งแต่ออกบัตรจนเสร็จสิ้น พร้อม method ที่ถูกเรียกและ event ที่ยิง:

```
1) สร้างคิว (CreateAsync)
   • CounterService.GetNextQueueNo() → สร้างเลขคิว เช่น "A-001"
   • สร้าง TbQueue + TbQueueService (status = None หรือ CheckIn ถ้า AutoCheckin)
   ⇒ SignalR: TriggerQueueChange
        │
        ▼
2) เช็คอิน (CheckinAsync)  [ข้ามได้ถ้า AutoCheckin]
   • ValidateCheckinAsync() ตรวจสอบสถานะก่อน
   • status = CheckIn, ตั้ง CheckinTime
   ⇒ SignalR: TriggerQueueChange
        │
        ▼
3) เรียกคิว (NextAsync) ที่เคาน์เตอร์
   • เลือกคิวเอง หรือ auto FIFO จากคิว CheckIn ที่เก่าสุด
   • status = Inprogress, ตั้ง StartTime + PointId
   • สร้าง TbQueueCalling (บันทึกการเรียก)
   ⇒ SignalR: TriggerQueueChange + TriggerQueueCalling
        │
        ▼
4) ระหว่างให้บริการ — เลือกได้หลายทาง:
   ┌── A) เสร็จสิ้น (CompleteAsync): status = Completed, ตั้ง EndTime, ลบ TbQueueCalling
   ├── B) ส่งต่อ (ForwardAsync): ปิด service เดิม (Completed) + สร้าง service ใหม่ (None) → กลับ step 2
   ├── C) พักคิว (PauseAsync): status = Holding, บันทึก TbQueuePauseHistory, ลบ TbQueueCalling
   └── D) เรียกซ้ำ (RecallAsync): ยิง event ซ้ำ ไม่เปลี่ยนสถานะ
```

---

## 5. Flow การออกบัตรคิว

**Endpoint:** `POST /api/queue/create` → `QueueService.CreateAsync(QueueCreateInputModel)`

**ขั้นตอน:**

1. รับ Input: `RoomId`, `ServiceTypeId`, `Note` (optional)
2. เรียก `CounterService.GetNextQueueNo(roomId)` เพื่อสร้างเลขคิวลำดับถัดไป
   - Key counter: `QueueNo_{RoomCode}_{วันที่}` (รีเซ็ตรายวัน)
   - รูปแบบ: `{RoomCode}-{เลขลำดับ}` เช่น `A-001`, `A-002`
   - วน CurrentRunNo จาก StartRunNo (1) → MaxRunNo (999) แล้วกลับมา 1
3. สร้าง `TbQueue` (QueueNo, QueueKey=GUID สำหรับ QR, Note)
4. สร้าง `TbQueueService`:
   - `QueueStatus = None` (ค่าเริ่มต้น)
   - **ถ้าห้องตั้ง `AutoCheckin = true`** → `QueueStatus = CheckIn` + ตั้ง `CheckinTime` ทันที
5. บันทึก DB
6. ⇒ **SignalR: `TriggerQueueChange`** (payload: pointId, roomId, serviceTypeId, queueNo, pointNo)

**ฝั่ง UI (Angular):**
- หน้า `QueueListComponent` (`/queue/list`) → กดปุ่ม "ออกบัตรใหม่"
- เปิด `AddQueueDialogComponent`: เลือกห้อง → เลือกประเภทบริการ → Submit
- เรียก `apiQueueCreatePost$Json()` → แสดงเลขคิวที่ได้

---

## 6. Flow การเช็คอิน

มี 2 แบบ:

### 6.1 เช็คอินแบบปกติ (Manual)
**Endpoint:** `POST /api/queue/checkin` → `CheckinAsync(CheckinInputModel)`

1. `ValidateCheckinAsync()` ตรวจสอบว่า:
   - คิวมีอยู่จริง (จาก QueueId, RoomId, ServiceTypeId)
   - สถานะยังเช็คอินได้ (ต้อง**ไม่ใช่** CheckIn / Completed / Holding / Inprogress / Cancelled)
2. อัปเดต `TbQueueService`: `status = CheckIn`, ตั้ง `CheckinTime`, `ServiceGroupId`
3. บันทึก DB → ⇒ **SignalR: `TriggerQueueChange`**

> มี endpoint `POST /api/queue/validate-checkin` สำหรับตรวจสอบอย่างเดียว (ไม่เปลี่ยนสถานะ)

### 6.2 เช็คอินผ่าน QR Code (Scan)
**Endpoint:** `GET /api/queue/get-queue-scancheckin` → `GetQueueScanCheckinAsync(ScanCheckinInputModel)`

1. ค้นหาคิวจาก `QueueNo` + วันที่ (เฉพาะวันนี้)
2. ตรวจสอบ RoomId / ServiceTypeId ตรงกัน
3. คืนรายละเอียดคิวให้ Kiosk/Mobile ยืนยัน → จากนั้นเรียก `CheckinAsync()` ตามปกติ

---

## 7. Flow การเรียกคิว

**Endpoint:** `POST /api/queue/next` → `NextAsync(QueueNextInputModel)`

**Input:** `PointId` (เคาน์เตอร์), `RoomId`, `ServiceTypeId`, `QueueId` (optional)

**ขั้นตอน:**

1. **ตรวจสอบ:** เคาน์เตอร์นี้กำลังเรียกคิวอื่นอยู่หรือไม่
   - ถ้ามี → error `INVALID_QUEUE_NO_DUPLICATE` (ห้ามเรียกซ้อน)
2. **เลือกคิวเป้าหมาย:**
   - ถ้าระบุ `QueueId` → เรียกคิวนั้น (ต้องอยู่สถานะ None / CheckIn / Holding)
   - ถ้าไม่ระบุ → auto เลือกคิว `CheckIn` ที่เก่าสุด (FIFO)
   - ถ้าไม่พบ → error `INVALID_QUEUE_NO_WAITING`
3. อัปเดต `TbQueueService`:
   - `QueueStatus = Inprogress`
   - ตั้ง `PointId`, `StartTime` (และ `CheckinTime` ถ้ายังไม่มี)
4. สร้าง `TbQueueCalling` (QueueNumber, PointId, CallerBy=ผู้เรียก, CallTime)
   - มี unique index กัน (QueueId, PointId) ซ้ำ
5. บันทึก DB
6. ⇒ **SignalR: `TriggerQueueChange` + `TriggerQueueCalling`**

**คืนค่า:** `QueueDataModel` (QueueId, QueueNo, RoomId, ServiceTypeId, PointId)

**ฝั่ง UI (เจ้าหน้าที่):**
- หน้า `QueueCallingComponent` (`/queue/calling`)
- เลือกเคาน์เตอร์ (Service Point) ผ่าน `QueueControlPanelComponent`
- กดปุ่ม "เรียกคิวถัดไป" → `callNextQueue()` → คิวแสดงในวิดเจ็ตด้านบน (กระพริบ 10 วินาที)

---

## 8. Flow ระหว่างให้บริการ

### 8.1 เสร็จสิ้น (Complete)
**Endpoint:** `POST /api/queue/complete` → `CompleteAsync(QueueCompleteInputModel)`

1. ลบ `TbQueueCalling` (ถ้ามี)
2. อัปเดต `TbQueueService`: `status = Completed`, ตั้ง `EndTime`
3. ⇒ **SignalR: `TriggerQueueChange`**

### 8.2 ส่งต่อ/โอนคิว (Forward / Transfer)
**Endpoint:** `POST /api/queue/forward` → `ForwardAsync(QueueForwardInputModel)`

**Input:** `QueueId`, `ServiceTypeId` (ปัจจุบัน), `ForwardToRoomId`, `ForwardToServiceTypeId`, `ForwardToNote`

1. ลบ `TbQueueCalling` เดิม
2. ปิด `TbQueueService` เดิม: `status = Completed`, `EndTime`, `Status = false` (archive)
3. สร้าง `TbQueueService` ใหม่ (QueueId เดิม):
   - ไปยัง `ForwardToRoomId` / `ForwardToServiceTypeId`
   - `status = None` (เริ่มใหม่ที่บริการปลายทาง), `Status = true`
4. ⇒ **SignalR: `TriggerQueueChange`**

> 💡 ทำให้คิวเดียวเดินผ่านได้หลายบริการ/ห้อง (Service Path) — หน้ารายละเอียดคิวแสดงเส้นทางทั้งหมด

**ฝั่ง UI:** `ForwardQueueDialogComponent` — เลือกห้อง/บริการปลายทาง + ใส่หมายเหตุ

### 8.3 พักคิว (Pause / Hold / Break)
**Endpoint:** `POST /api/queue/pause` → `PauseAsync(QueuePauseInputModel)`

**Input:** `QueueId`, `MainBreakReasonId`, `SubBreakReasonIdList`

1. สร้าง `TbQueuePauseHistory` (MainBreakReason, SubBreakReason, PauseStartTime)
2. อัปเดต `TbQueueService`: `status = Holding`, ตั้ง Note = เหตุผล
3. ลบ `TbQueueCalling` (ถ้ามี)
4. ⇒ **SignalR: `TriggerQueueChange`**

**ฝั่ง UI:** `PauseQueueDialogComponent` — เลือกเหตุผลหลัก + เหตุผลย่อย

### 8.4 เรียกซ้ำ (Recall)
**Endpoint:** `POST /api/queue/recall` → `RecallAsync(QueueRecallInputModel)`

1. ค้นหา `TbQueueCalling` จาก QueueId + PointId (ไม่พบ → error "ไม่พบคิว")
2. ยิง **SignalR: `TriggerQueueCalling`** ซ้ำ (เตือนให้เรียกเลขคิวเดิมอีกครั้ง)
3. **ไม่เปลี่ยนสถานะ** ใน DB

---

## 9. Flow Real-time ผ่าน SignalR

**Hub:** `EventHub` (`HAOS.Queue.Core/Hubs/EventHub.cs`) — URL: `/eventHub`

### Events ที่ระบบยิง

| Event | ยิงเมื่อ | Payload | ผู้รับ |
|-------|---------|---------|--------|
| `TriggerQueueChange` | สร้าง/เช็คอิน/เรียก/เสร็จ/ส่งต่อ/พักคิว | `{pointId, roomId, serviceTypeId, queueNo, pointNo}` | Clients.All |
| `TriggerQueueCalling` | เรียกคิว / เรียกซ้ำ | `{pointId, roomId, serviceTypeId, queueNo, pointNo}` | Clients.All |

### ฝั่ง Client (Angular)

- **Service:** `SignalRService` (`core/services/signalr.service.ts`)
- **Auto-Reconnect:** 2s → 5s → 10s
- เมื่อได้รับ `TriggerQueueChange` → เรียก:
  - `factDate()` — โหลดรายการคิวใหม่
  - `factDateSummary()` — โหลดตัวเลขสรุป
  - `getQueueCalling()` — โหลดคิวที่กำลังเรียก
- ใช้โดย: `QueueCallingComponent`, `QueueDetailComponent` (mobile), TV Display

```
   เจ้าหน้าที่กดเรียกคิว
          │
          ▼
   API: NextAsync()  ──► บันทึก DB
          │
          ▼
   EventHub ยิง TriggerQueueChange + TriggerQueueCalling
          │
   ┌──────┼───────────────┬───────────────┐
   ▼      ▼               ▼               ▼
 จอ TV  หน้าเจ้าหน้าที่อื่น  Mobile ลูกค้า   Dashboard
 (เด้งเลขคิว + เสียง)  (refresh list)  (อัปเดตตำแหน่ง)  (อัปเดตสถิติ)
```

---

## 10. Flow จอแสดงผล TV Display

**แอป:** `HAOS.QueueTV` (Angular แยกต่างหาก)

### Setup Flow (สำหรับผู้ดูแลจอ)

```
1) Login (/setup-access-token)
   • ใส่ username/password → เก็บ token ใน localStorage
        │
        ▼
2) ตั้งค่ารายการแสดงผล (/setup-work-list)
   • เลือกประเภทบริการ + ห้อง + เคาน์เตอร์ที่จะแสดง
   • บันทึกเป็น TvSelectedWorkAndRoom (localStorage)
        │
        ▼
3) หน้าจอหลัก (/screening-point-room)
```

### หน้าจอหลัก (`ScreeningPointRoomComponent`)

| ส่วน | แสดง |
|------|------|
| **ซ้าย** | Carousel เคาน์เตอร์ (สูงสุด 10/หน้า) — เลขเคาน์เตอร์, เลขคิวปัจจุบัน, ชื่อลูกค้า, **กระพริบ 10 วิ** เมื่อมีคิวใหม่ |
| **ขวา** | คิวที่ถูกข้าม (Skipped) — เลขเคาน์เตอร์ + เลขคิว (14/หน้า) |

**ฟีเจอร์:**
- 🔊 เสียงแจ้งเตือนเมื่อเรียกคิว (ปิดเสียงได้)
- อัปเดต Real-time ผ่าน SignalR `TriggerQueueChange`
- เก็บ config ใน localStorage (หาย/หมดอายุ → กลับหน้า setup)

---

## 11. Flow ฝั่งลูกค้า Mobile/Kiosk

**Controller:** `MobileController` — `GET /api/mobile/get-detail` (AllowAnonymous)

```
1) ลูกค้ารับบัตรคิว → ได้ queuekey (+ QR Code)
        │
        ▼
2) สแกน QR / เปิดลิงก์ → /mobile/detail?queuekey=...
        │
        ▼
3) เรียก GetQueueDetailByQueueKey(queuekey)
   คืน QueueDetailModel: เลขคิว, สถานะ, ตำแหน่งรอ, ประเภทบริการ, เคาน์เตอร์
        │
        ▼
4) ฟัง SignalR TriggerQueueChange → อัปเดตสถานะสด
```

**Components:**
- `QueueDetailComponent` — ติดตามสถานะ/ตำแหน่งคิว
- `QueueQrcodeComponent` — แสดง QR สำหรับเช็คอิน

---

## 12. Flow Authentication & Authorization

### 12.1 Login Flow
**Endpoint:** `POST /api/auth/sign-in` → `AuthenService.SignInAsync()`

```
1) ส่ง username/password + RememberMe
        │
        ▼
2) ตรวจสอบ username มีอยู่ + VerifyPassword (bcrypt)
   • นับ FailedLoginCount; ถ้าเกิน MaxFailedLoginCount → ล็อกบัญชี
   • ตรวจ User.Status & UserGroup.Status = true
        │
        ▼
3) สำเร็จ → ออก Token:
   • Access Token (JWT, HS256) — claims: UserId, Username, Email, fullName, roles(UserGroupId), remember
     อายุ: TokenLifetimeHours (หรือ RememberTokenLifetimeHours ถ้า RememberMe)
   • Refresh Token (32 bytes base64) เก็บใน TbUserRefreshToken
     อายุ: Token + 12 ชม.
        │
        ▼
4) คืน: AccessToken, RefreshToken, FromFirstLogin (ForceChangePassword), UserInfo (permissions)
```

### 12.2 Refresh Token Flow
**Endpoint:** `POST /api/auth/refresh-token`
- ตรวจ refresh token มีอยู่ + ยังไม่หมดอายุ → ออก AccessToken + RefreshToken ใหม่ (ตัวเก่าใช้ไม่ได้)

### 12.3 Authorization (PermissionAuthorize)
- ใช้ attribute `[PermissionAuthorize(["permission.key"])]` บน controller action
- `AuthorizeActionFilter` ดักก่อนทำงาน:
  1. ตรวจ JWT → ดึง UserGroupId จาก claim `roles`
  2. `CheckHasPermission(userGroupId, requestedPermissions)` เทียบกับ `TbUserGroupAuthorize` (HasPermission=true)
  3. ไม่มีสิทธิ์ → 403 Forbidden
- **ตัวอย่าง permission keys:** `dashboard`, `queue.calling`, `queuecard.issue_ticket`, `admin.users`, `admin.usergroups`, `master.service`, `master.room`, `master.breakreason`, `master.timeservice`, `report.report`

**กลุ่มผู้ใช้ (User Groups):** SuperAdmin (ทุกสิทธิ์), PowerUser, User

---

## 13. Flow ข้อมูลหลัก & รายงาน

### 13.1 Master Data
- **ServiceType** (`/api/servicetype`) — ประเภทบริการ [perm: `master.service`]
- **Room** (`/api/room`) — ห้อง/แผนก/จุดบริการ [perm: `master.room`]
- **QueueBreakReason** (`/api/queuebreakreason`) — เหตุผลพักคิว [perm: `master.breakreason`]
- **Configuration** (`/api/configuration`) — ค่า threshold เวลารอ Dashboard [perm: `master.timeservice`]
  - `Dashboard_Queue/Room/Service_NormalValue` & `_MediumValue` (เกณฑ์สีเตือนเวลารอ)
- **Master endpoints** (`/master/*`) — dropdown: usergroups, servicetypes, breakreasons, rooms, servicegroup ฯลฯ

### 13.2 Report
**Endpoint:** `POST /api/form/ExportReportPdf` → `ReportService.GetReport()`
1. โหลด SQL จาก `/Reports/{sqlFileName}.sql` → แทนค่า `@Parameter`
2. โหลด template Stimulsoft `.mrt`
3. รัน SQL + bind ข้อมูล → render → export PDF (culture th-TH)
4. คืน PDF bytes + filename

---

## 14. สรุป API Endpoints

### Queue (`/api/queue`)
| Method | Path | หน้าที่ |
|--------|------|--------|
| POST | `/create` | ออกบัตรคิว |
| POST | `/checkin` | เช็คอิน |
| POST | `/validate-checkin` | ตรวจสอบเช็คอิน |
| POST | `/next` | เรียกคิวถัดไป |
| POST | `/recall` | เรียกซ้ำ |
| POST | `/forward` | ส่งต่อคิว |
| POST | `/pause` | พักคิว |
| POST | `/complete` | เสร็จสิ้น |
| GET | `/get-detail` | รายละเอียดคิว + service path |
| GET | `/get-queue-calling` | คิวที่กำลังเรียก |
| GET | `/get-queue-scancheckin` | สแกนเช็คอิน QR |
| GET | `/get-queue-summary` | สรุปคิวตามห้อง |
| GET | `/get-service-points` | รายการเคาน์เตอร์ |
| GET | `/get-dashboard` | ข้อมูล Dashboard |
| POST | `/set-queuegroup-status` | เปิด/ปิดกลุ่มคิว |

### Auth (`/api/auth`)
| Method | Path | หน้าที่ |
|--------|------|--------|
| POST | `/sign-in` | เข้าสู่ระบบ |
| POST | `/refresh-token` | ต่ออายุ token |
| GET | `/get-currentuser` | ข้อมูลผู้ใช้ปัจจุบัน |
| POST | `/validate-password` | ตรวจรหัสผ่าน |
| POST | `/reset-password` | เปลี่ยนรหัสผ่าน |

### อื่น ๆ
- **Mobile:** `GET /api/mobile/get-detail`
- **Master:** `GET /master/*`
- **ServiceType/Room/QueueBreakReason:** CRUD มาตรฐาน
- **Users:** `/api/users` [perm: `admin.users`]
- **UserGroup:** `/api/usergroup` [perm: `admin.usergroups`]
- **Configuration:** `/api/configuration`
- **Form/Report:** `POST /api/form/ExportReportPdf`
- **System:** `GET /api/system/app-info`, `/api/system/mock-data`

---

## 15. ตาราง Database หลัก

| Entity | ตาราง | ฟิลด์สำคัญ | หน้าที่ |
|--------|-------|-----------|--------|
| Queue | `TbQueue` | QueueId, QueueNo, QueueKey(GUID), Note | บัตรคิวหลัก |
| Queue Service | `TbQueueService` | QueueServiceId, QueueId, RoomId, ServiceTypeId, **QueueStatus**, PointId, CheckinTime, StartTime, EndTime, Status | คำขอบริการ (1 คิวมีได้หลาย service path) |
| Queue Calling | `TbQueueCalling` | ActiveCallId, QueueId, PointId, QueueNumber, CallerBy, CallTime | บันทึกการเรียกที่กำลัง active (unique: QueueId+PointId) |
| Pause History | `TbQueuePauseHistory` | QueueServiceId, MainBreakReason, SubBreakReason, PauseStartTime | ประวัติการพักคิว |
| Counter | `TbCounter` | CounterKey, CurrentRunNo, StartRunNo, MaxRunNo, StringFormat | ตัวสร้างเลขคิว (รีเซ็ตรายวัน) |
| Queue Group Active | `TbQueueGroupActive` | RoomId, ServiceTypeId, Status | เปิด/ปิดกลุ่มคิว |
| User | `TbUser` | Id, Username, Password(bcrypt), UserGroupId, IsLocked, FailedLoginCount | ผู้ใช้ |
| User Group | `TbUserGroup` | Id, Name, Status | กลุ่มสิทธิ์ |
| User Authorize | `TbUserGroupAuthorize` | UserGroupId, MatrixKey, HasPermission | การกำหนดสิทธิ์ |
| Refresh Token | `TbUserRefreshToken` | RefreshToken, UserId, ExpiryTime | เก็บ refresh token |
| Service Type | `TbmServiceType` | ServiceTypeId, Name, Status | ประเภทบริการ |
| Room | `TbmRoom` | RoomId, Code, Name, Status | ห้อง/จุดบริการ |
| Configuration | `TbConfigurations` | KeyName, Value | ตั้งค่าระบบ |

---

## ไฟล์อ้างอิงสำคัญ (Source Code)

| สิ่งที่ต้องการดู | ตำแหน่งไฟล์ |
|-----------------|-------------|
| Queue Controller | `HAOS.Queue.WebApi/Controllers/QueueController.cs` |
| Queue Logic | `HAOS.Queue.Business/Service/Queue/QueueService.cs` |
| Counter (เลขคิว) | `HAOS.Queue.Business/Service/Queue/CounterService.cs` |
| สถานะคิว | `HAOS.Queue.Core/.../constServiceQueueStatus.cs` |
| SignalR Hub | `HAOS.Queue.Core/Hubs/EventHub.cs` |
| Auth Logic | `HAOS.Queue.Business/Service/User/AuthenService.cs` |
| หน้าเจ้าหน้าที่ | `WebClient/src/app/pages/queue/queue-calling/queue-calling.component.ts` |
| หน้าออกบัตร | `WebClient/src/app/pages/queue/queue-list/queue-list.component.ts` |
| SignalR Service | `WebClient/src/app/core/services/signalr.service.ts` |
| จอ TV | `QueueTV/WebClient/.../screening-point-room.component.ts` |

---

*เอกสารนี้สร้างจากการวิเคราะห์ source code ของ HAOS Queue System*
