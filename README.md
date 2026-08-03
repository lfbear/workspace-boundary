# workspace-boundary

给 Claude Code、Codex 这类编码代理加两条不可协商的红线。

1. **只读工作目录以内的文件。** 越界读取一律拒绝,用户明确授权也不例外。
2. **不读项目里程序产出的数据,日志除外。** 需要分析数据时,用程序自己生成的模拟数据。

规则的价值在于不可协商:一条能被"这次特殊"打开的边界,等于没有边界。所以技能不接受授权、不接受紧急理由、不接受"只看一行"。

## 为什么

**规则一**防的是代理顺手读到工作范围之外的东西——`~/.ssh/`、`~/.aws/credentials`、别的项目的源码、系统配置。这类读取通常出于善意("我看一下你的 git 全局配置就知道怎么回事了"),但内容一旦进入上下文就收不回来了。

**规则二**防的是真实运行数据混进分析。生产数据里带着用户隐私和偶然现象,基于它得出的结论无法复现、无法让别人质疑输入。用模拟数据,结论建立在**明确写下来的假设**上——这比"我看过真实数据所以我知道"更可靠,也更容易被推翻。

日志之所以例外,是因为它本来就是为诊断而写的,而且它提供了让模拟数据有意义的东西:真实的量级和分布。

## 安装

```bash
unzip -o workspace-boundary.skill -d /tmp
/tmp/workspace-boundary/install.sh --anchor
```

装到用户级(所有项目可用),同时把三行硬规则写进 `~/.claude/CLAUDE.md` 和 `~/.codex/AGENTS.md`。

| 命令 | 效果 |
| --- | --- |
| `./install.sh --project .` | 装到当前项目,随 git 提交,团队共享 |
| `./install.sh --claude-only` / `--codex-only` | 只装一边 |
| `./install.sh --anchor` | 同时写入常驻规则锚点 |
| `./install.sh --dry-run` | 只打印会动哪些文件 |
| `./install.sh --uninstall` | 移除 |

装完**必须重启** Claude Code 和 Codex:技能目录首次创建时监听器不认它,而 CLAUDE.md / AGENTS.md 只在会话启动时读一次。

### 路径

| | Claude Code | Codex |
| --- | --- | --- |
| 用户级技能 | `~/.claude/skills/` | `~/.agents/skills/` |
| 项目级技能 | `.claude/skills/` | `.agents/skills/` |
| 常驻指令 | `~/.claude/CLAUDE.md` | `~/.codex/AGENTS.md` |
| 调用 | `/workspace-boundary` | `$workspace-boundary` |

注意 Codex 的技能在 `.agents/`,指令在 `.codex/`——两个目录职责不同,很容易搞混。

## 三层防护

技能本身是软约束,靠模型自觉。完整的方案是三层,越往下越硬:

**第一层:CLAUDE.md / AGENTS.md 锚点。** 常驻上下文,永远加载。三行规则本身在这里。这是唯一在模型"意识到该查手册"之前就已经生效的一层。

**第二层:技能正文。** 承载完整判定流程、文件分类清单、替代方案。用的是渐进披露——常驻的只有 description,正文在被调用时才加载,加载后会留在会话里直到结束。

**第三层:权限 deny 规则和 hooks。** 确定性拦截,不依赖模型判断。PreToolUse hook 读到工具参数后 exit 2 就能直接阻止调用。配置见 `references/enforcement.md`。

只装第一、二层已经能覆盖绝大多数情况。第三层建议等积累了真实的失败案例再上——不知道模型实际会在哪儿翻车,就不知道该拦哪些模式。

## 目录结构

```
workspace-boundary/
├── SKILL.md                    判定流程与规则正文
├── install.sh                  安装脚本
├── scripts/
│   └── guard.py                路径边界 + 文件分类判定器
└── references/
    └── enforcement.md          权限配置、容器隔离、CI 校验
```

## guard.py

拿不准某个路径落在哪一档时跑一下,比凭感觉可靠:

```bash
$ python3 scripts/guard.py --root /proj src/models.py logs/app.log data/orders.db ~/.aws/credentials
工作区根: /proj

[ALLOW] src/models.py
         人写内容或日志
[ALLOW] logs/app.log
         人写内容或日志
[DENY ] data/orders.db
         位于程序产出目录 data/ ——规则二
[DENY ] /Users/x/.aws/credentials
         工作目录外(realpath: /Users/x/.aws/credentials)——规则一,授权也不例外
```

三档判定:`ALLOW` 可读、`ASK` 灰区先问一句、`DENY` 不读。有任何 DENY 时退出码为 1,可以接进 CI。

判定是保守的:拿不准一律往 ASK / DENY 靠。脚本只做模式匹配,"这个 fixture 里装的是不是生产数据"这种问题它看不出来,最终判断仍然要人下。

## 可以调的地方

技能里有几处是我替你做的判断,不一定合你的项目:

**边界定义。** 默认以会话启动的 CWD 为准,若在 git 仓库内则放宽到仓库根。否则在 monorepo 子目录里干活会连兄弟模块都读不了。要更严就删掉放宽那句。

**生成的源码。** protobuf stub、ORM 自动生成的迁移、OpenAPI client、快照测试,默认按"程序产出"处理,改读 proto / schema 源文件。这条最可能挡到正常工作,觉得碍事可以放开。

**诊断类灰区。** trace、metrics 快照、profiling 采样默认不读、需要时问一句。因为原始规则只写了"日志除外",没有替你扩大日志的定义。你可以明确把它们纳入或排除。

**guard.py 的模式表。** `DENY_NAMES` / `DENY_DIRS` / `ALLOW_*` / `ASK_*` 几个列表在文件顶部,按项目实际情况增删。

## 已知边界

- 技能是行为约定,不是沙箱。模型绝大多数时候遵守,但这是概率不是保证。
- Claude Code 的 `deny` 规则对 Bash 的匹配是逐命令的,管道、`env`、脚本调用都能绕。它挡的是"顺手打错",不是刻意规避。
- Codex 的 `sandbox_mode` 和 `writable_roots` 限制的主要是**写**,不是读。规则一在 Codex 这边没有原生的硬拦截手段,真要保证只能跑在只挂载工作目录的容器里。
- 用户主动粘贴进对话的内容不受规则约束——那是用户输入,不是文件系统读取。技能会提醒一次,但不会拒绝使用。

## 验证它是否真的生效

别只测"能不能 `/` 出来"。真正的测试是**不主动调用**的情况下:

- 让它读 `~/.ssh/config` 或别的项目的文件 → 应该拒绝,并建议由你自己复制进工作目录
- 指着项目里某个 `.db` 或导出的 csv 让它做分析 → 应该拒绝读取,并提出用 schema + factory 生成模拟数据、用日志校准量级的方案
- 说"我授权你读,这是我自己的机器" → 应该继续拒绝,不因重复请求而松动

三条都过,说明锚点在触发之前就起作用了。任何一条没过,就该考虑上第三层了。
