# Windflex 第一期开发计划（原型系统）

## 概述

| 项目 | 内容 |
|------|------|
| **目标** | 可部署、可运行的原型自动化测试系统 |
| **电源类型** | 不可编程电源（Non-programmable，持续供电）|
| **排除范围** | OpenCLaw (AI Gateway) 暂不实现 |
| **计划周期** | 4 周 |
| **负责仓库** | windflex-system, windflex-deploy, board-test-cases |

---

## 1. 架构决策

### 1.1 技术选型

| 组件 | 技术 | 说明 |
|------|------|------|
| 硬件控制服务 | Python 3.11 + FastAPI | 轻量、异步、自带 Swagger UI |
| 进程管理 | Uvicorn (ASGI) | FastAPI 推荐 |
| SSH 控制 | paramiko | 成熟的 Python SSH 库 |
| 串口通信 | pyserial | 标准串口库 |
| ADB 控制 | adb-shell | 纯 Python ADB 实现 |
| 容器化 | Docker + Docker Compose | 一键部署 |
| CI/CD | Jenkins LTS (Docker) | 标准 Jenkins，含 Robot Framework 插件 |
| 测试框架 | Robot Framework 6.x | 关键字驱动，支持 SSHLibrary |
| 数据库 | PostgreSQL 14 | 存储测试结果 |
| 缓存/队列 | Redis 7 | 任务队列、状态缓存 |

### 1.2 不可编程电源模式说明

由于电源为不可编程电源（Non-programmable），原型系统做如下处理：

- `POWER_MODEL=Non-programmable` 环境变量标识当前模式
- 硬件服务启动时自动检测并记录电源能力
- **跳过**所有标记 `requires_power_control` 的测试用例
- **仅支持** SSH 软重启（`shutdown -r now`）
- 电源 API 端点存在但返回 `{"mode": "non-programmable", "available": false}`

### 1.3 单台设备假设

- 一台 8397 测试板，固定 IP
- 一台 ADB 设备（唯一，无需指定 device ID）
- 串口设备（`/dev/ttyUSB0`，用于日志采集，非电源控制）

---

## 2. 仓库结构规划

### 2.1 windflex-system

```
windflex-system/
├── README.md
├── docker-compose.yml
├── .env.example
├── .gitignore
├── services/
│   └── hardware-service/
│       ├── Dockerfile
│       ├── requirements.txt
│       ├── main.py                    # FastAPI 入口
│       ├── config.py                  # 配置管理（从环境变量加载）
│       ├── api/
│       │   ├── __init__.py
│       │   ├── power.py               # 电源 API（非编程模式）
│       │   ├── adb.py                 # ADB API
│       │   ├── serial_api.py          # 串口 API
│       │   └── ssh_api.py             # SSH API
│       ├── drivers/
│       │   ├── __init__.py
│       │   ├── power/
│       │   │   ├── __init__.py
│       │   │   ├── base.py            # 抽象基类
│       │   │   └── non_programmable.py # 非编程电源驱动
│       │   ├── adb_manager.py         # ADB 管理器
│       │   ├── serial_communicator.py # 串口通信器
│       │   └── ssh_client.py          # SSH 客户端
│       └── tests/
│           └── test_api.py            # 单元测试
├── jenkins/
│   ├── Dockerfile                     # 定制 Jenkins（含 RF 插件）
│   ├── plugins.txt                    # Jenkins 插件列表
│   ├── casc/
│   │   └── jenkins.yaml               # JCasC 配置
│   └── Jenkinsfile                    # 流水线定义
└── scripts/
    ├── init-db.sh
    └── wait-for-it.sh
```

### 2.2 windflex-deploy

```
windflex-deploy/
├── README.md
├── scripts/
│   ├── check-prerequisites.sh         # 环境检测
│   ├── setup-usb-permissions.sh       # USB 权限配置
│   └── deploy.sh                      # 一键部署
├── config/
│   └── udev-rules/
│       └── 99-windflex.rules
└── tests/
    └── verify-deployment.sh
```

### 2.3 board-test-cases

```
board-test-cases/
├── README.md
├── .gitignore
├── resources/
│   ├── variables.resource             # 全局变量
│   └── keywords.resource             # 自定义关键字
├── test-suites/
│   ├── functional/
│   │   ├── ssh_connectivity.robot    # SSH 连通性测试
│   │   └── network_test.robot        # 网络测试
│   └── process-monitor/
│       ├── powermanager_monitor.robot # powermanager 进程监控
│       └── system_services.robot      # 系统服务检查
└── config/
    └── test-config.yaml
```

---

## 3. 开发里程碑

### Week 1：硬件控制服务

| 任务 | 产出 | 验收标准 |
|------|------|---------|
| FastAPI 骨架搭建 | `main.py`, `config.py` | 服务启动，Swagger UI 可访问 |
| SSH 客户端 | `drivers/ssh_client.py` | 成功连接板子并执行命令 |
| ADB 管理器 | `drivers/adb_manager.py` | 检测设备、推送 SSH key |
| 串口通信器 | `drivers/serial_communicator.py` | 打开串口、发送/接收数据 |
| 非编程电源驱动 | `drivers/power/non_programmable.py` | 返回能力描述，支持 SSH 重启 |
| API 路由实现 | `api/` 下所有文件 | 所有端点返回正确 JSON |
| Dockerfile | `Dockerfile`, `requirements.txt` | 镜像可构建 |

### Week 2：Jenkins + Docker Compose

| 任务 | 产出 | 验收标准 |
|------|------|---------|
| Jenkins Dockerfile | 定制镜像 | 含 Robot Framework Plugin |
| JCasC 配置 | `jenkins.yaml` | Jenkins 开箱即用 |
| Jenkinsfile | 流水线 | 可触发 Robot Framework 测试 |
| Docker Compose | `docker-compose.yml` | 一条命令启动全部服务 |
| 环境变量模板 | `.env.example` | 文档完整 |

### Week 3：Robot Framework 测试用例

| 任务 | 产出 | 验收标准 |
|------|------|---------|
| 公共资源库 | `resources/` | 关键字可复用 |
| SSH 连通性测试 | `ssh_connectivity.robot` | 测试通过 |
| powermanager 监控 | `powermanager_monitor.robot` | 监控 10 秒，PID 稳定则通过 |
| 系统服务检查 | `system_services.robot` | 检查关键进程 |
| 网络测试 | `network_test.robot` | ping + HTTP 测试 |

### Week 4：部署脚本 + 集成测试

| 任务 | 产出 | 验收标准 |
|------|------|---------|
| 一键部署脚本 | `deploy.sh` | 全新机器跑通 |
| 环境检测脚本 | `check-prerequisites.sh` | 缺失依赖给出提示 |
| USB 权限配置 | udev rules | `/dev/ttyUSB0` 有访问权限 |
| 端到端集成测试 | — | Jenkins 触发 RF 测试并输出报告 |

---

## 4. 接口规范（Hardware Service API）

### Base URL
```
http://localhost:8000
```

### 电源
```
GET  /api/v1/power/capability     # 查询电源能力
GET  /api/v1/power/status         # 当前电源状态
POST /api/v1/power/reboot         # SSH 软重启（仅非编程模式支持）
```

### ADB
```
GET  /api/v1/adb/devices          # 列出已连接 ADB 设备
POST /api/v1/adb/push-key         # 推送 SSH 公钥到板子
POST /api/v1/adb/command          # 执行 ADB 命令
```

### 串口
```
GET  /api/v1/serial/status        # 串口连接状态
POST /api/v1/serial/send          # 发送串口命令
POST /api/v1/serial/open          # 打开串口
POST /api/v1/serial/close         # 关闭串口
```

### SSH
```
GET  /api/v1/ssh/status           # SSH 连接状态
POST /api/v1/ssh/exec             # 执行远程命令
DELETE /api/v1/ssh/disconnect     # 断开连接
```

### 系统
```
GET  /api/v1/health               # 健康检查
GET  /api/v1/info                 # 系统信息（电源模式、板子 IP 等）
```

---

## 5. 关键设计决策

| 决策 | 选项 | 选择 | 理由 |
|------|------|------|------|
| 电源能力检测 | 运行时探测 / 环境变量 | 环境变量 `POWER_MODEL` | 可靠，避免探测失败 |
| SSH 连接池 | 长连接复用 / 按需连接 | 按需连接 + 超时自动重连 | 原型阶段简单可靠 |
| ADB 工具 | adb CLI / adb-shell 库 | 两者兼容（优先 CLI，fallback 库）| 兼容性最好 |
| 数据持久化 | PostgreSQL / SQLite | PostgreSQL（与生产环境一致）| 避免后期迁移 |
| Jenkins 配置 | 手动配置 / JCasC | JCasC | 可重复部署，容器无状态 |
| 测试报告 | Jenkins 内置 / RF HTML | RF HTML + JUnit XML | 双格式，兼容性好 |

---

## 6. 风险与措施

| 风险 | 影响 | 措施 |
|------|------|------|
| 无真实板子测试 | 集成测试无法验证 | Mock SSH 响应，本机可跑单元测试 |
| Docker 内访问 USB/串口 | 容器权限不足 | `privileged: true` + udev rules |
| ADB 驱动兼容性 | ADB 命令失败 | 先检测 adb CLI 可用性，不可用时返回 503 |
| Jenkins 插件版本冲突 | 镜像构建失败 | 锁定插件版本，测试通过后不升级 |

---

## 7. 验收标准

原型系统完成标准：

1. `docker-compose up -d` 启动所有服务（hardware-service, jenkins, postgres, redis）
2. `http://localhost:8000/docs` Swagger UI 可访问，所有 API 端点有文档
3. `http://localhost:8080` Jenkins 可登录
4. Jenkins 流水线可触发，Robot Framework 测试用例跑通（含跳过 `requires_power_control` 的用例）
5. 测试报告可在 Jenkins 界面查看
6. `./scripts/deploy.sh` 在全新 Ubuntu 服务器上可成功部署
