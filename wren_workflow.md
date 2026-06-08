# Data Mapping: `dmp_public_device` → StarRocks Tables

Tài liệu mô tả cách dữ liệu từ nguồn DMP (`dmp_public_device`) được map sang các bảng Raw, Staging và Golden trong local dev stack.

**Nguồn gốc logic:** `database/mock_data/gen_mock.py`  
**Schema tham chiếu:** `database/Starrock_Schema.txt`  
**CSV nguồn:** `database/mock_data/dmp_public_device.csv`

---

## 1. Tổng quan luồng dữ liệu

```
dmp_public_device.csv
        │
        ▼ load_devices()  — lấy 1 device / type (36 loại)
        │
        ├──► raw_dmp_public_device ──────────► stg_dmp_devices
        │
        ├──► device_profile_id ─────────────► raw_dmp_public_device_profile
        │                                      └──► stg_dmp_device_profiles
        │
        ├──► id + name + type ──────────────► stg_dmp_evt_connectivity (theo giờ)
        │                                      └──► stg_dmp_device_status_events
        │
        └──► relation (DEVICE → ASSET) ─────► raw_dmp_public_relation
                                               └──► stg_dmp_relations

Assets / Asset Profiles (synthetic, không từ CSV)
        │
        ├──► raw_dmp_public_asset / raw_dmp_public_asset_profile
        ├──► stg_dmp_assets / stg_dmp_asset_profiles
        └──► dim_asset / dim_asset_profile (Golden)

Parking (không liên quan dmp_public_device)
        └──► raw_parking_db_vehicle_histories → stg_vehicle_histories
```

---

## 2. Bước đọc nguồn: `load_devices()`

CSV gốc có **~14,000+ dòng** (nhiều device cùng `type`). Mock chỉ lấy **1 dòng đầu tiên mỗi `type`** → **36 devices**.

| CSV column | Dùng trong mock | Ghi chú |
|------------|-----------------|---------|
| `id` | `Device.id` | UUID, giữ nguyên |
| `name` | `Device.name` | Tên thiết bị |
| `type` | `Device.dtype` | Loại thiết bị (dedup key) |
| `label` | `Device.label` | Có thể rỗng |
| `device_profile_id` | `Device.device_profile_id` | Nếu rỗng → sinh `uid("profile-{type}")` |
| `customer_id` | — | Ghi đè bằng hằng `CUSTOMER_ID` |
| `tenant_id` | — | Ghi đè bằng hằng `TENANT_ID` |
| `created_time` | — | Ghi đè bằng `CREATED_MS = 1778500000000` |
| `processing_day` | — | Ghi đè bằng `"2026-06-01"` |
| `additional_info`, `device_data`, `firmware_id`, `software_id`, `external_id` | — | Để `NULL` trong mock |

---

## 3. Map cột: CSV → Raw → Staging

### 3.1 Device

| `dmp_public_device` (CSV) | `raw_dmp_public_device` | `stg_dmp_devices` |
|---------------------------|-------------------------|-------------------|
| `id` | `id` | `device_id` |
| `created_time` | `created_time` (bigint ms) | `created_at` (datetime) |
| `additional_info` | `additional_info` | `additional_info` |
| `customer_id` | `customer_id` | `customer_id` |
| `device_profile_id` | `device_profile_id` | `device_profile_id` |
| `device_data` | `device_data` | `device_data` |
| `type` | `type` | `type` |
| `name` | `name` | `name` |
| `label` | `label` | `label` |
| `tenant_id` | `tenant_id` | `tenant_id` |
| `firmware_id` | `firmware_id` | `firmware_id` |
| `software_id` | `software_id` | `software_id` |
| `external_id` | `external_id` | `external_id` |
| `version` | `version` (= 1) | `version` (= 1) |
| `processing_day` | `processing_day` (varchar) | `processing_day` (date) |
| — | — | `_dbt_loaded_at` (metadata dbt) |

**Catalog:**

- Raw: `sdp_dev_hive_catalog.sdp_raw`
- Staging: `sdp_dev_iceberg_catalog.sdp_staging`

### 3.2 Device Profile (suy ra từ `device_profile_id`)

Mỗi `device_profile_id` duy nhất trong 36 devices → 1 profile (~35 profiles).

| Nguồn | `raw_dmp_public_device_profile` | `stg_dmp_device_profiles` |
|-------|--------------------------------|---------------------------|
| `device_profile_id` | `id` | `device_profile_id` |
| `type` (từ device mẫu) | `type`, `name` (= "Profile {type}") | `type`, `name` |
| Synthetic | `transport_type` = MQTT nếu type chứa "bms", else DEFAULT | giống Raw |
| Synthetic | `provision_type` = "DEFAULT" | giống Raw |
| Synthetic | `description` = "Device profile for {type}" | giống Raw |
| Synthetic | `is_default` = true cho profile đầu tiên | giống Raw |
| Hằng số | `tenant_id`, `created_time`, `version`, `processing_day` | + `created_at`, `_dbt_loaded_at` |

### 3.3 Relation (DEVICE → ASSET, synthetic)

Không có trong CSV. Mock gán mỗi device vào 1 asset (round-robin 10 assets).

| Trường | `raw_dmp_public_relation` | `stg_dmp_relations` |
|--------|---------------------------|---------------------|
| `Device.id` | `from_id` | `from_id` |
| Hằng `"DEVICE"` | `from_type` | `from_type` |
| `Asset.id` (synthetic) | `to_id` | `to_id` |
| Hằng `"ASSET"` | `to_type` | `to_type` |
| Hằng `"COMMON"` | `relation_type_group` | `relation_type_group` |
| Hằng `"Contains"` | `relation_type` | `relation_type` |

**Join path:** `stg_dmp_devices.device_id` → `stg_dmp_relations.from_id` → `stg_dmp_assets.asset_id`

---

## 4. Bảng phụ thuộc device (synthetic events)

### 4.1 `stg_dmp_evt_connectivity`

Sinh **mỗi giờ × mỗi device** (2026-05-08 → 2026-06-08) → ~27,648 dòng.

| Device field | `stg_dmp_evt_connectivity` | Ghi chú |
|--------------|---------------------------|---------|
| — | `msgtype` | CONNECT / DISCONNECT (random theo giờ) |
| `id` | `deviceid` | |
| TENANT_ID | `tenantid` | |
| CUSTOMER_ID | `customerid` | |
| `name` | `devicecode` | |
| — | `status` | ONLINE / OFFLINE |
| — | `offlinereason`, `qualityscore`, `icmpreachable` | Synthetic |
| — | `ts` | Epoch ms theo time slot |
| — | `processing_day` | `YYYY-MM-DD` |

### 4.2 `stg_dmp_device_status_events`

Mỗi 6 giờ / device, hoặc khi status đổi → ~444 dòng.

| Device field | `stg_dmp_device_status_events` |
|--------------|-------------------------------|
| `id` | `device_id` |
| `name` | `device_code` |
| `type` | `device_type` |
| TENANT_ID | `tenant_id` |
| — | `event_id`, `event_type`, `current_status`, `previous_status`, `event_time`, `event_date`, … |

---

## 5. Asset / Golden — không map trực tiếp từ device

Assets và asset profiles **không có trong CSV**. Mock sinh 10 assets + 3 asset profiles, rồi gán device qua `raw_dmp_public_relation`.

| Bảng Golden | Nguồn | Liên kết với device |
|-------------|-------|---------------------|
| `dim_asset_profile` | 3 asset profiles synthetic | Gián tiếp qua asset |
| `dim_asset` | 10 assets synthetic (flatten profile) | Qua `stg_dmp_relations` |
| `dim_date` | Calendar 32 ngày | Không liên quan device |

**Device không có bảng dimension riêng** (`dim_device`) trong schema hiện tại.

---

## 6. Bảng KHÔNG liên quan `dmp_public_device`

| Bảng | Nguồn |
|------|-------|
| `raw_parking_db_vehicle_histories` | Synthetic parking (120 rows) |
| `stg_vehicle_histories` | Cùng nguồn parking |

---

## 7. Sơ đồ quan hệ (ER đơn giản)

```
device_profile ──< device ──< relation >── asset ──< asset_profile
                    │
                    ├──< stg_dmp_evt_connectivity
                    └──< stg_dmp_device_status_events

asset ──► dim_asset (join profile)
asset_profile ──► dim_asset_profile
```

---

## 8. Ví dụ query join

```sql
-- Device + profile + asset qua relation
SELECT
  d.device_id,
  d.name          AS device_name,
  d.type          AS device_type,
  dp.name         AS profile_name,
  a.name          AS asset_name,
  r.relation_type
FROM sdp_dev_iceberg_catalog.sdp_staging.stg_dmp_devices d
JOIN sdp_dev_iceberg_catalog.sdp_staging.stg_dmp_device_profiles dp
  ON d.device_profile_id = dp.device_profile_id
JOIN sdp_dev_iceberg_catalog.sdp_staging.stg_dmp_relations r
  ON d.device_id = r.from_id AND r.from_type = 'DEVICE'
JOIN sdp_dev_iceberg_catalog.sdp_staging.stg_dmp_assets a
  ON r.to_id = a.asset_id AND r.to_type = 'ASSET'
LIMIT 10;
```

---

## 9. Khác biệt Production vs Local Mock

| Khía cạnh | Production (DMP thật) | Local mock |
|-----------|----------------------|------------|
| Số device | Hàng nghìn (mọi row CSV) | 36 (1/type) |
| Asset / Relation | Từ DMP `public.asset`, `public.relation` | Synthetic trong `gen_mock.py` |
| Connectivity events | Stream telemetry thật | Random theo giờ |
| ETL | dbt models | `gen_mock.py` sinh SQL trực tiếp |
| S3 path | `s3://sdp-dev-raw-...` | `s3://warehouse/...` (MinIO) |

---

## 10. File liên quan

| File | Vai trò |
|------|---------|
| `mock_data/dmp_public_device.csv` | Nguồn device gốc |
| `mock_data/gen_mock.py` | Logic mapping + sinh data |
| `mock_data/schema_gen.py` | DDL 17 bảng |
| `Deploy_database/local-dev/init/01_schema.sql` | Schema deploy |
| `Deploy_database/local-dev/init/02_mock_data.sql` | Data deploy |
| `mock_devices_list.md` | Danh sách 36 device đại diện |
