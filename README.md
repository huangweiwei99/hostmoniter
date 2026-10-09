# hostmoniter

基于 **Docker Compose** 的 Dell 服务器硬件健康监控采集栈，通过 SNMP 轮询 **iDRAC**（Integrated Dell Remote Access Controller），采集温度、风扇转速、电源、RAID、磁盘、内存、CPU、网卡及事件日志等硬件健康指标，并写入时序数据库用于可视化与告警。

## 特性

- **两套采集链路并存**：
  - `categraf` → **VictoriaMetrics**（Prometheus 兼容，`/api/v1/write`）
  - `telegraf` → **InfluxDB2**（InfluxDB v2 原生写入协议）
- 直接消费 Dell 官方 **MIB**（DELL-RAC-MIB / iDRAC-SMIv2 等）解析 iDRAC OID
- 覆盖服务器硬件核心健康面：温度、风扇、电源、功耗、RAID/存储、物理磁盘、内存、CPU、PCI、网卡、机箱入侵与事件日志
- 采集器 / 时序库 / 数据目录全部由 `docker-compose.yaml` 一键编排

## 架构

```
                    ┌────────────── 采集层 ──────────────┐
Dell iDRAC (SNMP) ─▶│ categraf  ───────────────────────▶│──▶ VictoriaMetrics (host:8428)
  :161  (UDP)       │   (snmp_dell_idrac.toml)          │
                    │ telegraf  ───────────────────────▶│──▶ InfluxDB2 (host:8086)
                    │   (telegraf.d/idrac-input.conf)   │
                    └───────────────────────────────────┘
```

> 两套采集器独立运行、独立存储，可分别用于不同的可视化/告警后端，互不影响。

## 组件

| 服务                            | 镜像                                      | 作用                                              | 数据/配置目录                                    |
| ------------------------------- | ----------------------------------------- | ------------------------------------------------- | ------------------------------------------------ |
| `host_monitor-victoria-metrics` | `victoriametrics/victoria-metrics:latest` | 时序数据库，`host` 网络，保留 30 天               | `victoriametrics/vmdata/`                        |
| `host_monitor-categraf`         | `flashcatcloud/categraf:latest`           | 采集 Dell iDRAC SNMP 指标并推送 VM                | `categraf/conf/`, `categraf/mibs/`               |
| `host_monitor-influxdb2`        | `influxdb:latest`                         | InfluxDB2，自动初始化组织/用户/bucket，保留 30 天 | `influxdb2/data/`, `influxdb2/config/`           |
| `host_monitor-telegraf`         | `telegraf:latest`                         | 采集 Dell iDRAC SNMP 指标并推送 InfluxDB2         | `telegraf/telegraf.conf`, `telegraf/telegraf.d/` |

## 目录结构

```
hostmoniter/
├── docker-compose.yaml            # 一键编排全部服务
├── categraf/
│   ├── conf/
│   │   ├── config.toml            # categraf 全局配置（写入 VictoriaMetrics）
│   │   └── input.snmp/
│   │       └── snmp_dell_idrac.toml   # iDRAC SNMP 采集规则
│   └── mibs/
│       └── dell/                  # Dell 官方 MIB 文件（DELL-RAC-MIB 等）
├── telegraf/
│   ├── telegraf.conf              # telegraf 主配置（输出 InfluxDB2）
│   └── telegraf.d/
│       └── idrac-input.conf       # iDRAC SNMP 采集规则
├── influxdb2/
│   ├── data/                      # InfluxDB2 数据卷
│   └── config/                    # InfluxDB2 配置卷
└── victoriametrics/
    └── vmdata/                    # VictoriaMetrics 数据目录（运行时产物）
```

## 快速开始

```bash
# 1. 按需修改配置文件（见下方“配置”）
# 2. 启动全部服务
docker compose up -d

# 3. 查看状态
docker compose ps

# 4. 查看采集器日志
docker compose logs -f host_monitor-categraf
docker compose logs -f host_monitor-telegraf
```

- **VictoriaMetrics**：`http://<host>:8428`（Prometheus 兼容查询，可用 Grafana 接入）
- **InfluxDB2**：`http://<host>:8086`（可用其自带 UI / Flux 查询）

## 配置

### 1. 网络与后端地址

所有指标写入目标集中在以下位置，按实际环境修改：

| 文件                        | 项                             | 默认值                                 | 说明                                |
| --------------------------- | ------------------------------ | -------------------------------------- | ----------------------------------- |
| `categraf/conf/config.toml` | `[[writers]].url`              | `http://192.168.3.2:8428/api/v1/write` | categraf → VictoriaMetrics 写入地址 |
| `telegraf/telegraf.conf`    | `[[outputs.influxdb_v2]].urls` | `http://192.168.3.2:8086`              | telegraf → InfluxDB2 写入地址       |
| `docker-compose.yaml`       | 端口映射 /`network_mode`       | `8428`、`8086`、`8092/8094/8125`       | 对外暴露端口                        |

### 2. 被监控的 iDRAC 主机（SNMP agents）

**telegraf（`telegraf.d/idrac-input.conf`）**，SNMP v1，community `public`：

```
192.168.1.231  192.168.1.232  192.168.1.233  192.168.1.234
192.168.254.1  192.168.254.2  192.168.254.3  192.168.254.4
```

**categraf（`categraf/conf/input.snmp/snmp_dell_idrac.toml`）**，SNMP v2，community `public`，采集间隔 30s：

```
192.168.254.1  192.168.254.2  192.168.254.3  192.168.254.4
192.168.1.231  192.168.1.232  192.168.1.233
```

> 按需增删 `agents` 列表；生产环境请将 SNMP community 改为强口令，并限制 `161/udp` 访问来源。

### 3. MIB 文件

categraf 依赖 `/opt/categraf/mibs/dell` 目录下的 Dell MIB 做 OID → 名称翻译（`translator = "gosmi"`），仓库已内置常见 DELL-RAC / iDRAC MIB。

### Grafana 面板

iDRAC - Host Stats
ID:12106
InfluxDB 数据源:http://192.168.3.2:8086

Dell iDRAC SNMP Dashboard for VectoriaMetrics
ID:21107
Prometheus 数据源:http://192.168.3.2:8428

## 采集指标概览

| 类别      | 代表性指标（OID 字段名）                                                 |
| --------- | ------------------------------------------------------------------------ |
| 系统信息  | 型号、服务标签、OS 名称/版本、iDRAC URL、固件版本                        |
| 电源      | 电源状态、输入/输出电压、额定/当前功耗（Watts）、PSU 位置                |
| 风扇      | 各风扇转速（RPM）、风扇状态                                              |
| 温度      | 进风口/排风口温度、CPU 温度、温度告警阈值                                |
| RAID/存储 | 控制器状态/缓存、虚拟磁盘状态/容量/RAID 级别、物理磁盘状态/容量/介质类型 |
| 磁盘健康  | 磁盘 SMART 预测故障、SSD 剩余寿命、磁盘总线类型/接口速率                 |
| 内存      | 内存状态、类型、容量、频率、品牌、序列号                                 |
| CPU       | CPU 状态、当前/最大频率、核心/线程数、电压                               |
| PCI       | PCI 设备状态、品牌、FQDD                                                 |
| 网卡      | 网卡状态、厂商、当前/永久 MAC、FQDD                                      |
| 事件日志  | 日志条数、日志条目、严重级别、时间戳                                     |

## 端口一览

| 端口               | 服务                        | 协议      |
| ------------------ | --------------------------- | --------- |
| 8428               | VictoriaMetrics             | TCP       |
| 8086               | InfluxDB2                   | TCP       |
| 8092 / 8094 / 8125 | telegraf（statsd / inputs） | UDP / TCP |
| 161                | 外部 iDRAC SNMP（出站）     | UDP       |

## License

MIT
