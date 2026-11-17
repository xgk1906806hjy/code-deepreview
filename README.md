# code-deepreview

只读的四轴代码审查 skill。代理按正确性、安全性、性能、可维护性检查 diff，每条发现都要有位置、触发条件和影响。审查只出报告，不改代码、测试、文档或 git 状态。

Skill 正文是英文，供代理执行。本说明说明它查什么、怎么装、怎么用。

## 审查什么

| 轴 | 查什么 |
| --- | --- |
| 正确性 | 功能是否做对，逻辑、边界、状态、失败后是否仍提交，旧数据与部分更新，缓存是否还是旧值 |
| 安全性 | 可达入口上缺了的控制。不要求已经凑成完整攻击链。跨设备只按全局 id 查询，归到这一轴 |
| 性能 | 响应变慢的原因、慢 SQL、大 SQL、无上限内存、深分页、连接或结果集没有关闭 |
| 可维护性 | 高内聚、低耦合、流程是否能在一处看清结果、结构是否无谓复制。改名和把一条直线流程改短，不报 |

优先级只有 `P1` 和 `P2`。`P1` 是错误行为、可达入口缺控制、失败后仍提交、请求或任务上无上限的查询或分配。`P2` 是有上限但原因明确的慢查询或多余延迟，以及会让下一次修改做错的结构问题。

同一条缺陷只记一次，归到影响更重的那一轴。说不清怎么走到那段代码的，放进排除项。

仓库约束写在 `four-axis-review/axes.md` 的 Repository constraints。它针对日历服务的 GoFrame 布局：分层、DAO、双库事务、生成文件和公开接口契约。其他项目仍使用四轴；那些约束只在代码里出现对应写法时成立。

## 目录

```text
four-axis-review/
├── SKILL.md      # 审查步骤、范围和报告格式
├── axes.md       # 四轴检查项和仓库约束
└── examples.md   # 什么样的问题该报、该排除
```

## 安装

在要审查的项目里安装：

```bash
npx skills add xgk1906806hjy/code-deepreview --skill four-axis-review
```

也可以把 `four-axis-review/` 整个目录复制到下面任一位置：

- 项目内：`.agents/skills/four-axis-review/` 或 `.cursor/skills/four-axis-review/`
- 当前用户、所有项目：`%USERPROFILE%\.cursor\skills\four-axis-review\`

三个文件必须留在同一目录。`SKILL.md` 会读取旁边的 `axes.md` 和 `examples.md`。

## 用法

安装后，在目标仓库里直接说要审查。代理读取 `SKILL.md` 并按其中的步骤执行。

默认范围是当前分支相对默认基线的 diff，包含已暂存和未暂存的改动。本地未跟踪的源文件也会纳入。diff 为空时停止，不审查未改动的代码。

可以用这些说法收窄范围：

- `Review the current branch.`
- `四轴审查`
- `Review internal/logic/member for security and slow SQL.`
- `Review the changes since origin/main.`
- `Review this module. State what you did not read.`

整库或整模块审查要单独点名。报告里会写实际读过的入口，以及没跑的测试、没看的表结构、没压测的下游。

审查过程中不要期望它改代码。需要验证时，再单独让它跑受影响包里的某一个测试。

## 报告

```markdown
# Four-axis review

## Verdict
一句话说明能不能合，风险集中在哪里。

## Coverage
- Base: `origin/main`  Head: `<sha>`  Dirty worktree: yes/no
- Read: 实际判断过的入口和文件
- Not verified: 没跑的测试、没看的库表、没量过的下游

## Findings

| ID | Axis | Priority | Location | Trigger | Impact | Check |
| --- | --- | --- | --- | --- | --- | --- |
| R01 | Correctness | P1 | path/file.go:120 | 怎么走到这里 | 会造成什么 | 最小确认方法 |

## Excluded suspicions
- 看起来像问题、但证据排除掉的点。
```

没有发现时，结论里直接说明，覆盖范围和排除项仍然保留。

## 许可

[MIT](LICENSE)
