# Review axes

Apply these checks to the inventory from the skill. A matching pattern becomes a finding only when the trigger and impact are real for this diff. When one defect matches two axes, file it once under the axis with the worse impact.

## Correctness

Judge logic and behavior. A finding is a path that returns the wrong result, persists the wrong result, or drops work the caller asked for.

- Function: the branch performs a different operation from the request or the spec in scope. Examples: update-one also rewriting the series, a delete that leaves the occurrence visible, a filter that keeps the rows it should remove.
- Logic: inverted condition, off-by-one, overflow, truncated conversion, a case that falls through to the wrong default, or a comparison that treats omitted, zero, and explicit clear as the same value.
- Boundaries: nil, empty, duplicate ids, the first and last element, a window that starts on the end instant, and the maximum count the API accepts.
- State: a terminal status with no retry, a retry that repeats a completed effect, a conditional update that ignores a stale revision, or a sweep that treats still-valid work as finished.
- Failure: an error logged and then discarded with `return nil` inside a transaction; a later step fails after an earlier write; a side effect starts before commit. Across `default` and `calendar`, the second write needs an explicit compensation when the first has committed.
- Concurrency: lost updates, check-then-act across requests, unique-key races that create duplicate active rows.
- Derived data: a stored total, sort key, projection, or link row that some write paths do not update.
- Time: timezone, all-day versus timed, window edges, recurrence exceptions that fail to suppress the original occurrence, duration that drops days.
- Contract: callers still treat an empty result, a zero value, or an error as the old meaning. Map a domain failure to the error the API already documents.
- Compatibility: a new required field, a stricter validation, or a new enum rejects a row already stored or a request the current clients still send.
- Patch: an omitted pointer, a zero, and an explicit null keep the meaning that API already uses. A partial update must not clear fields the client did not send.
- Freshness: after a write, a cache, list, or projection on that path still returns the previous value.
- Lifetime: request work uses `context.Background()` or continues after the handler returns, so the caller's cancel and deadline are lost.
- Shared memory: a package-level map or slice is written by concurrent requests without synchronization.
- Limits: a value longer than the column or the API limit is truncated and the call still succeeds.
- Soft delete: inserting the same natural key again collides with the deleted row, or updates that deleted row.
- Identity: an id that changes after a split, a merge, or a migration, so later work no longer matches the original row.

## Security

Judge reachable vulnerabilities, including missing controls that are not yet turned into a full exploit. Name the entry, the missing control, and the asset.

- Authorization is enforced on the server for this resource and this action, on reads, lists, writes, batch, and delete. A filter in the caller or the UI is not enough. Batch ids are checked per id.
- Device binding, member binding, and calendar ownership are checked on every path that uses those ids. A lookup or update keyed only by a global id, with no device, member, or calendar predicate, can match another owner's row. File that under Security. The same missing predicate that stays inside one device and hits the wrong member is Correctness.
- A new route, job, or branch with no auth decision, or a failed auth check that falls through, is a finding.
- Automated or webhook callers are classified as a source. That classification does not grant permission. A retried mutation that is not naturally idempotent needs a dedupe key.
- Input is bounded and typed before it reaches a query, a command, a path, a URL, a template, or a log. Concatenated SQL, a fetched URL, or a filesystem path taken from the request is a finding.
- Queries use the generated DAO. String-built SQL, `g.DB().Model`, and `g.DB().Exec` in production logic are findings when the diff adds them.
- Secrets, tokens, SQL, and another caller's rows do not appear in logs, errors, or responses.
- A public API change (path, method, request field, response field, or validation) with no matching spec or change note in the reviewed scope is a `P1` finding. When the spec in scope authorizes it, record it under Coverage as an intentional contract change.

## Performance

Judge response latency, slow SQL, large SQL, and memory. Name the cause. A missing benchmark is a coverage limit, not a reason to drop a visible query or allocation defect. A bounded lookup on an equality key is not a finding.

- Latency: database or remote calls in series on the request path where one batch would do; an outbound call with no timeout that blocks the response; request-path work that the module already runs in a job.
- Slow SQL: a changed `Where`, `Order`, or `Join` that wraps the column in a function, uses a leading wildcard, or compares it to another type, so an index on the raw column cannot be used. One query per parent row. A transaction that holds row locks across a remote call. When the repo has no index definition for that predicate, file it as `medium` and say the schema was not inspected.
- Large SQL: `SELECT *` or a scan of unused columns on a wide table. An `IN` list or bulk insert built from unbounded input. A join that multiplies rows before `Limit`. A query that returns the full history when the caller asked for one page. A deep `OFFSET` on a table that grows with history. An `ORDER BY` that does not match the index used by `WHERE`; when no index definition is in the repo, file it as `medium` and say the schema was not inspected.
- Memory: an unbounded result loaded into a slice or map. `io.ReadAll` or a JSON decode of an unbounded body. Recurrence expansion, string growth, or a cache insert with no cap. One goroutine per item with no limit. Query rows or response bodies left open, so connections stay checked out. A hot path that logs or marshals the full payload.
- Locks and retries: lock order that pins a hot row across unbounded work. A job that retries by replaying the whole history.

`P1` when one request or job can grow without a limit. `P2` when the query or call is bounded but the cause of the extra latency or width is visible.

## Maintainability

Judge cohesion, coupling, flow, and simplicity. File a finding when the structure is the defect, not when a name or a line count could be nicer.

- Cohesion: one module, file, or function owns unrelated rules. Protocol parsing, authorization, and persistence for different operations belong apart. Two operations that do not share data do not share a write procedure.
- Coupling: one module reaches into another module's logic, dao, or private types. A cross-module call skips `service`. A shared helper grows a branch that only one caller needs.
- Flow: the steps of one business operation are split so the failure outcome is not visible in one place, or a callback continues after the operation has already succeeded or failed.
- Simplicity: the diff copies an existing procedure and edits a few lines, or adds a layer that forwards a call without making a decision. A single linear procedure can stay long. The same status or transition encoded as raw numbers in several places is a finding, because a new state has to be found and edited in each copy.
- Dependency direction is `api -> controller -> service -> logic -> dao/model`. A reverse import, or a controller that touches the database, is a finding.
- `api` holds protocol and comments. `controller` converts parameters and calls services. Rules, state machines, and transactions stay in `logic`. `dao` does not encode workflow.
- Hand-written DAO, DO, entity, or service-interface files, or a generated `internal/service` file that imports `controller`, `logic`, or tests.
- Docs in the reviewed scope describe a different contract or state machine than the code, and a later change would follow the doc.

## Repository constraints

These are mandatory for this repository. Read the matching section of `.ai/05-模块清单-职责与代码边界.md` when a hit needs the exact allowed exception.

### Layering

- Cross-module calls go through `service`.
- `controller` does not query or write the database.
- Do not recommend moving business rules out of `logic` into `service`.

### Data access

- Business reads and writes use `dao.Xxx.Ctx(ctx)`.
- New production code does not add raw SQL, `tx.Exec`, `g.DB().Model`, `g.DB().Exec`, or `Data(g.Map{...})`.
- Writes use `do` structs. A `nil` `do` field is omitted, not SQL `NULL`. Clearing a column requires an explicit `gdb.Raw("NULL")` on that field.
- `Scan` targets `*entity` or `[]*entity`. A missing row is a nil pointer or an empty slice with `err == nil`.
- Row locks use `LockUpdate()`.
- Do not hand-write `WhereNull(deleted_at)`. Soft delete is automatic. Hard delete uses `Unscoped().Delete()`.
- `created_at` and `updated_at` are framework-managed unless the business operation is specifically a timestamp change.
- Schema changes come from `make gen_all`. Flag DDL added under production logic.

Existing raw SQL in legacy DAO files is not a new finding. A diff that copies that pattern into new code is a finding.

### Transactions

- Multi-step writes in one database use `dao.Xxx.Ctx(ctx).Transaction(ctx, func(ctx context.Context, _ gdb.TX) error { ... })`.
- Work inside the closure uses the closure `ctx` and `dao.Xxx.Ctx(ctx)`.
- The closure returns the error. Logging it and returning nil commits the partial work.
- Do not add manual `Commit` or `Rollback`, and do not thread `gdb.TX` through new helpers.
- A transaction covers one database group. `default` and `calendar` stay in separate transactions. Logic orders the commits and the compensation. There is no cross-database SQL `JOIN` and no fake distributed transaction.
- Prefer `dao.Xxx.Ctx(ctx).Transaction` so an outer custom `Ctx` still runs. `dao.Xxx.Transaction` alone skips that outer `Ctx`.

### Generated files

- DAO, DO, entity, service, and controller generation is updated with `make gen_all`.
- Review the full generation diff. Flag imports in `internal/service` that pull in `controller`, `logic`, tests, or the service package itself.

### Request context

When the diff touches HTTP, RPC, queue, cron, or MQTT entry points:

- The entry binds a fresh typed `RequestContext`. Business code does not read transport-private context keys.
- JWT identity is converted to `Principal` at the middleware boundary.
- `IsAutomatedMutation` and webhook source do not grant authorization.
- Work after the response may use `WithoutCancel` only with a new explicit timeout.
- Goroutines do not capture `*ghttp.Request` or the response. They receive an immutable DTO built during the request.
- Business cache and logic stay on color version v1. `UIColorVersion` is converted at the HTTP boundary only.

### Tests and delivery

- A suggested check names the affected package and test. It does not suggest `go test ./...`.
- A fix that requires a CI or `Jenkinsfile` edit is out of scope. Report the product defect, not a pipeline change.
