# 主机与磁盘信息格式（home_cloud）

本文档定义由 `home_cloud` 项目生成（采集）的**主机（节点）与磁盘信息**的数据格式。这些信息由 home_cloud 中的脚本采集，并通过 RemoteStorage（`ufs-nodes/`）或本地磁盘（如 `O:\app_data\disks`）存储，供其它应用（如 vue-element-admin 的节点页面、监控看板）读取展示。

## 数据来源

| 脚本 / 模块 | 函数 | 生成内容 | 存储位置 |
|---|---|---|---|
| `scripts/disk_info.py` | `get_disk_info()` / `save_to_json()` | 单块磁盘的 SMART 信息（含所属宿主信息） | RemoteStorage `ufs-nodes/{serial}.json` |
| `scripts/save_disk_info_to_json.py` | `save_disk_info_to_json()` | 单主机的主机信息 + 物理/逻辑磁盘 + 分区映射（含多主机合并） | 本地 `O:\app_data\disks\*.json` |
| `remote_storage/client.py` | `update_my_status()` | 主机/节点在线状态心跳 | RemoteStorage `ufs-nodes/{uuid}.json` |

> 与 [ufs-nodes-remotestorage-format.md](./ufs-nodes-remotestorage-format.md) 的关系：那份文档描述的是 `vue-element-admin` 的 `nodes.vue` 页面写入的**节点控制对象**（含 `capabilities` / `inbox` / `mqtt` 等控制字段，键名为节点 `uuid`）；本文档描述的是 **home_cloud 采集的主机/磁盘遥测数据**（主机状态键名为设备 `uuid`、磁盘文件键名为磁盘 `serial`）。两者可共存于 `ufs-nodes/` 目录，字段互不冲突。

---

## 1. 主机/节点状态（心跳）

由 `update_my_status()` 写入 RemoteStorage，路径 `ufs-nodes/{uuid}.json`，`uuid` 来自 `UfsConfig.get_device_uuid()`。

### 数据结构

```json
{
  "uuid": "550e8400-e29b-41d4-a716-446655440000",
  "name": "m10",
  "status": "active",
  "local_ip": "192.168.1.20",
  "last_access": 1719876543210,
  "last_access_str": "2024-07-02 12:09:03"
}
```

### 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `uuid` | `string` | 是 | 设备唯一标识（UUID v4），来自 `UfsConfig.get_device_uuid()` |
| `name` | `string` | 是 | 设备名，来自 `UfsConfig.get_device_name()` |
| `status` | `string` | 否 | 节点状态，采集时固定为 `"active"` |
| `local_ip` | `string` | 否 | 本机局域网 IP，来自 `get_local_ip()` |
| `last_access` | `number` | 是 | 最后访问时间，毫秒时间戳（`time.time() * 1000`） |
| `last_access_str` | `string` | 否 | 可读的最后访问时间，格式 `"YYYY-MM-DD HH:MM:SS"` |

---

## 2. 磁盘信息（SMART）

由 `disk_info.py` 采集（依赖 `pySMART.DeviceList`），经 `save_to_json()` 写入 RemoteStorage `ufs-nodes/{serial}.json`，文件名取磁盘序列号。

### 数据结构

```json
{
  "uuid": "9e630b28-f52b-4b98-a9bc-e2e9ebe601bc",
  "host_uuid": "550e8400-e29b-41d4-a716-446655440000",
  "host_name": "m10",
  "type": "disk",
  "status": "active",
  "last_access": 1719876543210,
  "name": "m10/\\\\.\\PHYSICALDRIVE0",
  "model": "Samsung SSD 980 PRO 1TB",
  "serial": "S5GXNF0R123456",
  "capacity": "931.51 GB",
  "firmware": "1B2QGXA7",
  "interface": "csmi",
  "temperature": 38,
  "health": "PASSED",
  "attributes": [
    {
      "id": 5,
      "name": "Reallocated_Sector_Ct",
      "value": 100,
      "worst": 100,
      "threshold": 10,
      "raw_value": 0
    }
  ]
}
```

### 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `uuid` | `string` | 是 | 本记录唯一标识，随机 `uuid4()`（**注意：每次采集都会变化**） |
| `host_uuid` | `string` | 是 | 所属主机 UUID（`UfsConfig.get_device_uuid()`） |
| `host_name` | `string` | 是 | 所属主机名（`UfsConfig.get_device_name()`） |
| `type` | `string` | 是 | 固定为 `"disk"` |
| `status` | `string` | 否 | 固定为 `"active"` |
| `last_access` | `number` | 是 | 毫秒时间戳 |
| `name` | `string` | 是 | 磁盘名，格式为 `"{host_name}/{device.name}"` |
| `model` | `string` | 否 | 磁盘型号 |
| `serial` | `string` | 是 | 磁盘序列号，**同时作为文件名** |
| `capacity` | `string` | 否 | 容量（含单位，如 `"931.51 GB"`） |
| `firmware` | `string` | 否 | 固件版本 |
| `interface` | `string` | 否 | 接口类型（如 `csmi` / `sata` / `nvme`） |
| `temperature` | `number` | 否 | 温度（摄氏度） |
| `health` | `string` | 否 | 健康评估（如 `PASSED` / `FAILED`），取自 `device.assessment` |
| `attributes` | `array` | 否 | SMART 属性数组，见下表 |

#### attributes[] 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `number` | SMART 属性 ID（`attr.num`） |
| `name` | `string` | 属性名（如 `Reallocated_Sector_Ct`） |
| `value` | `number` | 当前值（`attr.value`） |
| `worst` | `number` | 最差值（`attr.worst`） |
| `threshold` | `number` | 阈值（`attr.thresh`） |
| `raw_value` | `number` | 原始值（`attr.raw`） |

---

## 3. 主机 + 磁盘明细（含分区映射）

由 `save_disk_info_to_json.py` 采集，写入本地 `O:\app_data\disks\`：

- `disk_info_{device_name}_{YYYYMMDD_HHMMSS}.json`：带时间戳的快照
- `disk_info_{device_name}_latest.json`：当前主机最新数据
- `all_hosts_disk_info.json`：所有主机合并数据

### 3.1 单主机明细

```json
{
  "host_info": {
    "device_uuid": "550e8400-e29b-41d4-a716-446655440000",
    "device_name": "m10",
    "hostname": "DESKTOP-ABC",
    "platform": "Windows",
    "platform_version": "10.0.19045",
    "machine": "AMD64",
    "serial_number": "ABC123-XYZ"
  },
  "collection_time": "2024-07-02T12:09:03.123456",
  "last_updated": "2024-07-02T12:09:03.123456",
  "physical_disks": [
    {
      "DeviceID": "\\\\.\\PHYSICALDRIVE0",
      "Caption": "Samsung SSD 980 PRO 1TB",
      "Model": "Samsung SSD 980 PRO 1TB",
      "SerialNumber": "S5GXNF0R123456",
      "SizeGB": 931.51,
      "InterfaceType": "SCSI",
      "MediaType": "SSD",
      "Partitions": 4,
      "Status": "OK",
      "FirmwareRevision": "1B2QGXA7"
    }
  ],
  "logical_disks": [
    {
      "DeviceID": "C:",
      "VolumeName": "System",
      "FileSystem": "NTFS",
      "SizeGB": 930.5,
      "FreeSpaceGB": 420.3,
      "UsedSpaceGB": 510.2,
      "UsagePercent": 54.83,
      "DriveType": 3
    }
  ],
  "disk_partition_mappings": [
    {
      "PhysicalDisk": "\\\\.\\PHYSICALDRIVE0",
      "PhysicalDiskModel": "Samsung SSD 980 PRO 1TB",
      "PhysicalDiskSerial": "S5GXNF0R123456",
      "Partition": "Disk #0, Partition #1",
      "LogicalDisk": "C:",
      "VolumeName": "System"
    }
  ]
}
```

#### host_info 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `device_uuid` | `string` | 设备 UUID（`UfsConfig.get_device_uuid()`） |
| `device_name` | `string` | 设备名（`UfsConfig.get_device_name()`） |
| `hostname` | `string` | 系统主机名（`socket.gethostname()`） |
| `platform` | `string` | 平台（`platform.system()`，如 `Windows`） |
| `platform_version` | `string` | 平台版本（`platform.version()`） |
| `machine` | `string` | 机器架构（`platform.machine()`，如 `AMD64`） |
| `serial_number` | `string` | 主板序列号（`Win32_BIOS.SerialNumber`，已 strip） |

#### physical_disks[] 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `DeviceID` | `string` | 物理磁盘标识（如 `\\.\PHYSICALDRIVE0`） |
| `Caption` | `string` | 简短描述 |
| `Model` | `string` | 型号 |
| `SerialNumber` | `string` | 序列号（已 strip） |
| `SizeGB` | `number` | 容量（GB，四舍五入 2 位） |
| `InterfaceType` | `string` | 接口类型（`SCSI` / `IDE` / `USB` 等） |
| `MediaType` | `string` | 介质类型（`SSD` / `HDD` / `Fixed hard disk media`） |
| `Partitions` | `number` | 分区数 |
| `Status` | `string` | 状态（如 `OK`） |
| `FirmwareRevision` | `string` | 固件版本 |

#### logical_disks[] 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `DeviceID` | `string` | 逻辑盘符（如 `C:`） |
| `VolumeName` | `string` | 卷标 |
| `FileSystem` | `string` | 文件系统（`NTFS` / `FAT32` / `exFAT`） |
| `SizeGB` | `number` | 总大小（GB） |
| `FreeSpaceGB` | `number` | 空闲空间（GB） |
| `UsedSpaceGB` | `number` | 已用空间（GB） |
| `UsagePercent` | `number` | 使用率（%，四舍五入 2 位） |
| `DriveType` | `number` | 驱动器类型（`3` = 本地磁盘） |

#### disk_partition_mappings[] 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `PhysicalDisk` | `string` | 物理磁盘 `DeviceID` |
| `PhysicalDiskModel` | `string` | 物理磁盘型号 |
| `PhysicalDiskSerial` | `string` | 物理磁盘序列号 |
| `Partition` | `string` | 分区标识 |
| `LogicalDisk` | `string` | 对应逻辑盘符 |
| `VolumeName` | `string` | 卷标 |

> 仅 `DriveType == 3`（本地磁盘）的逻辑盘会被采集。

### 3.2 多主机合并文件

`all_hosts_disk_info.json` 结构：

```json
{
  "hosts": [
    {
      "host_info": { "device_uuid": "...", "device_name": "...", "hostname": "...", "platform": "...", "platform_version": "...", "machine": "...", "serial_number": "..." },
      "collection_time": "2024-07-02T12:09:03.123456",
      "last_updated": "2024-07-02T12:09:03.123456",
      "physical_disks": [ ... ],
      "logical_disks": [ ... ],
      "disk_partition_mappings": [ ... ]
    }
  ],
  "last_updated": "2024-07-02T12:09:03.123456"
}
```

合并规则：以 `host_info.device_uuid` 为主键，已存在则整体更新，否则追加；`last_updated` 写入整体最后更新时间。

---

## 4. 存储位置汇总

| 数据 | 存储后端 | 路径 / 命名 |
|------|----------|-------------|
| 主机状态心跳 | RemoteStorage | `ufs-nodes/{device_uuid}.json` |
| 磁盘 SMART | RemoteStorage | `ufs-nodes/{disk_serial}.json` |
| 主机+磁盘明细快照 | 本地磁盘 | `O:\app_data\disks\disk_info_{device_name}_{YYYYMMDD_HHMMSS}.json` |
| 主机最新明细 | 本地磁盘 | `O:\app_data\disks\disk_info_{device_name}_latest.json` |
| 多主机合并 | 本地磁盘 | `O:\app_data\disks\all_hosts_disk_info.json` |

---

## 5. 数据来源（源码）

- **主机状态心跳**：`home_cloud/remote_storage/client.py` → `update_my_status()`
- **磁盘 SMART**：`home_cloud/scripts/disk_info.py` → `get_disk_info()` / `save_to_json()`
- **主机+磁盘明细**：`home_cloud/scripts/save_disk_info_to_json.py` → `save_disk_info_to_json()` / `merge_host_data()`
- **设备标识**：`home_cloud/config_utils/ufs_configs.py` → `UfsConfig.get_device_uuid()` / `get_device_name()`
