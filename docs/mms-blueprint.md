# Maintenance Management System (MMS) Blueprint

เอกสารนี้ออกแบบระบบ Web Application สำหรับทีมช่างซ่อมเครื่องจักร (Boomlift, Scissor Lift, ระบบไฟฟ้า, ไฮดรอลิก) โดยเน้นการใช้งานหน้างานผ่านมือถือ

## 1) Recommended Architecture

- **Frontend**: Next.js 14 (App Router) + TypeScript + Tailwind CSS
- **Backend**: Next.js Route Handlers (`app/api/*`) + Service Layer
- **Database**: PostgreSQL (แนะนำใช้ Supabase Postgres)
- **Auth**: Google Login (NextAuth/Auth.js)
- **Storage**: Supabase Storage (เก็บภาพอะไหล่ก่อน/หลังเปลี่ยน)
- **Analytics**: SQL View + Materialized View สำหรับ Dashboard

---

## 2) Project Folder Structure (Next.js 14)

```txt
mms-app/
├─ src/
│  ├─ app/
│  │  ├─ (auth)/
│  │  │  ├─ login/page.tsx
│  │  │  └─ callback/route.ts
│  │  ├─ dashboard/page.tsx
│  │  ├─ machines/
│  │  │  ├─ page.tsx
│  │  │  ├─ [machineId]/page.tsx
│  │  │  └─ new/page.tsx
│  │  ├─ maintenance/
│  │  │  ├─ page.tsx
│  │  │  ├─ [recordId]/page.tsx
│  │  │  └─ new/page.tsx
│  │  ├─ spare-parts/
│  │  │  ├─ page.tsx
│  │  │  └─ replacement/new/page.tsx
│  │  ├─ history/page.tsx
│  │  ├─ api/
│  │  │  ├─ machines/route.ts
│  │  │  ├─ machines/[id]/route.ts
│  │  │  ├─ maintenance/route.ts
│  │  │  ├─ maintenance/[id]/route.ts
│  │  │  ├─ replacements/route.ts
│  │  │  ├─ prechecks/route.ts
│  │  │  ├─ postchecks/route.ts
│  │  │  └─ analytics/dashboard/route.ts
│  │  ├─ layout.tsx
│  │  └─ page.tsx
│  ├─ components/
│  │  ├─ forms/
│  │  │  ├─ machine-form.tsx
│  │  │  ├─ maintenance-form.tsx
│  │  │  ├─ replacement-form.tsx
│  │  │  ├─ precheck-form.tsx
│  │  │  └─ postcheck-form.tsx
│  │  ├─ dashboard/
│  │  │  ├─ kpi-cards.tsx
│  │  │  ├─ part-frequency-chart.tsx
│  │  │  ├─ machine-failure-chart.tsx
│  │  │  └─ cost-trend-chart.tsx
│  │  └─ ui/
│  ├─ lib/
│  │  ├─ db.ts
│  │  ├─ auth.ts
│  │  ├─ validators/
│  │  ├─ repositories/
│  │  └─ services/
│  ├─ types/
│  └─ config/
├─ prisma/
│  ├─ schema.prisma
│  └─ migrations/
├─ docs/
│  └─ mms-blueprint.md
└─ package.json
```

---

## 3) Database Schema (PostgreSQL)

> ครอบคลุมทั้ง 8 modules ตาม requirement

### 3.1 Master tables

```sql
create table companies (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  created_at timestamptz not null default now()
);

create table locations (
  id uuid primary key default gen_random_uuid(),
  company_id uuid not null references companies(id),
  name text not null,
  address text,
  created_at timestamptz not null default now()
);

create table technicians (
  id uuid primary key default gen_random_uuid(),
  user_id text unique, -- map จาก auth provider
  full_name text not null,
  phone text,
  skill_tags text[],
  active boolean not null default true,
  created_at timestamptz not null default now()
);

create table machine_models (
  id uuid primary key default gen_random_uuid(),
  brand text,
  model_code text not null,
  machine_type text not null, -- boomhift/scissor/electric/hydraulic
  created_at timestamptz not null default now()
);
```

### 3.2 Module 1: Machine Database

```sql
create table machines (
  id uuid primary key default gen_random_uuid(),
  machine_code text unique not null,
  model_id uuid not null references machine_models(id),
  serial_number text unique not null,
  company_id uuid not null references companies(id),
  location_id uuid references locations(id),
  working_hours numeric(10,1) not null default 0,
  status text not null default 'active',
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create index idx_machines_company on machines(company_id);
create index idx_machines_model on machines(model_id);
```

### 3.3 Module 2: Maintenance Record

```sql
create table maintenance_records (
  id uuid primary key default gen_random_uuid(),
  machine_id uuid not null references machines(id),
  symptom text not null,
  repair_date date not null,
  inspector_technician_id uuid references technicians(id),
  inspection_method text,
  inspection_result text,
  root_cause text,
  downtime_minutes int,
  created_by uuid references technicians(id),
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create index idx_maint_machine_date on maintenance_records(machine_id, repair_date desc);
```

### 3.4 Module 4: Pre-Check ก่อนเปลี่ยนอะไหล่

```sql
create table precheck_items (
  id uuid primary key default gen_random_uuid(),
  code text unique not null,
  name text not null,
  group_name text not null,
  required boolean not null default true
);

create table precheck_results (
  id uuid primary key default gen_random_uuid(),
  maintenance_record_id uuid not null references maintenance_records(id) on delete cascade,
  item_id uuid not null references precheck_items(id),
  status text not null check (status in ('pass','fail','na')),
  measured_value text,
  note text,
  checked_by uuid references technicians(id),
  checked_at timestamptz not null default now(),
  unique (maintenance_record_id, item_id)
);
```

### 3.5 Module 3: Spare Part Replacement

```sql
create table spare_parts (
  id uuid primary key default gen_random_uuid(),
  part_number text unique not null,
  part_name text not null,
  unit text not null default 'pcs',
  standard_cost numeric(12,2) not null default 0,
  created_at timestamptz not null default now()
);

create table spare_part_replacements (
  id uuid primary key default gen_random_uuid(),
  maintenance_record_id uuid not null references maintenance_records(id) on delete cascade,
  spare_part_id uuid not null references spare_parts(id),
  quantity numeric(10,2) not null,
  replacement_reason text not null,
  pre_replace_check_method text,
  pre_replace_check_result text,
  post_replace_result text,
  part_cost numeric(12,2) not null default 0,
  replaced_by uuid references technicians(id),
  replaced_at timestamptz not null default now()
);

create table replacement_images (
  id uuid primary key default gen_random_uuid(),
  replacement_id uuid not null references spare_part_replacements(id) on delete cascade,
  image_type text not null check (image_type in ('before','after')),
  image_url text not null,
  uploaded_at timestamptz not null default now()
);
```

### 3.6 Module 5: Post-Check หลังเปลี่ยน

```sql
create table postcheck_results (
  id uuid primary key default gen_random_uuid(),
  maintenance_record_id uuid not null unique references maintenance_records(id) on delete cascade,
  operation_normal boolean,
  voltage_value numeric(10,2),
  hydraulic_value numeric(10,2),
  field_test_result text,
  note text,
  checked_by uuid references technicians(id),
  checked_at timestamptz not null default now()
);
```

### 3.7 Module 6/7/8: Analytics + History views

```sql
create view v_part_failure_frequency as
select sp.part_number, sp.part_name,
       count(*) as replacement_count,
       sum(r.quantity) as total_quantity,
       sum(r.part_cost) as total_cost
from spare_part_replacements r
join spare_parts sp on sp.id = r.spare_part_id
group by sp.part_number, sp.part_name;

create view v_machine_failure_frequency as
select m.machine_code, mm.model_code,
       count(mr.id) as maintenance_count,
       avg(coalesce(mr.downtime_minutes,0)) as avg_downtime_min
from maintenance_records mr
join machines m on m.id = mr.machine_id
join machine_models mm on mm.id = m.model_id
group by m.machine_code, mm.model_code;

create view v_root_cause_summary as
select coalesce(root_cause, 'unknown') as root_cause,
       count(*) as occurrences
from maintenance_records
group by coalesce(root_cause, 'unknown');
```

---

## 4) Example API (Next.js Route Handlers)

### 4.1 Create Maintenance Record

`POST /api/maintenance`

```json
{
  "machineId": "uuid",
  "symptom": "ไม่ยกขึ้น",
  "repairDate": "2026-03-10",
  "inspectorTechnicianId": "uuid",
  "inspectionMethod": "วัดแรงดัน + ตรวจรีเลย์",
  "inspectionResult": "รีเลย์ค้าง",
  "rootCause": "relay_fail",
  "downtimeMinutes": 120
}
```

Response `201`

```json
{
  "id": "uuid",
  "message": "Maintenance record created"
}
```

### 4.2 Submit Pre-check

`POST /api/prechecks`

```json
{
  "maintenanceRecordId": "uuid",
  "items": [
    { "code": "POWER_IN", "status": "pass", "measuredValue": "24V" },
    { "code": "FUSE", "status": "pass" },
    { "code": "RELAY", "status": "fail", "note": "ค้าง" }
  ]
}
```

### 4.3 Record Spare Part Replacement

`POST /api/replacements`

```json
{
  "maintenanceRecordId": "uuid",
  "sparePartId": "uuid",
  "quantity": 1,
  "replacementReason": "รีเลย์เสีย",
  "preReplaceCheckMethod": "วัดความต้านทาน",
  "preReplaceCheckResult": "open circuit",
  "postReplaceResult": "ทำงานปกติ",
  "partCost": 950,
  "images": [
    { "imageType": "before", "imageUrl": "https://.../before.jpg" },
    { "imageType": "after", "imageUrl": "https://.../after.jpg" }
  ]
}
```

### 4.4 Dashboard Summary

`GET /api/analytics/dashboard?from=2026-03-01&to=2026-03-31`

```json
{
  "totalMaintenanceJobs": 184,
  "topUsedParts": [
    { "partNumber": "RLY-24V-001", "count": 22 }
  ],
  "frequentFailedMachines": [
    { "machineCode": "BL-009", "count": 11 }
  ],
  "sparePartCost": 245000.5,
  "rootCauseStats": [
    { "rootCause": "relay_fail", "count": 37 }
  ]
}
```

---

## 5) Example UI Forms (Mobile-first)

### 5.1 Maintenance Form (Step Form)

- Step 1: เลือกเครื่อง (ค้นหาโดย Machine code / Serial)
- Step 2: บันทึกอาการเสีย + วันที่ซ่อม
- Step 3: เลือกช่าง + วิธีตรวจสอบ + ผลตรวจ
- Step 4: เลือกสาเหตุการเสีย (Dropdown + Free text)

**UX แนะนำ:**
- ปุ่มใหญ่ (`h-12`) กดง่ายด้วยนิ้ว
- มี quick chips สำหรับอาการเสียที่พบบ่อย
- มี autosave ทุก 10 วินาที

### 5.2 Pre-check Form

- แสดง checklist แบบ card ทีละข้อ
- แต่ละข้อมีตัวเลือก Pass / Fail / N/A
- ช่องกรอกค่า เช่น Voltage, Resistance
- ถ้า `Fail` ให้บังคับกรอก Note

### 5.3 Replacement Form

- เลือกอะไหล่จาก Part Number
- ระบุจำนวน + เหตุผลการเปลี่ยน
- อัปโหลดภาพก่อน/หลัง (อย่างน้อย 1 ภาพ)
- ปุ่ม `บันทึกและไป Post-check`

### 5.4 Post-check Form

- สวิตช์ "เครื่องทำงานปกติ"
- ค่าแรงดัน / ค่าไฮดรอลิก
- ผลทดสอบใช้งานจริง
- ถ้าทดสอบไม่ผ่าน ให้สร้าง Follow-up Work Order อัตโนมัติ

---

## 6) Dashboard ผู้จัดการ (Module 8)

### KPI Cards

1. จำนวนงานซ่อมทั้งหมด
2. MTTR เฉลี่ย (Mean Time to Repair)
3. ค่าใช้จ่ายอะไหล่รวม
4. เครื่องที่ซ่อมซ้ำสูงสุด

### Charts

- Top 10 อะไหล่ที่ใช้มากที่สุด
- เครื่องรุ่นที่มีปัญหาบ่อย
- Pareto สาเหตุการเสีย (80/20)
- แนวโน้มค่าใช้จ่ายอะไหล่รายสัปดาห์/เดือน

### Filters

- วันที่ (รายวัน/สัปดาห์/เดือน)
- บริษัท
- สถานที่
- ประเภทเครื่องจักร
- ช่างผู้รับผิดชอบ

---

## 7) Suggested API Permission Model

- **Technician**: create/update maintenance, pre/post check, replacements
- **Supervisor**: approve root cause, adjust cost, close job
- **Manager**: read-only dashboard + export
- **Admin**: master data management

ใช้ Row Level Security (RLS) หากอยู่บน Supabase เพื่อแยกข้อมูลตามบริษัท/ทีม

---

## 8) Rollout Plan (Recommended)

- **Phase 1 (4-6 weeks)**: Modules 1,2,3,4,5 + basic history
- **Phase 2 (2-3 weeks)**: Analytics (6,8) + export PDF/Excel
- **Phase 3 (2 weeks)**: Predictive alert (แจ้งเตือนอะไหล่เสี่ยงเสียซ้ำ)

---

## 9) Additional Notes for Field Technicians

- รองรับโหมด offline draft (กรอกได้แม้สัญญาณอ่อน)
- มีภาษาไทยเต็มรูปแบบ
- ใช้กล้องมือถือถ่ายภาพอะไหล่ทันที
- รองรับ dark mode สำหรับงานกลางคืน
