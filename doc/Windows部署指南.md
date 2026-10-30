# CTBZStock 草台班子量化 Windows 部署指南

本文针对当前仓库，目标是先跑通万和掘金模拟账户，再设置每日自动运行。本文是部署操作手册，尚未在 Windows 上完成端到端验证，不代表该旧版本现在可以直接连接券商服务。

## 1. 先明确部署条件

运行链路：Windows → 万和掘金客户端及其登录会话 → 掘金 Python SDK `gm.api` → 本项目策略 → 模拟账户或证券实盘账户。

- 仓库明确提供的是万和证券接入方案。信达等其他券商不在本手册已核实范围内。
- 模拟盘需要掘金账户及客户端中创建的模拟账户；按仓库说明，模拟盘不要求先开万和证券实盘户。
- 实盘需要券商账户、量化交易权限、客户端绑定的交易账户和有效策略配置。具体开通条件向万和确认。
- SDK 能安装不等于交易链路已连通，必须逐层验收。
- 代码没有回测功能。本手册不通过真实账户试单来验证安装。

## 2. 准备软件和目录

建议使用独立 Windows 10/11 或 Windows Server x64 环境，至少按原文档准备 2 核、2 GB 内存；实际还需留足客户端和依赖运行空间。

准备：

1. 万和掘金客户端：向万和或掘金官方获取当前可用的安装包及 SDK 配套要求。仓库历史链接为 `http://download.vanho.cn/download/juejin/Vanhogm.exe`，本次未验证仍有效；安装前核对来源与签名。
2. Python x64：本仓库文档推荐 3.8～3.10。可优先使用 **3.9 x64 独立虚拟环境做旧版复现**，原因是固定的 numpy/scipy/grpcio 版本较旧；该 Python 版本已结束维护，仅作为兼容性起点。生产长期运行前应评估迁移到券商支持、仍受维护的 Python/SDK 组合。
3. Git for Windows，用于拉取代码。后文启动使用 `.cmd`，无需依赖 Git Bash。
4. 能访问掘金、万和服务、Python 包仓库、东方财富行情；使用飞书通知时还需访问飞书开放平台。

建议目录：

```text
C:\quant\CTBZStock\                 模拟部署代码
C:\quant\CTBZStock\.venv\           Python 虚拟环境
C:\quant\CTBZStock\deploy\          本手册新增的本机启动脚本
C:\quant\CTBZStock\logs\            控制台日志
```

Windows 设置中启用时间同步，将时区设为北京时间（中国标准时间），关闭交易时段自动睡眠及休眠。不要先假定关闭远程桌面后客户端仍可正常运行，需要实测会话行为。

## 3. 获取代码并记录版本

以下命令在 **PowerShell** 执行。将仓库地址替换为自己能访问的真实地址，不把令牌写到命令中。

```powershell
New-Item -ItemType Directory -Force C:\quant | Out-Null
Set-Location C:\quant
git clone <你的仓库地址> CTBZStock
Set-Location C:\quant\CTBZStock
git status --short
git rev-parse HEAD
py -0p
py -3.9 -m venv .venv
.\.venv\Scripts\python.exe -m pip install --upgrade pip setuptools wheel
```

确认选中的 Python 与客户端 SDK 支持的位数、版本一致。记录 commit。不要照搬旧脚本中的 `git checkout develop/master`：你的仓库未必有这些分支，而且运行时切分支会影响部署。

## 4. 修复已确认的配置初始化错误

当前 `schema/config.py` 有必填字段：

```python
cb_start_time: str
```

但模拟、实盘入口都没有传入，初始化会出现 `ValidationError: cb_start_time Field required`。两个配置文件已经定义了 `CB_RUN_TIME_NODE`，因此分别修改：

- `main/mtv_develop/cronjob.py`
- `main/mtv_master/cronjob.py`

在两处 `Config(...)` 内补上：

```python
cb_start_time=cons.CB_RUN_TIME_NODE,
```

例如：

```python
config = Config(
    mtv_start_time=cons.MTV_RUN_TIME_NODE if not MTV_DEBUG_START else get_real_time_run_time_node(),
    mtv_report_time=cons.MTV_REPORT_TIME_NODE if not MTV_DEBUG_REPORT else get_real_time_run_time_node(),
    mtv_switch=cons.MTV_SWITCH,
    cb_start_time=cons.CB_RUN_TIME_NODE,
    trade_start_time=cons.TRADE_START_TIME,
    shipping_space=cons.SHIPPING_SPACE,
    # 后面的现有参数继续保留
)
```

上面是局部示例，不要删除原有参数。修改后用 `git diff` 检查。这份文档没有自动修改策略源代码。

## 5. 安装依赖，先解决兼容性

仓库锁定了 `gm==3.0.171`、`numpy==1.23.0`、`scipy==1.8.1`、`grpcio==1.47.0` 等旧包，而 `pydantic`、`akshare` 等没有锁版本。直接安装今天的最新版组合不保证兼容。

建议复制一份用于部署的依赖清单，保留原始文件：

```powershell
Copy-Item requirements.txt requirements.windows.txt
notepad requirements.windows.txt
```

在复制件中把原来的 `pydantic` 改为：

```text
pydantic>=1.10,<2
```

理由：源代码使用 `.dict()` 等 Pydantic 1 风格，优先缩小旧代码的迁移变量；这不是已经验证的完整依赖锁。保留其余行先尝试安装：

```powershell
.\.venv\Scripts\python.exe -m pip install -r requirements.windows.txt
.\.venv\Scripts\python.exe -m pip check
.\.venv\Scripts\python.exe -c "import gm.api, numpy, scipy, requests, akshare, attrdict, pydantic, loguru; print('imports OK')"
```

必须让安装命令成功退出、`pip check` 无冲突且全部 import 成功，才能继续。

常见分支：

- **gm 旧版本无可用包或客户端拒绝旧 SDK**：先向官方确认支持的 SDK/Python 组合，再调整复制件中的版本；升级 gm 后重新核对所有调用，不用最新版覆盖后直接启用交易。
- **AKShare 与旧 numpy/Python 冲突**：需要选择仍提供 `stock_zh_a_spot_em` 的兼容 AKShare 版本，或协调升级整个依赖组合。当前未验证具体可用版本，不提供猜测的版本号。以解析结果和后续行情检查为准。
- **attrdict 提示 `collections.Mapping` 等导入错误**：检查是否实际用了 Python 3.10/3.11/3.12，优先回到选定的旧版复现环境，或单独评估替换该库。
- **pip 开始编译 numpy/scipy/grpcio 并失败**：核对 Python 版本和 x64 wheel 支持；不要仅为此盲目安装编译工具。

成功后记录实际版本：

```powershell
.\.venv\Scripts\python.exe -m pip freeze > requirements.windows.lock.txt
```

该锁文件只在此次 Windows 组合验收成功后，才是后续复现依据。

## 6. 客户端登录并创建模拟策略

1. 安装并手动启动万和掘金客户端，登录掘金账户。
2. 创建模拟交易账户，建议命名 `develop`。
3. 创建空策略，并明确把它绑定到刚创建的模拟账户。
4. 记录该策略的 strategy ID 和账户所需 token；字段位置以当前客户端为准。
5. 确认交易服务连接正常，能在客户端看到模拟资金与持仓。
6. 将策略 ID 和 token 填入 `config/develop.py` 对应字段。

不要复用仓库里已有的 ID/token 来推定账户归属。不要将新凭证提交到 Git、截图或发送到日志。

**关键区别**：两个 Python 入口都调用 `gmapi.run(..., mode=gmapi.MODE_LIVE)`，这是实时运行模式，不能据此判断绑定的是真钱还是模拟账户。`Mode.SIMU` 是项目自己的标记；账户性质必须以客户端策略绑定为准。

## 7. 配置时间和通知

编辑 `config/develop.py`：

| 配置 | 用途与初次部署建议 |
| --- | --- |
| `STRATEGY_ID` | 新创建、绑定模拟账户的策略 ID |
| `TOKEN` | 与该策略/账户对应的有效凭证 |
| `MTV_SWITCH` | 首次设置为 `False`，禁止注册选股调仓任务 |
| `MTV_RUN_TIME_NODE` | 策略任务时间；目前 develop 默认 `09:28:10` |
| `MTV_REPORT_TIME_NODE` | 持仓报告时间；目前 develop 默认 `15:05:00` |
| `TRADE_START_TIME` | 交易执行放行时间；目前 develop 默认 `09:31:30` |
| `CB_RUN_TIME_NODE` | 补齐 Config 初始化所需字段；当前示例没有据此注册另一项任务 |
| `SHIPPING_SPACE` | 保留项目原值；当前目标仓位计算不能仅凭该名字认定受其控制 |
| `FEISHU_GROUP` | 通知群名称，当前实现按群名称查找 |
| `FEISHU_APP_ID`、`FEISHU_APP_SECRET` | 飞书自建应用凭证 |

初次保持两个入口中的 `MTV_DEBUG_START = False`、`MTV_DEBUG_REPORT = False`。

飞书需创建应用、启用机器人、配置当前 API 所需群与消息权限并发布生效，把机器人加入目标群。旧代码用了旧版群列表及消息接口，需实际验证当前租户可用性；配置空值不等于已经禁用通知，发送失败会被捕获并记日志。

另外，`third_part/feishu_service.py` 会调试输出 token 响应和认证 headers。正式部署前应删除或脱敏这些日志，避免令牌进入日志文件；限制配置和日志目录的读取权限。

## 8. 分层检查，不直接启用调仓

### 8.1 语法、导入和配置构造

在项目根目录执行：

```powershell
.\.venv\Scripts\python.exe -m compileall -q main schema strategy transaction utils third_part config
.\.venv\Scripts\python.exe -c "from schema.config import Config; from schema.status import Mode; from config import develop as c; Config(mtv_start_time=c.MTV_RUN_TIME_NODE, mtv_report_time=c.MTV_REPORT_TIME_NODE, mtv_switch=c.MTV_SWITCH, cb_start_time=c.CB_RUN_TIME_NODE, trade_start_time=c.TRADE_START_TIME, shipping_space=c.SHIPPING_SPACE, mode=Mode.SIMU, feishu_group=c.FEISHU_GROUP, feishu_app_id=c.FEISHU_APP_ID, feishu_app_secret=c.FEISHU_APP_SECRET); print('Config OK')"
```

成功标准：无语法错误、无缺包、输出 `Config OK`。不要打印整个配置对象，因为里面有通知凭证；入口原有 `logger.info` 输出完整 config 的那一行也应删除或改成不含凭证的简短消息。

### 8.2 独立检查外部行情

```powershell
.\.venv\Scripts\python.exe -c "import akshare as ak; d=ak.stock_zh_a_spot_em(); needed={'代码','名称','最新价','今开','涨跌幅','昨收','成交额','成交量'}; print('rows:',len(d)); print('missing:',needed-set(d.columns))"
```

成功标准：非空、缺失字段集合为空。该检查只能证明当时能获取东方财富快照，不证明盘中每次请求都可靠。

### 8.3 启动连接和只读报告

先打开并登录客户端，确认 `MTV_SWITCH=False`，再运行：

```powershell
Set-Location C:\quant\CTBZStock\main\mtv_develop
& C:\quant\CTBZStock\.venv\Scripts\python.exe -u cronjob.py
```

必须从该入口目录运行：目前源码用相对路径 `../../` 添加项目根目录。日志文件也落在入口目录下的 `data`。

成功标准：无异常退出、出现初始化/交易服务连接日志、客户端策略连接状态正常。若需提前验证只读报告，可将 `MTV_REPORT_TIME_NODE` 改为北京时间几分钟后的时间，再重启；普通报告只在交易日注册，非交易日应等待交易日或仅在确认模拟账户后使用 `MTV_DEBUG_REPORT=True` 做报告检查，完成后恢复。

注意：`MTV_DEBUG_START=True` 会触发调仓任务，不可用于只读连通性检查。无订单日志也可能仅因未到时间或当天非交易日，不能单凭进程存在判断跑通。

### 8.4 模拟调仓验收

完成上述检查后：

1. 再核对客户端策略绑定模拟账户。
2. 停止旧进程，将 `MTV_SWITCH=True`，恢复原交易时间，保持两个 DEBUG 开关为 False。
3. 在交易日任务时间之前启动并保持客户端在线。不要首次在盘中任意改时间补跑。
4. 检查股票池、市值数据、资金持仓和目标订单是否生成成功。
5. 在客户端确认模拟委托、成交、撤单以及最终持仓与日志一致。
6. 检查先卖后买、部分成交、未成交撤单与重报行为；不要只检查“已发单”日志。
7. 确认收盘报告正常，并观察下一交易日是否正常启动。

若订单生成阶段出现空候选列表、字段变更、索引或网络异常，应保留脱敏堆栈，先修复再继续。当前策略有固定数量索引及除法逻辑，空数据/不足数量属于需要实际检查的情况。

## 9. 创建稳定的 Windows 启动脚本

在根目录创建 `deploy` 和 `logs`：

```powershell
New-Item -ItemType Directory -Force C:\quant\CTBZStock\deploy, C:\quant\CTBZStock\logs | Out-Null
```

保存以下内容为 `C:\quant\CTBZStock\deploy\run-sim.cmd`：

```bat
@echo off
setlocal
set "PROJECT_DIR=C:\quant\CTBZStock"
if not exist "%PROJECT_DIR%\logs" mkdir "%PROJECT_DIR%\logs"
cd /d "%PROJECT_DIR%\main\mtv_develop"
"%PROJECT_DIR%\.venv\Scripts\python.exe" -u cronjob.py >> "%PROJECT_DIR%\logs\sim-console.log" 2>&1
exit /b %errorlevel%
```

双击或从 CMD 执行验证一次，确认日志能写入。之后检查没有遗留重复进程。

客户端启动脚本可保存为 `deploy\start-client.cmd`：

```bat
@echo off
setlocal
rem 按实际快捷方式属性中的目标路径修改，不要原样照抄。
set "CLIENT_EXE=C:\Users\YOUR_USER\AppData\Roaming\Vanho Goldminer3\vanhogm3.exe"
if not exist "%CLIENT_EXE%" exit /b 1
start "" "%CLIENT_EXE%"
```

首次需人工完成登录。自动打开程序不保证自动登录成功，是否支持记住登录及无人值守请按当前客户端规则验证。

## 10. 设置 Windows 任务计划程序

先用模拟盘完成手动验收，再自动化。

### 任务 A：客户端

- 操作：程序 `C:\Windows\System32\cmd.exe`。
- 参数：`/c "C:\quant\CTBZStock\deploy\start-client.cmd"`。
- 起始于：`C:\quant\CTBZStock\deploy`。
- 触发器：工作日北京时间 09:15，或按客户端连接所需时间提前。
- 使用安装和登录客户端的同一 Windows 用户；初次选“仅当用户登录时运行”。
- 定时启动后人工确认客户端已登录并连接成功，再考虑无人值守。

### 任务 B：模拟策略

- 操作：程序 `C:\Windows\System32\cmd.exe`。
- 参数：`/c "C:\quant\CTBZStock\deploy\run-sim.cmd"`。
- 起始于：`C:\quant\CTBZStock\deploy`。
- 触发器：工作日北京时间 09:23，必须早于 `09:28:10` 策略时间。
- 同一用户、“仅当用户登录时运行”。需要后台运行时另行验证客户端会话条件。
- “如果任务已在运行”：选择“不启动新实例”。
- 首次关闭失败后自动重启，避免交易期间不明状态下重复执行策略。
- 首次关闭“错过计划后尽快运行”，避免下午开机补启动被误解为已执行早盘任务。
- 不设置会在收盘报告前终止进程的短运行时限。

Windows 工作日并不等于交易日；代码还会使用交易日判断。手动启动一个进程后，计划任务的“不启动新实例”不能识别它，因此不要手动和计划任务同时启动。

原 `tool` 下脚本硬编码旧电脑目录并包含切分支操作，建议使用上面的本机脚本。旧文档的 `develop.sh/shutdown.sh` 与实际 `mtv_develop.sh/shuwdown.sh` 名称也不一致，不照搬旧文档路径。

## 11. 每日检查与停止

开盘前：确认系统时间、客户端登录、模拟账户、策略唯一实例、凭证有效性及数据接口可用。

收盘后：查看 `logs\sim-console.log`、`main\mtv_develop\data` 中日志及客户端成交记录，核对持仓、资金、撤单、异常和通知。日志需要按日期归档，避免无限增长，并注意现有日志可能包含凭证。

停止手动进程可在前台按 Ctrl+C。计划任务可先使用“结束”，随后在任务管理器确认该命令及其 Python 子进程已退出。停止进程不保证撤销券商端挂单，需要到客户端核对和处理未完成委托。

**不要执行原 `tool/shuwdown.sh` 作为通用停止方案**：它会强制结束所有 `python.exe` 和 `vanhogm3.exe`，可能影响其他项目。仅终止此次部署对应的进程；重启前先确认旧实例确已结束。

## 12. 切到实盘前的独立步骤

实盘不属于“模拟部署成功就自动启用”的下一步，应独立核对：

1. 券商确认实盘量化权限、客户端/SDK组合及账户支持。
2. 在客户端创建并绑定真实交易账户的独立策略，取得新的 ID/token。
3. 使用独立目录、虚拟环境和日志，例如 `C:\quant\CTBZStock-live`，避免与模拟部署混用配置。
4. 修改 `config/master.py`，先保持 `MTV_SWITCH=False`，完成只读资金持仓核对。
5. 确认 `main/mtv_master/cronjob.py` 同样补齐 `cb_start_time`，且不输出凭证。
6. 单独验证策略风控和订单状态处理，再决定是否启用真实交易。当前代码未证明具备完整的生产风控。
7. 实盘启动脚本的工作目录改为 `main\mtv_master`，时序必须早于 master 默认策略时间 `09:25:10`，不能直接沿用模拟盘全部设置。

当前示例 `SHIPPING_SPACE` 虽进入配置，但生成目标资金时主要使用资金加持仓市值等分，不应把调低它当成已经实现小资金限额。需要单独实现并验证预算限制。

## 13. 常见故障定位

| 现象 | 先检查 |
| --- | --- |
| `No module named gm` | 是否使用 `.venv\Scripts\python.exe`，gm 安装是否成功 |
| 找不到 `schema/utils/config` | 当前工作目录是否为 `main\mtv_develop` |
| `cb_start_time Field required` | 两个入口是否补上 `cb_start_time=cons.CB_RUN_TIME_NODE` |
| token/策略/账户错误 | 客户端绑定、配置来自哪一个账户、SDK与服务兼容性 |
| 进程在线但没有选股任务 | `MTV_SWITCH`、交易日、时区、是否错过注册的任务时间 |
| 东方财富数据为空或字段报错 | AKShare 实际版本、返回字段、网络状态与访问限制 |
| 飞书失败 | 机器人入群、应用权限、群名称、旧 API 可用性 |
| 只有发单日志没有成交 | 客户端委托状态、拒单原因、价格、资金持仓、撤单/重报状态 |
| 手动可运行而计划任务失败 | Windows 用户、工作目录、venv绝对路径、登录会话与日志权限 |
| 重复委托或重复日志 | 手动与定时进程是否重复、是否发生失败自动重启 |

## 14. 部署完成验收表

- [ ] 已记录代码 commit、Python 与客户端版本。
- [ ] 已修复两个入口的必填配置缺失。
- [ ] 依赖安装、`pip check`、导入和配置构造全部成功。
- [ ] 已记录实际成功的 Windows 依赖锁文件。
- [ ] 客户端策略确认绑定模拟账户。
- [ ] 只读连接、资金持仓和行情字段检查成功。
- [ ] 已在交易日核对模拟下单、成交及最终持仓。
- [ ] 飞书通知验证完成，或明确记录为尚未启用。
- [ ] 已去除凭证日志，配置与日志权限合适。
- [ ] 计划任务在实际登录/远程桌面使用方式下验收通过。
- [ ] 不会重复启动，掌握停止方式与未完成委托核查方法。

只有安装成功称为“环境装好”；连接和只读检查通过称为“服务连通”；实际模拟订单与持仓核对通过，才能称为“模拟交易链路跑通”。
