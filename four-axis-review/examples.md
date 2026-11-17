# Calibration examples

These show the report bar. They are not findings in the current tree.

## Finding

A transaction logs a failed member-resource delete and returns nil.

```markdown
| R01 | Correctness | P1 | internal/logic/member/save.go:88 | Save updates the member, then Delete of resources fails | The closure returns nil, so the member change commits without the resource change | go test ./internal/logic/member -run TestSave_RollsBackWhenResourceDeleteFails |
```

Trigger and impact are both present. The check names one package and one test.

## Intentional contract change

The diff adds a response field, and the OpenSpec change in the same scope documents that field. Put it in Coverage:

```markdown
- Intentional contract change: POST /member/v1/list adds `sortMode`, authorized by openspec/changes/add-member-view-sort
```

Do not also file it as `R0x`.

## Slow SQL

```markdown
| R02 | Performance | P2 | internal/logic/schedule/list.go:64 | List filters with WHERE DATE(start_time) = ? | The function on start_time prevents an index lookup and scans the device's rows on every list | Explain the predicate; compare with a range on the raw start_time column |
```

The cause is the function wrapped around the column. No benchmark is required.

## Large result and memory

```markdown
| R03 | Performance | P1 | internal/logic/schedule/expand.go:41 | A device with a long RRULE history calls Expand | Occurrences are appended with no instance cap and held until the response is built | Put a cap on the expansion and test a rule that would exceed it |
```

## Coupling

```markdown
| R04 | Maintainability | P2 | internal/logic/meal/sync.go:22 | Meal save updates schedule rows by importing schedule's logic package | A schedule rule change has to be edited inside meal, and the dependency direction is reversed | Move the call to the schedule service and keep one owner for the schedule write |
```

## Excluded suspicion

A list handler has no page argument. The query is `Where(device_id).Limit(50)` on the primary key. Exclude it:

```markdown
- List has no client page argument. The query already Limit(50)s by device id equality (internal/logic/widget/list.go:40). Bounded indexed lookup.
```

## Cross-owner lookup

```markdown
| R05 | Security | P1 | internal/logic/task/get.go:30 | Get loads a task by id only | The id of another device's task returns that task | Add the device predicate and test with a task id from a second device |
```

File this once. Do not also file it as Correctness.

## Not a finding

- A function is longer than the surrounding ones, and the steps are still one business operation in order.
- A name could be clearer.
- An equality lookup on an id is assumed slow because no benchmark was attached.
- An admin-only helper has no auth check, and no route or job calls it.
- Legacy `tx.Exec` remains in an untouched DAO file.
- Generic GoFrame guidance would put this rule in `service`. This repository keeps it in `logic`.

## Short report

```markdown
# Four-axis review

## Verdict
One P1 correctness gap: a failed resource delete still commits the member update. The rest of the diff stays inside the existing transaction and auth checks.

## Coverage
- Base: `origin/main`  Head: `abc1234`  Dirty worktree: no
- Read: `internal/logic/member/save.go`, its service caller, and `member_save_test.go`
- Not verified: tests not run; `calendar` schema not inspected

## Findings

| ID | Axis | Priority | Location | Trigger | Impact | Check |
| --- | --- | --- | --- | --- | --- | --- |
| R01 | Correctness | P1 | internal/logic/member/save.go:88 | Resource delete returns an error after the member update | The closure returns nil, so the member row commits without its resources | go test ./internal/logic/member -run TestSave_RollsBackWhenResourceDeleteFails |

## Excluded suspicions
- The handler does not re-check device binding. `DeviceAuth` already rejects a mismatched device before the controller runs (`internal/controller/member/member_v1_save.go:24`).
```
