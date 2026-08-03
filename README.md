# 如何使用

`unzip -o workspace-boundary.skill -d /tmp && /tmp/workspace-boundary/install.sh --anchor`

# 把规则落成硬拦截

技能是行为约定：它靠模型自觉遵守，能覆盖绝大多数情况，但拦不住一次判断失误。真正的强制要靠权限配置。两层都配上，技能负责"知道为什么"，配置负责"确实做不到"。

## Claude Code：`.claude/settings.json`

放在项目根，随仓库提交，团队共享。

```json
{
  "permissions": {
    "deny": [
      "Read(/**)",
      "Read(~/**)",
      "Read(//**)",
      "Read(**/*.db)",
      "Read(**/*.sqlite*)",
      "Read(**/*.dump)",
      "Read(**/*.pkl)",
      "Read(**/*.parquet)",
      "Read(**/dist/**)",
      "Read(**/build/**)",
      "Read(**/uploads/**)",
      "Read(**/data/**)",
      "Read(**/__snapshots__/**)",
      "Bash(cat:~/*)",
      "Bash(sqlite3:*)",
      "Bash(psql:*)",
      "Bash(mysql:*)",
      "Bash(scp:*)"
    ],
    "allow": [
      "Read(./**)",
      "Read(**/*.log)",
      "Read(**/logs/**)"
    ]
  }
}
```

要点：

- `Read(/**)` + `Read(~/**)` 是规则一的主力，`deny` 优先级高于 `allow`，所以 `Read(./**)` 不会把它们抵消掉。
- Bash 的模式匹配是逐命令的，覆盖不全（管道、`env`、脚本调用都能绕）。命令行工具那一层挡的是"顺手打错"，不是恶意规避——真正的隔离靠容器或只读挂载。
- 每加一条 `deny`，同时想清楚它会不会挡住正常工作；被误挡时正确做法是精化模式，不是删掉整条。

## 容器化隔离（更彻底）

只挂载工作目录，且把产出数据目录挂成不可见：

```bash
docker run --rm -it \
  -v "$PWD":/work \
  -v /dev/null:/work/data \
  -w /work \
  --network none \
  <image>
```

在容器里跑 agent，规则一从"约定"变成"物理上不存在的路径"。规则二用空挂载或 `--tmpfs` 覆盖掉 `data/`、`uploads/` 之类目录同理。

## 让模拟数据成为默认可选项

规则二的落地成本主要在"要分析时手边没有假数据"。在项目里预置这些，规则就不再是阻力：

- `make seed` / `npm run seed`：一条命令生成一批模拟数据
- `tests/factories.py`（factory_boy / faker）或 `tests/factories.ts`
- schema 与类型定义放在显眼位置，注释里写清取值范围和业务约束
- 日志里保留可用于对齐量级的统计（每日单量、P95 耗时、错误率）

预置得越齐，越不需要"就看一眼真实数据"。

## CI 校验（可选）

在 CI 里跑一遍 `scripts/guard.py`，对 PR 里新增的分析脚本做一次静态检查，确认没有直接指向禁止路径的读取。这更多是提醒作用，不必做成硬门禁。
