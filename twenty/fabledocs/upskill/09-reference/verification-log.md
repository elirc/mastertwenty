# Verification Log

Authored: 2026-07-11. Method: static reading of a fresh clone (no git history — `git log` reports no commits on `main`). **No dependencies were installed; no app, tests, or DB were run.** Every command in these docs is therefore marked *inferred*. Every line-anchored claim was made with the file open at authoring time.

## Files read in full or in large part (line anchors derived from these)

- `packages/twenty-server/src/main.ts` (all)
- `engine/api/graphql/workspace-schema.factory.ts` (all)
- `engine/api/graphql/workspace-resolver-builder/factories/create-one-resolver.factory.ts` (all)
- `engine/api/common/common-query-runners/common-base-query-runner.service.ts` (all, L1–L432)
- `engine/api/common/common-query-runners/common-create-one-query-runner.service.ts` (all)
- `engine/metadata-modules/webhook/jobs/call-webhook.job.ts` (all)
- `engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts` (all)
- `engine/guards/jwt-auth.guard.ts` (L15–L55), `engine/guards/workspace-auth.guard.ts` (all)
- `modules/workflow/workflow-runner/jobs/run-workflow.job.ts` (all)
- `engine/twenty-orm/repository/workspace-select-query-builder.ts` (L330–L390)
- `engine/twenty-orm/repository/workspace-insert-query-builder.ts` (L180–L280)
- `engine/workspace-event-emitter/workspace-event-emitter.ts` (L39–L67)
- `engine/core-modules/message-queue/drivers/bullmq.driver.ts` (grep-verified L248: `attempts: 1 + (options?.retryLimit || 0)`)
- `twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts` (all)
- `twenty-front/src/modules/object-record/record-field/ui/hooks/usePersistField.ts` (all)
- `twenty-front/src/modules/auth/states/currentWorkspaceMemberState.ts` (all)
- Root `package.json`, `CLAUDE.md` (all)

## Directory inventories confirmed (structure claims)

`packages/` list; `twenty-server/src/engine/{api,core-modules,metadata-modules,twenty-orm,guards,subscriptions,workspace-manager}`; `twenty-server/src/modules/{messaging,workflow,…}`; `twenty-front/src/modules` list; `test/integration/graphql/suites` file list; `.github/workflows` list; existence of `.env.example` (server + front), `twenty-e2e-testing/playwright.config.ts`, `field-metadata.service.ts`, REST core handlers, message-queue drivers, permissions module files, `secure-http-client` (via import), record-position/dataloaders/subscriptions dirs.

## Key confirmed behaviors (from code, not docs)

1. Per-workspace GraphQL schema computed from cached SDL + factory resolvers (workspace-schema.factory.ts).
2. Pipeline order: throttle (API-key only) → validate → system-object settings permission → parse/complexity → hooks → workspace-context execution (common-base-query-runner.service.ts#L101-L159).
3. Permission validation + row-level predicates inside ORM query builders; `executeExistsQuery` deliberately blocked.
4. Insert path re-selects with permission bypass for event payloads, then emits CREATED + UPSERTED in-process (EventEmitter2).
5. Webhook job: HMAC-SHA256 over `timestamp:payload`, nonce header, SSRF-guarded client, 5s timeout, catch-all without rethrow (→ effective single-attempt delivery); fan-out chunk=20, enqueue retryLimit=3.
6. Workflow job: system auth context, status guards on start/resume, error → FAILED + rethrow; staled-runs cleanup command exists.
7. Front create: client-side uuid v4, optimistic cache insert, rollback on error, aggregate refetch post-create.

## Uncertainties / not covered (labeled in-text as *investigate*)

- Transaction scope around ORM writes and workspace migrations (rollback-vs-emit windows; migration atomicity) — **not verified**.
- Locking/queue semantics that would make the workflow-start race real or impossible.
- SecureHttpClientService internals (DNS-rebinding handling).
- Browser/user-token rate limiting outside the common pipeline (captcha/session layers).
- Whether spreadsheet import uses single-create or batch hooks.
- Runtime behavior of all commands (nothing executed); exact CI job contents (workflow files listed, not parsed).
- Large subsystems read only structurally: messaging/calendar sync, billing, AI modules, front views/page-layout internals, MCP/OpenAPI surfaces, workspace-migration builder detail.

## 2026-10-06 re-check

Static re-check against `elirc/mastertwenty` (snapshot commit `19dbb5af`, Twenty under `twenty/`). Nothing was installed or run.

- All relative links in `fabledocs/`, `fabledocs2/`, `fabledocs3/` and `fabledocs4/` resolve, and every `#Lx-Ly` anchor is within its file.
- The webhook seam (behavior 5 above, ADR-010) still matches the code. In `call-webhook.job.ts`, the catch at L102–L116 records an audit event and does not rethrow. `call-webhook-jobs.job.ts:66-73` chunks by 20 and enqueues with `retryLimit: 3`. `bullmq.driver.ts:248` sets `attempts: 1 + retryLimit`. `workspace-event-emitter.ts:70` emits in-process.
- Not covered in these docs: `call-webhook-jobs-for-metadata.job.ts`, a second fan-out for metadata events. It enqueues the same `CallWebhookJob` (L63–L68), so it has the same single-attempt delivery.

## Line-number drift warning

Anchors are exact as of authoring. The upstream project moves fast; if an anchor misses, search the quoted symbol name in the same file/directory.
