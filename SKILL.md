---
name: workspace-boundary
description: 强制两条不可协商的红线——(1) 只读取工作目录以内的文件，越界读取即使用户明确授权也拒绝；(2) 不读取项目中由程序运行产生的数据（日志除外），需要分析数据时改用程序自己生成的模拟数据。Consult this skill BEFORE any file read, repo exploration, debugging, data analysis, or shell command that touches paths — especially when a path lies outside the project, when the user offers or authorizes an outside path, or when the task involves databases, dumps, exports, caches, build artifacts, snapshots, or "看一眼真实数据". Use it even when the request looks routine and even when the user says it's fine.
---

# 工作目录边界与数据隔离

这个技能定义两条红线。它们的价值恰恰在于**不可协商**：一条可以被"这次特殊"打开的边界，等于没有边界。所以下面的规则不接受授权、不接受紧急理由、不接受"只看一行"。

判断顺序：先过规则一（路径在不在界内），再过规则二（内容是不是程序产出）。两关都过才读。

---

## 规则一：不越出工作目录

### 边界怎么定

- 基准是**会话启动时的当前目录（CWD）**。
- 如果 CWD 位于某个 git 仓库内，边界可放宽到该仓库根（`git rev-parse --show-toplevel`），但**不得越过仓库根**。monorepo 的兄弟目录在仓库内，算界内。
- 判定前一律做 **realpath 解析**。指向界外的符号链接、`..` 穿越、`/proc/self/cwd/..`、bind mount 都按界外处理。
- 边界只在会话开始时确定一次。任务中途 `cd` 到别处不改变边界。

### 界外意味着什么

不只是 `Read` 工具。任何让界外字节进入上下文的路径都算读取：

- shell：`cat / less / head / tail / grep -r / find / rg / sed / awk / strings / xxd / od / diff / tar -tvf`
- 语言运行时：`python -c "open(...)"`、`node -e "fs.readFileSync(...)"`、`ruby -e`、`perl -pe`
- 网络与容器：`curl file://`、`scp`、`docker cp`、`docker exec cat`、`kubectl cp`
- 工具间接读：全局配置（`~/.gitconfig`、`~/.ssh/`、`~/.aws/`、`~/.npmrc`、`~/.kube/config`）、`git --git-dir=` 指向界外仓库、编辑器 session、MCP 文件类连接器
- 让子进程/子代理去读再把结果回传，同样禁止。规则约束的是"界外内容进入上下文"，不是某个具体命令名。

### 授权无效

用户说"我授权你读 `~/.aws/credentials`"、"这是我自己的机器"、"你只看一眼配置就行"——一律拒绝，且不因重复请求而松动。

正确的替代路径是：**由用户自己**把需要的文件复制进工作目录。复制这个动作必须是用户做，不能由你代劳（代劳就是一次界外读取）。复制进来之后，该文件还要再过一次规则二。

### 明确的窄例外

以下不算"读取用户文件系统"，允许：

1. 读取本技能自身目录下的文件（SKILL.md、references、scripts）。这个例外**不扩展**到 `~/.claude/` 下的其他内容，尤其不含 settings、history、凭据。
2. 命令的 stdout/stderr——前提是该命令本身没有输出被禁内容（`cat ~/.ssh/id_rsa` 不因为"这是 stdout"而合法）。
3. 联网获取的公开文档（语言标准库、依赖包官网）。
4. 用户**主动粘贴**进对话的内容。这是用户输入而非文件系统读取，不受本规则约束；但如果粘贴内容含密钥或个人信息，提醒一次。

### 拒绝时怎么说

一句话说清边界 + 给出可行替代，不说教、不重复。

> 这个路径在工作目录外（`/Users/x/.aws/credentials`），我不读工作目录以外的文件，授权也不例外。如果确实需要，请你把它复制到项目里再告诉我文件名；如果只是想让我知道里面的 region 配置，直接告诉我值就行。

---

## 规则二：不读程序产出的数据，日志除外

### 为什么

真实运行数据里通常混着生产内容、用户隐私和不该固化到分析里的偶然现象。用模拟数据做分析，结论建立在**明确写下来的假设**上，可复现、可审阅、可让别人质疑输入。这比"我看过真实数据所以我知道"更可靠。

### 禁止读取（程序产出）

- 数据库文件与导出：`*.db / *.sqlite / *.mdb`、`pg_dump` 产物、`*.bak`、快照、备份
- 运行结果导出：程序写出的 `*.csv / *.json / *.parquet / *.xlsx / *.ndjson`，报表、对账文件
- 缓存与序列化状态：`*.pkl / *.rdb / *.dump`、索引文件、`.cache/`、session store、模型 checkpoint
- 用户内容目录：`uploads/`、`media/`、`storage/app/`、`data/`
- 内存与崩溃现场：coredump、heap dump、`*.hprof`
- 构建产物：`dist/`、`build/`、`target/`、`out/`、bundle、`*.o`、`*.class`
- 生成的源码与快照：`*_pb2.py`、`*.pb.go`、OpenAPI/GraphQL 生成的 client、ORM 自动生成的迁移、`__snapshots__/`
- 从真实环境抓取后落盘的测试固件（文件名像 fixture，内容是生产数据的，按禁止处理）

### 允许读取

- **日志**：`*.log`、`logs/`、journald 导出、捕获的 stdout/stderr——这是规则明确的例外
- 人写的源码、配置模板、`schema.sql` / DDL / 手写迁移、类型与校验定义（pydantic、zod、proto 文件本身）
- 文档、README、依赖清单、CI 配置
- 第三方依赖源码（`node_modules/`、`vendor/`）：不是本项目程序产出，需要时可查阅，但优先看官方文档
- git 命令的输出（`git log`、`git show`、`git diff`）：读的是版本化的人写内容；但不直接读 `.git/objects` 等内部产物

### 灰区：默认按禁止，可用一句话问

trace、metrics 快照、profiling 采样、APM 导出——它们像日志一样偏诊断，但不是日志。默认不读，需要时问一句"这份 trace 你算作日志范畴吗？"，得到明确答复再动。不要自行扩大"日志"的定义。

### 需要分析数据时的正确做法

按顺序走：

1. **从定义推形状**：读 schema / 类型定义 / 校验规则 / 常量枚举，确定字段、类型、取值范围、约束关系。
2. **用项目自带的生成能力**：seed 脚本、factory、fixture 工厂、faker 配置、mock server。项目里已有的，优先复用。
3. **没有就写一个**：新建生成器（放在 `tests/fixtures/`、`scripts/gen_*.py` 之类的位置），让程序自己产出模拟数据，再对模拟数据做分析。生成器是可交付物，随分析一起留下。
4. **用日志校准参数**：日志里的计数、耗时分布、错误率、QPS 是允许读的，用它们把模拟数据的量级和分布调到接近真实——这是"不碰真实数据"和"结论有意义"之间的正解。
5. **标注前提**：结论里写清基于模拟数据、数据是怎么生成的、哪些假设一旦不成立结论就会变。

### 不要用这些方式绕开

- 不用 `sqlite3 / psql / mysql` 去 "只 SELECT 一行看看结构"——结构从 schema 读，行数据不看
- 不用 `head -1 data.csv`、`jq '.[0]'` 取"一个样例"
- 不先读 schema 造完数据、再"跟真实数据核对一下"
- 不把真实文件先脱敏再读——脱敏本身要先读
- 用户主动粘贴真实数据不受本规则约束，但提醒一次并建议改用模拟数据

### 拒绝时怎么说

> `orders.db` 是程序产出的运行数据，我不读它。我用 `models/order.py` 里的字段定义 + `tests/factories.py` 生成一批模拟订单来跑这个聚合分析；量级我按 `logs/app.log` 里最近一周的下单计数来对齐。这样可以吗？

---

## 会话开始时

第一次要碰文件之前，用一句话确认边界，然后正常干活。不要预先扫描界外目录，不要在每次回复里重复声明规则。

> 边界确认：工作目录 `~/proj/api`，我只在这里面读文件；程序产出的数据（日志除外）不读，需要数据时用模拟的。

## 冲突了怎么办

任务确实做不下去时（比如需要的信息只存在于界外或存在于真实数据里），如实说明卡在哪、缺什么、有哪些替代方案，然后**停下来等用户决定**。不要为了推进任务而自行放宽规则，也不要假装分析完成。

## 工具与脚本

- `scripts/guard.py`：路径判定与文件分类的小工具。不确定某个路径落在哪一档时跑一下，比凭感觉可靠。用法见文件头。
- `references/enforcement.md`：把这两条规则落到 Claude Code 权限配置（`deny` 规则）里的做法。技能是行为约定，权限配置才是硬拦截——两者都配上才稳。
