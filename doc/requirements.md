# 自动化测试系统需求文档

## 1. 项目概述

### 1.1 项目目标
开发一套可 Docker 化部署的自动化测试系统，用于测试 8397 Linux 开发板，实现测试流程自动化、结果分析和测试用例管理。

### 1.2 项目结构（三仓库架构）
```
┌─────────────────────────────────────────────────────────────┐
│  1. windflex-deploy（部署脚本仓库）                          │
│     - 服务器环境检测和配置                                   │
│     - Docker 和依赖安装                                      │
│     - 电源驱动安装（如需要）                                 │
│     - 一键部署脚本                                           │
│     Git Repo: github.com/yourorg/windflex-deploy            │
└─────────────────────────────────────────────────────────────┘
                            ↓ 部署
┌─────────────────────────────────────────────────────────────┐
│  2. windflex-system（系统代码和配置仓库）                    │
│     - Docker Compose 配置                                    │
│     - 环境变量模板（.env.example）                           │
│     - Hardware Service 代码                                   │
│     - OpenCLaw (AI Gateway) 代码                              │
│     - Jenkins 配置（Jenkinsfile）                            │
│     Git Repo: github.com/yourorg/windflex-system            │
└─────────────────────────────────────────────────────────────┘
                            ↓ 加载测试用例
┌─────────────────────────────────────────────────────────────┐
│  3. test-cases（测试用例仓库 - 独立 Git Repo）               │
│     - Robot Framework 测试用例 (.robot 文件)                  │
│     - 测试数据和配置文件                                     │
│     - 测试报告和日志                                         │
│     Git Repo: github.com/yourorg/board-test-cases           │
│                                                              │
│     贡献者：                                                 │
│     - 用户提交 MR/PR                                         │
│     - LLM 自动生成并提交 MR/PR                               │
│     - Jenkins 自动拉取并执行                                 │
└─────────────────────────────────────────────────────────────┘
```

### 1.3 硬件环境
```
┌─────────────────┐
│  Ubuntu Server  │
│                 │
│  - Jenkins      │────── SSH ──────┐
│  - OpenCLaw     │                  │
│  - Docker       │                  ▼
│                 │         ┌─────────────────┐
│  - USB/Serial ──┼─────────│  8397 Board     │
│  - ADB          │         │  (固定 IP)       │
│                 │         │                 │
└─────────────────┘         └─────────────────┘
         │
         │ Serial/USB
         ▼
┌──────────────────┐
│ Programmable PSU │
│ (可编程电源)      │
│                  │
│  - 电压控制       │
│  - 电流监控       │
└──────────────────┘
```

### 1.3 硬件环境
```
┌─────────────────┐
│  Ubuntu Server  │
│                 │
│  - Jenkins      │────── SSH ──────┐
│  - OpenCLaw     │                  │
│  - Docker       │                  ▼
│                 │         ┌─────────────────┐
│  - USB/Serial ──┼─────────│  8397 Board     │
│  - ADB          │         │  (固定 IP)       │
│                 │         │                 │
└─────────────────┘         └─────────────────┘
         │
         │ Serial/USB
         ▼
┌──────────────────┐
│ Programmable PSU │
│ (可编程电源)      │
│                  │
│  - 电压控制       │
│  - 电流监控       │
└──────────────────┘
```

### 1.4 通信方式
- **Server → 电源**: Serial/USB 通信
- **Server → 板子**: 
  - SSH（主要方式，通过密钥认证）
  - 串口（备用方式）
  - ADB（用于推送 SSH Key）
- **板子 IP**: 固定 IP 地址

## 2. 系统组件

### 2.1 架构图
```
┌─────────────────────────────────────────────────────────────┐
│                    Docker Container                          │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                  Jenkins Server                        │  │
│  │  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐ │  │
│  │  │ Robot       │  │ Test         │  │ Report       │ │  │
│  │  │ Framework   │  │ Scheduler    │  │ Generator    │ │  │
│  │  └─────────────┘  └──────────────┘  └──────────────┘ │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              OpenCLaw (AI Gateway)                     │  │
│  │  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐ │  │
│  │  │ AI Test     │  │ Failure      │  │ MR           │ │  │
│  │  │ Generator   │  │ Analyzer     │  │ Generator    │ │  │
│  │  └─────────────┘  └──────────────┘  └──────────────┘ │  │
│  │  ┌─────────────┐  ┌──────────────┐                   │  │
│  │  │ LLM         │  │ Log          │                   │  │
│  │  │ Integration │  │ Parser       │                   │  │
│  │  └─────────────┘  └──────────────┘                   │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              Hardware Control Service                  │  │
│  │  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐ │  │
│  │  │ Power       │  │ ADB          │  │ SSH          │ │  │
│  │  │ Controller  │  │ Manager      │  │ Client       │ │  │
│  │  └─────────────┘  └──────────────┘  └──────────────┘ │  │
│  │  ┌─────────────┐                                      │  │
│  │  │ Serial      │                                      │  │
│  │  │ Communicator│                                      │  │
│  │  └─────────────┘                                      │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## 3. 详细功能需求

### 3.1 硬件控制服务 (Hardware Control Service)

#### 3.1.1 电源管理
- **电源类型支持**
  - 可编程电源：DLX-30V10AMP（默认）
  - 不可编程电源：常供电模式
  - 系统自动检测电源类型

- **上电/下电控制**（仅可编程电源）
  - 支持远程开关板子电源
  - 支持定时开关机
  - 支持电源序列控制
  
- **电压/电流监控**（仅可编程电源）
  - 实时读取输出电压（精度要求：±0.1V）
  - 实时读取输出电流（精度要求：±0.01A）
  - 支持电压/电流阈值告警
  - 数据记录和历史查询

- **电源保护**（仅可编程电源）
  - 过压保护
  - 过流保护
  - 短路保护
  - 温度监控（如支持）

- **常供电模式**（不可编程电源）
  - 板子持续供电，无法软件控制开关机
  - 仅支持软件重启（通过 SSH 命令）
  - 跳过需要电源控制的测试用例

#### 3.1.2 ADB 管理
- **SSH Key 推送**
  - 通过 ADB 推送 SSH 公钥到板子（单一设备，无需指定设备 ID）
  - 自动配置 authorized_keys
  - 支持密钥轮换
  
- **ADB 连接管理**
  - 检测唯一设备连接状态
  - 自动重连机制
  - ADB 命令执行

#### 3.1.3 串口通信
- 支持 USB 转串口设备
- 波特率可配置（9600, 115200 等）
- 支持 AT 命令发送和响应解析
- 串口日志记录

#### 3.1.4 SSH 连接管理
- 自动 SSH 连接到测试板（固定 IP）
- SSH 密钥认证（无需密码）
- 连接池管理
- 命令执行超时控制
- 执行结果捕获和返回

#### 3.1.5 电源状态记录
- **测试日志集成**
  - 电源状态直接写入测试日志（Robot Framework log）
  - 使用 `Log Power Status` 关键字在关键节点记录
  - 日志格式：`[timestamp] Power: V=12.1V, I=0.5A, P=6.05W`
  
- **记录时机**
  - 测试开始/结束时
  - 电源操作（开机/关机/重启）前后
  - 关键测试步骤执行时
  
- **异常标记**
  - 电压/电流超出阈值时在日志中标记警告
  - 电源异常与测试失败关联

#### 3.1.6 电源能力检测
- **检测机制**
  - 通过环境变量 `POWER_MODEL` 判断电源类型
  - `POWER_MODEL=DLX-30V10AMP`：可编程电源，执行电源控制测试
  - `POWER_MODEL=Non-programmable`：不可编程电源，跳过电源控制测试
  - 检测结果在系统启动时确定，支持运行时刷新

- **能力标识**
  - `Power Control Available`: True/False（根据 POWER_MODEL 判断）
  - `Power Model`: DLX-30V10AMP / Non-programmable
  - 在仪表板显示当前电源状态

- **测试用例过滤**
  - 根据 `POWER_MODEL` 环境变量自动过滤可执行的测试用例
  - 可编程电源（DLX-30V10AMP）：运行所有测试用例
  - 不可编程电源：跳过标记为 `requires_power_control` 的测试用例
  - 在测试报告中说明跳过的用例及原因

### 3.2 Jenkins CI/CD 系统

#### 3.2.1 Robot Framework 集成
- **插件要求**
  - Robot Framework Jenkins Plugin
  - GitLab Plugin（拉取测试用例）
  - 支持 .robot 格式测试用例
  - 支持测试变量和参数化
  
- **测试库支持**
  - SSHLibrary（SSH 连接）
  - RequestsLibrary（HTTP API 测试）
  - 自定义电源控制库
  - 自定义串口通信库
  - 自定义进程监控库
  
- **Git 平台集成**
  - 同时支持 GitHub 和 GitLab
  - Jenkins 配置对应平台凭证
  - 支持从 GitHub/GitLab 拉取测试用例仓库
  - Webhook 触发自动测试
  - 可配置默认 Git 平台

#### 3.2.2 测试用例分类
根据测试需求分为两类：

**1. 需要电源控制的测试**（Tags: `requires_power_control`）
- 测试板开机流程
- 测试板关机流程
- 电源状态切换测试
- 功耗测试
- 仅当 `POWER_MODEL=DLX-30V10AMP` 时运行

**2. 无需电源控制的测试**（Tags: `no_power_control`）
- 软件功能测试
- 进程稳定性监控
- SSH/网络测试
- 系统服务测试
- 所有电源模式下都可运行

#### 3.2.2 测试用例类型
- **测试用例电源需求属性**
  - `Requires Power Control`: True/False（是否需要可编程电源）
  - `Power State`: ON/OFF/BOTH（需要的电源状态）
  - 系统根据 `POWER_MODEL` 环境变量自动过滤可执行的测试用例

```robot
*** Test Cases ***
测试板开机流程
    [Documentation]  测试板子正常开机流程
    [Tags]    requires_power_control    power:ON
    Log Power Status    start=power_on
    Power On Board
    Log Power Status    event=power_on_complete
    Wait Until Keyword Succeeds    5 min    10 s    Check Board SSH Accessible
    Log Power Status    event=ssh_accessible
    Check Voltage Normal
    Check Current Normal
    Check System Process Running
    Log Power Status    end=power_on_success

测试板关机流程
    [Documentation]  测试板子正常关机流程
    [Tags]    requires_power_control    power:OFF
    Log Power Status    start=power_off
    Execute SSH Command    sync
    Execute SSH Command    shutdown -h now
    Wait Until Keyword Succeeds    2 min    5 s    Check Board Power Off
    Log Power Status    event=power_off_complete
    Check Voltage Zero
    Check Current Near Zero
    Log Power Status    end=power_off_success

测试业务逻辑
    [Documentation]  执行用户自定义测试
    [Tags]    no_power_control    power:ON
    Log Power Status    event=test_start
    ${result}=    Execute SSH Command    /path/to/test_script.sh
    Log Power Status    event=test_end
    Should Be Equal As Strings    ${result}    SUCCESS

测试软件功能（无需电源控制）
    [Documentation]  仅测试软件功能，不需要电源控制
    [Tags]    no_power_control    power:ANY
    ${result}=    Execute SSH Command    /path/to/software_test.sh
    Should Be Equal As Strings    ${result}    SUCCESS

测试 powermanager 进程稳定性
    [Documentation]  监控 powermanager 进程是否稳定运行（无需电源控制）
    [Tags]    no_power_control    power:ON    process_monitor
    [Setup]    Log    Starting powermanager process monitor
    
    # 第一次检查进程
    ${pid1}=    Execute SSH Command    pidof powermanager || echo "not_running"
    Log    First check - powermanager PID: ${pid1}
    
    # 如果进程不在运行，测试失败
    Should Not Be Equal    ${pid1}    not_running
    ...    msg=powermanager process is not running
    
    # 等待 10 秒
    Sleep    10s    Wait for 10 seconds to check process stability
    
    # 第二次检查进程
    ${pid2}=    Execute SSH Command    pidof powermanager || echo "not_running"
    Log    Second check - powermanager PID: ${pid2}
    
    # 如果进程不在运行，测试失败
    Should Not Be Equal    ${pid2}    not_running
    ...    msg=powermanager process crashed during monitoring
    
    # 检查 PID 是否一致
    Should Be Equal As Strings    ${pid1}    ${pid2}
    ...    msg=powermanager process restarted (PID changed from ${pid1} to ${pid2})
    ...    values=False
    
    Log    powermanager process is stable with PID: ${pid1}
```

**进程监控测试用例说明**：
- **目的**: 监控板子上的 `powermanager` 进程是否稳定运行
- **标签**: `no_power_control`, `process_monitor`
- **执行逻辑**:
  1. 第一次检查：获取 `powermanager` 进程的 PID
  2. 如果进程不在运行，立即失败
  3. 等待 10 秒
  4. 第二次检查：再次获取 `powermanager` 进程的 PID
  5. 如果进程不在运行，失败（进程崩溃）
  6. 比较两次 PID 是否一致
  7. 如果 PID 不一致，失败（进程重启）
- **成功条件**: 进程持续运行且 PID 保持一致
- **失败条件**:
  - 进程未运行
  - 进程在监控期间崩溃
  - 进程在监控期间重启（PID 变化）
- **适用场景**: 
  - 常供电模式下的稳定性测试
  - 系统服务健康检查
  - 长期运行测试的一部分

#### 3.2.3 测试调度
- 支持定时任务
- 支持 Git Webhook 触发
- 支持手动触发
- 测试队列管理
- 优先级调度

#### 3.2.4 报告生成
- HTML 测试报告
- JUnit XML 格式输出
- 趋势分析图表
- 邮件通知

### 3.3 OpenCLaw (AI Gateway)

#### 3.3.1 AI 测试用例生成
- **LLM 集成**
  - 默认支持 GitHub Copilot API
  - 可切换至 Claude、OpenAI GPT-4 或其他 LLM
  - 根据需求文档自动生成 Robot Framework 测试用例
  - 支持测试用例优化建议
  
- **用例模板**
  - 提供标准测试用例模板
  - 支持自定义 Prompt 工程
  - 批量生成测试用例
  
- **LLM 配置管理**
  - Web 界面配置 LLM Provider
  - API Key 安全存储
  - 支持多个 LLM 配置快速切换

#### 3.3.2 失败分析
- **日志收集**
  - 自动收集测试日志
  - 系统日志（dmesg, journalctl）
  - 应用日志
  - 电源日志
  
- **AI 驱动分析**
  - LLM 分析失败日志
  - 根因定位和解释
  - 提供修复建议
  - 历史失败对比
  
- **智能分类**
  - 失败类型自动分类（硬件/软件/网络）
  - 相似问题推荐
  - 已知问题匹配

#### 3.3.3 Git 平台 MR/PR 自动化
- **MR/PR 自动生成**
  - 根据 AI 生成的测试用例创建代码
  - 自动提交到 GitHub 或 GitLab 仓库
  - GitHub: 创建 Pull Request
  - GitLab: 创建 Merge Request
  - 填写 PR/MR 描述（包含 AI 分析结果）
  
- **PR/MR 管理**
  - PR/MR 状态跟踪
  - CI/CD 流水线触发
  - Code Review 辅助
  - 自动回复 Review 评论

#### 3.3.4 仪表板
- 实时测试状态
- 通过率统计
- AI 分析结果展示
- 硬件状态监控

## 4. 技术栈

### 4.1 后端技术
- **语言**: Python 3.9+
- **Web 框架**: FastAPI / Flask
- **硬件通信**: 
  - pyserial（串口）
  - paramiko（SSH）
  - pyvisa（电源控制，如支持）
  - adb-shell / pure-python-adb（ADB）
- **AI 集成**: 
  - LangChain / LlamaIndex
  - GitHub Copilot API（默认）
  - Anthropic Claude API / OpenAI API（可切换）
- **Git 平台集成**:
  - PyGithub（GitHub API）
  - python-gitlab（GitLab API）
- **任务队列**: Celery + Redis

### 4.2 前端技术
- **框架**: Vue.js 3 / React
- **UI 库**: Element Plus / Ant Design
- **图表**: ECharts

### 4.3 数据库
- **主数据库**: PostgreSQL（配置、测试结果、GitLab 凭证等）
- **缓存**: Redis

### 4.4 DevOps
- **容器化**: Docker
- **编排**: Docker Compose / Kubernetes
- **CI/CD**: Jenkins
- **监控**: Prometheus + Grafana

## 5. Docker 部署方案

### 5.1 容器结构
```yaml
version: '3.8'

services:
  jenkins:
    image: jenkins/jenkins:lts
    ports:
      - "${JENKINS_HTTP_PORT:-8080}:8080"
      - "${JENKINS_JNLP_PORT:-50000}:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/usb:/var/run/usb  # USB 设备
    devices:
      - ${SERIAL_DEVICE:-/dev/ttyUSB0}:/dev/ttyUSB0  # 串口设备映射（电源）
      - /dev/bus/usb:/dev/bus/usb  # ADB 设备
    privileged: true
    environment:
      - TZ=${TZ:-Asia/Shanghai}
      - JAVA_OPTS=-Duser.timezone=${TZ:-Asia/Shanghai}
    restart: unless-stopped
  
  hardware-service:
    build: ./hardware-service
    ports:
      - "${HARDWARE_SERVICE_PORT:-8000}:8000"
    volumes:
      - /var/run/usb:/var/run/usb
    devices:
      - ${SERIAL_DEVICE:-/dev/ttyUSB0}:/dev/ttyUSB0
      - /dev/bus/usb:/dev/bus/usb
    privileged: true
    environment:
      - TZ=${TZ:-Asia/Shanghai}
      - BOARD_IP=${BOARD_IP}
      - BOARD_SSH_PORT=${BOARD_SSH_PORT:-22}
      - POWER_MODEL=${POWER_MODEL:-DLX-30V10AMP}
      - SERIAL_DEVICE=${SERIAL_DEVICE:-/dev/ttyUSB0}
      - BAUD_RATE=${BAUD_RATE:-9600}
      - LOG_LEVEL=${LOG_LEVEL:-INFO}
    restart: unless-stopped
  
  openclaw:
    build: ./openclaw
    ports:
      - "${OPENCLAW_PORT:-3000}:3000"
    depends_on:
      - postgres
      - redis
    environment:
      - TZ=${TZ:-Asia/Shanghai}
      - LLM_PROVIDER=${LLM_PROVIDER:-copilot}
      - LLM_API_KEY=${LLM_API_KEY}
      - GIT_PLATFORM=${GIT_PLATFORM:-gitlab}
      - GIT_URL=${GIT_URL}
      - GIT_USER=${GIT_USER}
      - GIT_TOKEN=${GIT_TOKEN}
      - LOG_LEVEL=${LOG_LEVEL:-INFO}
    restart: unless-stopped
  
  postgres:
    image: postgres:14
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      - TZ=${TZ:-Asia/Shanghai}
      - POSTGRES_PASSWORD=windflex_db_password
    restart: unless-stopped
  
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    restart: unless-stopped

volumes:
  jenkins_home:
  pgdata:
  redis_data:
```

### 5.2 设备映射
- 串口设备：`/dev/ttyUSB0` → 容器内 `/dev/ttyUSB0`（电源控制）
- USB 设备：`/dev/bus/usb` → 容器内（ADB 访问）
- 网络：host 模式或端口映射
- 注意：需要配置 udev 规则确保容器内设备权限

## 6. 接口设计

### 6.1 硬件控制 API
```python
# 电源控制
POST /api/power/on    # 上电
POST /api/power/off   # 下电
GET  /api/power/status  # 获取电源状态

# 电压电流读取
GET /api/power/voltage  # 读取电压
GET /api/power/current  # 读取电流

# ADB 管理
POST /api/adb/push-key   # 推送 SSH Key
GET  /api/adb/devices    # 列出设备
POST /api/adb/command    # 执行 ADB 命令

# 串口通信
POST /api/serial/send   # 发送串口命令
GET  /api/serial/receive  # 读取串口响应

# SSH 连接
POST /api/ssh/exec      # 执行 SSH 命令
GET  /api/ssh/status    # SSH 状态检查
```

### 6.2 测试管理 API
```python
# 测试执行
POST /api/tests/run     # 执行测试
POST /api/tests/schedule  # 调度测试
GET  /api/tests/status/{id}  # 获取测试状态

# 结果查询
GET /api/tests/results  # 获取测试结果
GET /api/tests/report/{id}  # 获取测试报告

# 失败分析
POST /api/analysis/analyze  # 分析失败
GET  /api/analysis/history  # 历史分析
```

## 7. 安全要求

### 7.1 访问控制
- Jenkins 用户认证
- API Token 认证
- SSH 密钥管理
- 角色权限管理

### 7.2 数据安全
- 敏感配置加密存储
- 测试数据备份
- 日志审计

## 8. 开发计划

### Phase 1: 基础设施（2 周）
- [ ] Docker 环境搭建
- [ ] 硬件控制服务开发
- [ ] 电源控制接口实现
- [ ] SSH 连接管理

### Phase 2: Jenkins 集成（2 周）
- [ ] Jenkins Docker 部署
- [ ] Robot Framework 集成
- [ ] 自定义测试库开发
- [ ] 测试用例示例编写

### Phase 3: OpenCLaw 服务（3 周）
- [ ] LLM 集成（AI Gateway）
- [ ] 失败分析引擎（AI 驱动）
- [ ] GitLab MR 自动化
- [ ] Web 界面开发

### Phase 4: 测试和优化（1 周）
- [ ] 系统集成测试
- [ ] 性能优化
- [ ] 文档完善

## 9. 风险和挑战

### 9.1 技术风险
- DLX-30V10AMP 电源通信协议兼容性
- Docker 容器内访问硬件设备的权限问题
- 串口通信的稳定性
- 不可编程电源模式下测试覆盖率降低

### 9.2 缓解措施
- 提前测试 DLX-30V10AMP 通信协议
- 使用 privileged 模式或正确配置 udev 规则
- 添加重试机制和超时处理
- 设计测试用例时明确标注电源需求属性

## 10. 配置参数

### 10.1 环境变量配置（Ubuntu Server）
所有敏感配置通过环境变量在 Ubuntu Server 上配置，容器启动时自动加载。

**环境变量文件位置**: `/etc/windflex/.env` 或项目根目录 `.env`

```bash
# ==================== 板子配置 ====================
BOARD_IP=192.168.1.100          # 板子固定 IP 地址
BOARD_SSH_PORT=22               # SSH 端口

# ==================== 电源配置 ====================
POWER_MODEL=DLX-30V10AMP        # DLX-30V10AMP 或 Non-programmable
SERIAL_DEVICE=/dev/ttyUSB0      # 串口设备路径
BAUD_RATE=9600                  # 波特率：9600 或 115200

# ==================== Git 平台配置 ====================
GIT_PLATFORM=gitlab             # github 或 gitlab
GIT_URL=https://gitlab.example.com/username/repo.git
GIT_USER=username               # Git 用户名
GIT_TOKEN=your_git_token        # Git Personal Access Token

# Jenkins Git 凭证（可选，默认使用 GIT_TOKEN）
JENKINS_GIT_USER=${GIT_USER}
JENKINS_GIT_TOKEN=${GIT_TOKEN}

# ==================== LLM 配置 ====================
LLM_PROVIDER=copilot            # copilot/claude/openai
LLM_API_KEY=your_llm_api_key    # LLM API Key

# ==================== 其他配置 ====================
TZ=Asia/Shanghai                # 时区
LOG_LEVEL=INFO                  # 日志级别：DEBUG/INFO/WARNING/ERROR
```

### 10.2 板子配置
- **固定 IP 地址**: 通过 `BOARD_IP` 环境变量配置
- **SSH 端口**: 通过 `BOARD_SSH_PORT` 环境变量配置
- **ADB**: 单一设备，无需设备 ID

### 10.3 电源配置
- **电源型号**: 通过 `POWER_MODEL` 环境变量配置
- **电源控制**: 自动检测（系统启动时检测）
- **串口设备**: 通过 `SERIAL_DEVICE` 环境变量配置
- **波特率**: 通过 `BAUD_RATE` 环境变量配置

### 10.4 Git 平台配置
- **Git 平台选择**: 通过 `GIT_PLATFORM` 环境变量配置（github/gitlab）
- **仓库 URL**: 通过 `GIT_URL` 环境变量配置
- **用户名**: 通过 `GIT_USER` 环境变量配置
- **Access Token**: 通过 `GIT_TOKEN` 环境变量配置
  - GitHub: Personal Access Token (PAT)
  - GitLab: Personal Access Token
  - 权限要求：repo, read:org, workflow（GitHub）；api, read_repository, write_repository（GitLab）

### 10.5 LLM 配置
- **LLM Provider**: 通过 `LLM_PROVIDER` 环境变量配置
- **API Key**: 通过 `LLM_API_KEY` 环境变量配置
- **切换方式**: 修改 `LLM_PROVIDER` 环境变量或 Web 界面配置

### 10.6 环境变量加载方式
**方式 1：系统级配置**（推荐）
```bash
# 创建配置文件
sudo mkdir -p /etc/windflex
sudo vim /etc/windflex/.env

# 在 docker-compose.yml 中引用
env_file:
  - /etc/windflex/.env
```

**方式 2：项目级配置**
```bash
# 在项目根目录创建 .env 文件
cp .env.example .env
vim .env

# docker-compose 自动加载当前目录的 .env 文件
```

**方式 3：命令行导出**
```bash
export BOARD_IP=192.168.1.100
export GIT_TOKEN=your_token
# ... 导出其他变量

docker-compose up -d
```

## 11. 待确认事项

1. **板子的固定 IP 地址？**
2. **ADB 设备是否需要特殊驱动？**
3. **主要使用的 Git 平台？**（GitHub/GitLab 或两者都用）
4. **Git 仓库的 URL？**

## 12. 仓库详细说明

### 12.1 windflex-deploy（部署脚本仓库）
**位置**: `github.com/yourorg/windflex-deploy`

**目录结构**:
```
windflex-deploy/
├── README.md                    # 部署说明
├── scripts/
│   ├── check-prerequisites.sh   # 检查系统要求（Docker, Git 等）
│   ├── install-power-driver.sh  # 安装电源驱动（如需要）
│   ├── setup-usb-permissions.sh # 配置 USB 设备权限
│   ├── configure-system.sh      # 系统配置（时区、网络等）
│   └── deploy.sh                # 一键部署主脚本
├── config/
│   ├── udev-rules/              # USB 设备 udev 规则
│   │   └── 99-windflex.rules
│   └── system/                  # 系统配置文件
│       └── windflex.service     # systemd 服务配置
└── tests/
    └── verify-deployment.sh     # 验证部署是否成功
```

**主要功能**:
1. **环境检测**:
   - 检查 Ubuntu 版本
   - 检查 Docker 和 Docker Compose 是否安装
   - 检查 Git 是否安装
   - 检查网络连接

2. **驱动安装**:
   - 安装 USB 转串口驱动（如需要）
   - 配置电源设备通信驱动

3. **权限配置**:
   - 配置 USB 设备访问权限
   - 配置串口设备访问权限
   - 配置 ADB 设备访问权限

4. **一键部署**:
   - 克隆 windflex-system 仓库
   - 创建环境变量配置
   - 启动 Docker 容器
   - 验证服务运行状态

**使用方式**:
```bash
# 1. 克隆部署脚本
git clone https://github.com/yourorg/windflex-deploy.git
cd windflex-deploy

# 2. 运行一键部署
sudo ./scripts/deploy.sh

# 3. 验证部署
./tests/verify-deployment.sh
```

### 12.2 windflex-system（系统代码和配置仓库）
**位置**: `github.com/yourorg/windflex-system`

**目录结构**:
```
windflex-system/
├── README.md                    # 系统说明
├── docker-compose.yml           # Docker Compose 配置
├── .env.example                 # 环境变量模板
├── .gitignore
├── services/
│   ├── hardware-service/        # 硬件控制服务
│   │   ├── Dockerfile
│   │   ├── requirements.txt
│   │   ├── main.py              # FastAPI 主程序
│   │   ├── api/
│   │   │   ├── power.py         # 电源控制 API
│   │   │   ├── adb.py           # ADB 管理 API
│   │   │   ├── serial.py        # 串口通信 API
│   │   │   └── ssh.py           # SSH 连接 API
│   │   └── drivers/
│   │       ├── power/           # 电源驱动
│   │       │   └── dlx30v10amp.py
│   │       └── adb/             # ADB 驱动
│   │           └── adb_manager.py
│   │
│   └── openclaw/                # OpenCLaw (AI Gateway)
│       ├── Dockerfile
│       ├── requirements.txt
│       ├── main.py              # FastAPI 主程序
│       ├── ai/
│       │   ├── llm_gateway.py   # LLM 集成
│       │   ├── test_generator.py # 测试用例生成
│       │   └── failure_analyzer.py # 失败分析
│       └── gitlab/
│           ├── mr_generator.py  # MR/PR 生成
│           └── api_client.py    # Git 平台 API 客户端
│
├── jenkins/
│   ├── Jenkinsfile              # Jenkins 流水线配置
│   └── config.xml               # Jenkins 配置
│
├── docs/
│   └── requirements.md          # 需求文档
│
└── scripts/
    ├── init-db.sh               # 数据库初始化
    └── backup.sh                # 数据备份脚本
```

**主要功能**:
1. **Docker Compose 配置**: 定义所有服务的容器编排
2. **Hardware Service**: 电源、ADB、串口、SSH 控制
3. **OpenCLaw**: AI Gateway，集成 LLM，生成测试用例和分析失败
4. **Jenkins 配置**: CI/CD 流水线定义

**使用方式**:
```bash
# 1. 克隆系统仓库
git clone https://github.com/yourorg/windflex-system.git
cd windflex-system

# 2. 配置环境变量
cp .env.example .env
vim .env  # 编辑配置

# 3. 启动服务
docker-compose up -d

# 4. 查看日志
docker-compose logs -f
```

### 12.3 test-cases（测试用例仓库 - 独立 Git Repo）
**位置**: `github.com/yourorg/board-test-cases`

**目录结构**:
```
board-test-cases/
├── README.md                    # 测试用例说明
├── test-suites/                 # 测试套件
│   ├── power-management/        # 电源管理测试
│   │   ├── power-on.robot       # 开机测试
│   │   ├── power-off.robot      # 关机测试
│   │   └── power-stability.robot # 电源稳定性测试
│   ├── process-monitor/         # 进程监控测试
│   │   ├── powermanager.robot   # powermanager 进程监控
│   │   └── system-services.robot # 系统服务监控
│   └── functional/              # 功能测试
│       ├── ssh-test.robot       # SSH 连接测试
│       └── network-test.robot   # 网络测试
├── resources/                   # 测试资源
│   ├── variables.resource       # 全局变量
│   └── keywords.resource        # 自定义关键字
├── config/                      # 配置文件
│   ├── test-config.yaml         # 测试配置
│   └── power-config.yaml        # 电源配置
├── reports/                     # 测试报告（Git 忽略）
│   ├── log.html
│   ├── report.html
│   └── output.xml
├── logs/                        # 测试日志（Git 忽略）
│   └── *.log
└── .gitignore                   # Git 忽略配置
```

**主要功能**:
1. **测试用例存储**: Robot Framework 测试用例
2. **版本控制**: 跟踪测试用例变更历史
3. **协作开发**: 支持多人协作和 Code Review
4. **AI 生成**: LLM 自动生成测试用例并提交 MR/PR

**贡献者工作流**:

**用户提交测试用例**:
```bash
# 1. 克隆测试用例仓库
git clone https://github.com/yourorg/board-test-cases.git
cd board-test-cases

# 2. 创建新分支
git checkout -b feature/my-new-test

# 3. 编写测试用例
vim test-suites/functional/my-test.robot

# 4. 提交并推送
git add .
git commit -m "Add new functional test"
git push origin feature/my-new-test

# 5. 创建 MR/PR
# 在 GitHub/GitLab Web 界面创建 Merge Request / Pull Request
```

**LLM 自动生成测试用例**:
```bash
# OpenCLaw AI Gateway 自动生成测试用例并提交
# 1. 分析需求文档或失败日志
# 2. 生成 Robot Framework 测试用例
# 3. 自动创建 Git 分支
# 4. 提交代码并创建 MR/PR
# 5. 填写 MR/PR 描述（包含 AI 分析结果）
```

**Jenkins 工作流**:
```
1. Webhook 触发（代码推送或 MR 创建）
   ↓
2. Jenkins 拉取测试用例仓库
   ↓
3. 根据 POWER_MODEL 过滤测试用例
   ↓
4. 执行 Robot Framework 测试
   ↓
5. 生成测试报告
   ↓
6. 上传报告到 Jenkins
   ↓
7. 通知失败分析（OpenCLaw）
```

**Git 忽略配置** (.gitignore):
```gitignore
# 测试报告和日志
reports/
logs/
*.log
*.xml

# 临时文件
*.pyc
__pycache__/
.env

# IDE 配置
.vscode/
.idea/
```

### 12.4 仓库关系图
```
┌─────────────────────┐
│  windflex-deploy    │  部署脚本
│  (Git Repo)         │
└──────────┬──────────┘
           │ 执行部署
           ▼
┌─────────────────────┐
│  windflex-system    │  系统代码
│  (Git Repo)         │  和配置
└──────────┬──────────┘
           │ 加载测试用例
           │ (Jenkins 拉取)
           ▼
┌─────────────────────┐
│  test-cases         │  测试用例
│  (独立 Git Repo)    │  (用户/LLM 提交)
└─────────────────────┘
```
