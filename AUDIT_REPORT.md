# Audit Report

## 1. Executive Summary

Repository `/workspace/LCC` was audited as an imported full-stack React/Vite + Express/Firebase/Gemini application. The app installs, type-checks, builds, and serves the production bundle in this environment, but it must **not** be considered production-ready because several High findings remain open.

Finding counts:

| Severity | Count |
| --- | ---: |
| Critical | 0 |
| High | 6 |
| Medium | 8 |
| Low | 6 |

Most important results:

- `npm ci`, `npm run lint`, and `npm run build` pass on Node `v24.15.0` / npm `11.4.2`.
- Production server starts without Firebase/Gemini secrets and `/api/health` returns JSON, but most APIs are blocked when Firestore is not configured.
- CI was not reproducible because GitHub Actions used `npm ci || npm install`; this audit changed it to strict `npm ci`.
- `.env.example` had incomplete/mismatched environment coverage and a real personal admin email; this audit changed it to safer placeholders and added missing variables.
- `npm audit` reports 22 vulnerabilities, including 8 high, mainly `axios` and transitive Firebase/gRPC/storage packages.
- Backend uses Firebase token verification and many user-scoped paths, but route validation is inconsistent and `server.ts` is too large for safe long-term maintenance.
- Firestore/Storage rules are default-deny and user-scoped, but anonymous users are treated as signed-in if the frontend feature flag enables them.
- File parsing has size truncation after extraction, but ZIP/XML decompression bombs and large XLSX/PDF parser cost are not fully bounded before parsing.
- Gemini calls are backend-mediated and user key storage is encrypted with AES-GCM, but model defaults are inconsistent with docs and there is limited timeout/concurrency control around SDK calls.

## 2. Repository State

Commands executed:

```bash
git status --short --branch
git log --oneline --decorate -10
git remote -v
git branch -a
```

Observed state before changes:

- Current branch: `work`.
- Working tree: clean.
- Recent history includes `c278bea Merge pull request #1 from tranhailx92/codex/clone-va-xoa-du-lieu-cu-tren-github` and `d5c0078 Import QLCV_PQG application`.
- `git remote -v` printed no configured remote in this container, so a real GitHub PR URL cannot be generated from the CLI environment.
- Local branch list contained `work` only before the audit branch was created.
- No tracked `dist`, `node_modules`, `.env`, service-account JSON, PDF/DOCX/ZIP artifact, or log file was observed in the top-level repository listing.

Audit branch created:

```text
codex/audit-security-and-production-readiness
```

## 3. Architecture Overview

```text
Browser
  -> React 19 / Vite frontend (src/main.tsx, src/App.tsx)
  -> Express API (server.ts)
  -> Firebase Auth token verification (Firebase Admin SDK)
  -> Firestore named database / Firebase Storage
  -> Gemini API through backend-controlled routes
  -> Document processing/export libraries (PDF, DOCX, XLSX, PPTX, pdfmake)
  -> Google Drive public file/folder import through backend routes
```

Entrypoints and major modules:

| Area | Files |
| --- | --- |
| Frontend entrypoint | `index.html`, `src/main.tsx`, `src/App.tsx` |
| Backend entrypoint | `server.ts` |
| Middleware | `server/middleware/security.ts`, `server/middleware/rateLimit.ts` |
| Firebase client | `src/lib/firebase.ts` |
| Firestore rules | `firestore.rules` |
| Storage rules | `storage.rules` |
| API client / AI service | `src/services/apiClient.ts`, `src/services/geminiService.ts` |
| Editorial/publishing | `src/components/editorial/*`, `src/lib/publishing/*`, `src/lib/exportArticleModel.ts` |
| Admin UI | `src/components/admin/AdminWorkspace.tsx` |
| Legacy/proposals | `src/features/proposals/*`, `src/components/proposals/*`, proposal routes in `server.ts` |
| CI | `.github/workflows/ci.yml` |
| Operations docs | `README.md`, `DEPLOYMENT.md`, `ADMIN_RUNBOOK.md`, `SMOKE_TEST.md` |

Actual data flow observed in code:

1. Frontend initializes Firebase from `VITE_FIREBASE_*` and `VITE_FIRESTORE_DATABASE_ID`.
2. UI obtains Firebase ID tokens and sends them as `Authorization: Bearer ...` to `/api/*`.
3. Backend verifies tokens with `adminAuth.verifyIdToken()`.
4. User-scoped reads/writes use `users/{uid}/...` paths.
5. Firebase Storage is used for illustrations and chat attachments under user-prefixed paths.
6. AI calls use backend routes and resolve either server `GEMINI_API_KEY`/`GOOGLE_API_KEY` or encrypted per-user keys.
7. Document content is extracted server-side or client-side depending on route/feature, then stored as Firestore document fields or attachment excerpts.

## 4. Build and Runtime Results

Environment:

| Tool | Version |
| --- | --- |
| Node | `v24.15.0` |
| npm | `11.4.2` |

Command results:

| Command | Result | Notes |
| --- | --- | --- |
| `npm ci` | PASS | Installed 739 packages; npm reported 22 vulnerabilities and deprecation warnings. |
| `npm run lint` | PASS | `tsc --noEmit` completed in ~22 seconds. |
| `npm run build` | PASS | Vite + esbuild completed; backend bundle generated. |
| `NODE_ENV=production node dist/server.cjs` | PASS with degraded config | Server starts with missing Firebase/Gemini env and reports Firestore not configured. |
| `curl /api/health` | PASS | Returns JSON with `ok: true`, `firestoreReady: false`, `aiConfigured: false`. |
| `curl /` | PASS | Serves built `index.html`. |
| `curl /some/frontend/route` | PASS | SPA fallback serves `index.html`. |

Build outputs:

- Frontend assets emitted under `dist/assets`.
- Backend bundle: `dist/server.cjs` (~287 KB) and `dist/server.cjs.map` (~465 KB).
- Vite warned about chunks over 500 KB:
  - `pdfmake` ~1,012 KB
  - `vfs_fonts` ~855 KB
  - `firebase` ~472 KB
  - `xlsx` ~429 KB
  - app index chunks ~1,271 KB and ~405 KB

Runtime observations:

- `/api/health` is registered before static fallback and returns JSON.
- Static assets and direct frontend routes are served by production server.
- Missing Firebase credentials do not crash boot; API gateway blocks non-health `/api` routes when DB is uninitialized.
- No explicit `process.on('unhandledRejection')`, `process.on('uncaughtException')`, or graceful `SIGTERM/SIGINT` shutdown handler was found.

## 5. Critical Findings

No Critical issue was proven in the audited source. This does not mean the app is production-ready; High findings remain open.

## 6. High Findings

### [HIGH-01] Direct dependency `axios` has multiple high-severity advisories

- Severity: High
- Category: Dependency / SSRF / DoS / credential leakage
- File: `package.json`, `package-lock.json`
- Lines: `package.json` dependency list; lockfile resolved `axios@1.15.2`
- Evidence: `npm audit` reports direct `axios@1.15.2` vulnerable to multiple advisories including NO_PROXY bypass, ReDoS, resource allocation, proxy credential leakage, and prototype-pollution gadgets for ranges below `1.16.0`.
- Impact: Backend routes use axios for URL and Google Drive-related fetches. A vulnerable HTTP client increases SSRF/proxy leak/DoS risk in exactly the parts of the system that fetch user-provided or public URLs.
- Exploitation scenario: An authenticated user submits a crafted URL or operates through a proxy environment and triggers axios behavior that bypasses expected proxy/private-network assumptions or consumes excess resources.
- Recommended fix: Upgrade axios to the patched wanted version (`1.18.1` per `npm outdated`) in a dedicated dependency PR, then rerun URL/Drive smoke tests.
- Fix applied: Not applied in this PR to avoid dependency lockfile churn during audit.
- Verification: `npm audit` still reports this issue.

### [HIGH-02] CI install step allowed non-reproducible fallback to `npm install`

- Severity: High
- Category: CI/CD integrity
- File: `.github/workflows/ci.yml`
- Lines: 22-23 before fix
- Evidence: Workflow used `run: npm ci || npm install`, meaning a broken or out-of-sync lockfile could pass CI by mutating resolution behavior.
- Impact: CI could validate a dependency tree different from production/reviewer machines, hiding lockfile drift and supply-chain issues.
- Exploitation scenario: A PR changes `package.json` without lockfile updates; `npm ci` fails but CI falls back to `npm install`, making the PR appear valid.
- Recommended fix: Use strict `npm ci` only.
- Fix applied: Yes. `.github/workflows/ci.yml` now runs only `npm ci`.
- Verification: `npm ci`, `npm run lint`, and `npm run build` pass locally after the change.

### [HIGH-03] Backend and documentation disagree on Gemini model defaults

- Severity: High
- Category: AI reliability / production configuration
- File: `server.ts`, `.env.example`, `DEPLOYMENT.md`
- Lines: `server.ts` default constants around model initialization; env/docs model variables
- Evidence: Backend defaults include `gemini-3.5-flash` and `gemini-3.1-pro-preview`, while `.env.example`/deployment docs use `gemini-2.5-flash-lite`, `gemini-2.5-flash`, and `gemini-2.5-pro`.
- Impact: A deployment without explicit model env vars can select models that may be unavailable or inconsistent with docs, causing AI calls to fail at runtime.
- Exploitation scenario: Production starts successfully but AI routes fail because fallback model names are not configured to supported deployed models.
- Recommended fix: Standardize model names across code and docs in a dedicated AI config PR; keep an explicit whitelist and fail health/debug checks if configured model is unsupported.
- Fix applied: Partially. `.env.example` now includes explicit `GEMINI_FALLBACK_MODEL=gemini-2.5-flash-lite`, but backend default constants were not changed in this PR.
- Verification: Build still passes; AI runtime not tested due missing real Gemini key.

### [HIGH-04] File extraction parses complex documents before strong parser-cost limits

- Severity: High
- Category: File processing / denial of service
- File: `server.ts`
- Lines: attachment extraction route around buffer download and PDF/DOCX/XLSX parsing
- Evidence: The backend downloads a Storage object and parses PDFs with `extractPdfText`, DOCX with `mammoth.extractRawText`, and spreadsheets with `xlsx.read`; it truncates content to `maxChars` after extraction.
- Impact: A compressed XLSX/DOCX/PDF can consume CPU/memory before truncation. Storage rules cap uploads at 10 MB, but decompression/parser expansion can be much larger than stored size.
- Exploitation scenario: Authenticated user uploads a crafted XLSX with many sheets/cells or a complex PDF; extraction route consumes excessive CPU/memory and degrades service.
- Recommended fix: Add parser-level limits: max pages, max sheets, max cells/CSV output per sheet, max decompressed XML size where possible, extraction timeout/abort, and queue heavy parsing.
- Fix applied: Not applied; requires careful parser behavior tests.
- Verification: Not verified beyond source review.

### [HIGH-05] Anonymous Auth feature flag may grant full signed-in user data paths

- Severity: High
- Category: Firebase/Auth authorization posture
- File: `.env.example`, `src/config/featureFlags.ts`, `firestore.rules`
- Lines: `.env.example` anonymous flag; feature flag reads `VITE_ENABLE_ANONYMOUS_AUTH`; rules treat any `request.auth != null` as signed in.
- Evidence: `.env.example` enables anonymous auth by default. Firestore rules allow signed-in owners to create/read/update many `users/{uid}` subcollections without checking `request.auth.token.firebase.sign_in_provider`.
- Impact: If production enables anonymous auth, anonymous users can create durable user-scoped data and consume Firestore/Storage/AI-adjacent flows subject to UI/API controls.
- Exploitation scenario: A bot repeatedly obtains anonymous identities and creates tasks/documents/attachments until quotas/cost limits are hit.
- Recommended fix: For production, default anonymous auth to false and/or add server/rules-level constraints for anonymous users; require verified email/Google for cost-bearing routes.
- Fix applied: Not changed because current project instructions warn not to alter auth mode casually.
- Verification: Source review only; no live Firebase credential available.

### [HIGH-06] No graceful shutdown or global fatal-error handling found

- Severity: High
- Category: Runtime stability / DevOps
- File: `server.ts`
- Lines: server listen block near the end of `startServer()`
- Evidence: `app.listen()` starts the server, but source search did not find `process.on('SIGTERM')`, `process.on('SIGINT')`, `unhandledRejection`, or `uncaughtException` handlers.
- Impact: Render/container shutdowns may terminate active requests abruptly; unhandled promise failures can crash without structured logging/cleanup.
- Exploitation scenario: A long-running file parse or AI request is interrupted during deploy; process exits without graceful close or actionable telemetry.
- Recommended fix: Add process-level signal handlers and fatal-error logging; close HTTP server with timeout.
- Fix applied: Not applied in this PR because it touches production server lifecycle and needs runtime validation.
- Verification: Source search and production start smoke test.

## 7. Medium Findings

### [MEDIUM-01] `.env.example` missed variables used by backend and contained a real personal admin email

- Severity: Medium
- Category: Configuration hygiene
- File: `.env.example`, `server.ts`, `server/middleware/security.ts`
- Lines: env sample; backend reads `FIRESTORE_DATABASE_ID`, `FIREBASE_STORAGE_BUCKET`, `GOOGLE_API_KEY`, `GEMINI_FALLBACK_MODEL`, `ALLOWED_ORIGINS`, `ENABLE_PROPOSAL_API`, `ALLOW_CUSTOM_MODELS`
- Evidence: Source reads these variables but the sample did not fully document them; `ADMIN_EMAILS` used a real email address.
- Impact: Deployments are likely to miss required/important variables; personal email in templates increases privacy and bootstrap-admin mistakes.
- Exploitation scenario: Operator copies `.env.example` to production and accidentally grants bootstrap admin to the template address or misses CORS/database settings.
- Recommended fix: Document missing variables and use placeholders.
- Fix applied: Yes. `.env.example` now documents named Firestore DB, storage bucket, Google API fallback, fallback model, CORS origins, proposal/API model flags, and uses `admin@example.com`.
- Verification: `npm run lint` and `npm run build` pass.

### [MEDIUM-02] CORS allows all origins in non-production

- Severity: Medium
- Category: Backend security
- File: `server/middleware/security.ts`
- Lines: CORS origin callback
- Evidence: Middleware allows any origin when `NODE_ENV !== 'production'`.
- Impact: Acceptable for local development but dangerous if staging/preview deployments run with non-production `NODE_ENV` and real credentials.
- Exploitation scenario: A preview environment with real Firebase/Gemini config and `NODE_ENV=development` accepts browser requests from arbitrary sites.
- Recommended fix: Require `ALLOWED_ORIGINS` for any shared/staging environment and introduce explicit `ALLOW_DEV_CORS=true` only for local.
- Fix applied: `.env.example` now documents `ALLOWED_ORIGINS`; behavior not changed.
- Verification: Runtime health smoke shows CORS/security headers present.

### [MEDIUM-03] `/api/health` may disclose environment metadata in debug/non-production

- Severity: Medium
- Category: Information disclosure
- File: `server.ts`
- Lines: health route around debug fields
- Evidence: Health includes project/database IDs by default and adds credential source/model details outside production or when debug is enabled.
- Impact: Helpful for diagnosis, but can expose project identifiers and operational state.
- Exploitation scenario: Public staging health endpoint reveals Firebase project/database IDs and provider readiness to an attacker.
- Recommended fix: Keep public health minimal; move detailed diagnostics to an authenticated admin route.
- Fix applied: Not applied.
- Verification: `curl /api/health` returned JSON with project/database fields empty in this environment.

### [MEDIUM-04] Firestore rules allow broad proposal subcollection writes with minimal schema validation

- Severity: Medium
- Category: Firestore rules / data integrity
- File: `firestore.rules`
- Lines: proposal subcollection matches
- Evidence: Several proposal subcollections allow owner `create, update` with only ID validation and no size/type constraints.
- Impact: Users can write oversized or malformed proposal documents directly through the client if proposal UI/API is enabled.
- Exploitation scenario: A signed-in user writes large arbitrary objects to proposal subcollections, increasing Firestore cost or breaking UI assumptions.
- Recommended fix: Add field/type/size validation per proposal document type before enabling the module.
- Fix applied: Not applied.
- Verification: Source review only.

### [MEDIUM-05] Direct client storage rules rely on content type provided at upload

- Severity: Medium
- Category: Storage upload validation
- File: `storage.rules`
- Lines: chat attachment and illustration write rules
- Evidence: Rules validate `request.resource.contentType` and size, but not file signatures. This is normal for Firebase rules but insufficient against MIME spoofing.
- Impact: Malicious content can be uploaded with an allowed content type and later parsed/rendered unsafely if downstream checks trust metadata.
- Exploitation scenario: User uploads non-PDF content as `application/pdf`; extraction route attempts parser operations on unexpected bytes.
- Recommended fix: Add backend signature sniffing before parsing and store verified type/status.
- Fix applied: Not applied.
- Verification: Source review only.

### [MEDIUM-06] User-facing auth errors can include raw Firebase verification messages

- Severity: Medium
- Category: Information disclosure / auth UX
- File: `server.ts`
- Lines: `getUserIdFromRequest()` returns `Token Error: ${...}`
- Evidence: Token verification error messages are stored in request headers and returned to clients.
- Impact: Detailed backend/Firebase error strings may reveal configuration mismatch details beyond what normal users need.
- Exploitation scenario: A malformed token request obtains verbose Firebase Admin SDK error text useful for probing configuration.
- Recommended fix: Return stable error codes to clients and log detailed errors server-side only.
- Fix applied: Not applied.
- Verification: Source review only.

### [MEDIUM-07] Large frontend bundles remain despite some dynamic chunks

- Severity: Medium
- Category: Frontend performance / iPad responsiveness
- File: `vite.config.ts`, `src/App.tsx`, export/document modules
- Lines: build output and imports
- Evidence: Vite reports chunks >500 KB, including `pdfmake`, `vfs_fonts`, `firebase`, `xlsx`, and app index chunks.
- Impact: Slow initial load and memory pressure on mobile/iPad.
- Exploitation scenario: Users on lower-memory iPads hit reloads or slow UI while loading heavy document/export libraries.
- Recommended fix: Lazy-load PDF/DOCX/XLSX/PPTX/export/admin/editorial workspaces only when opened; split admin/proposal/editorial routes.
- Fix applied: Not applied.
- Verification: Build output warning.

### [MEDIUM-08] Backend is concentrated in one very large `server.ts`

- Severity: Medium
- Category: Maintainability / testability
- File: `server.ts`
- Lines: entire file, routes from `/api/health` through admin routes
- Evidence: `server.ts` contains Firebase initialization, encryption, middleware helpers, AI orchestration, Drive routes, document routes, chat routes, proposal routes, admin routes, and static serving.
- Impact: Changes are high-risk and hard to test; route-level security patterns are easy to miss.
- Exploitation scenario: A future fix adds a route without authentication or validation because there is no route module pattern.
- Recommended fix: Gradually split into `server/config`, `middleware`, `routes`, `controllers`, `services`, `schemas`, and `repositories`.
- Fix applied: Not applied.
- Verification: Source structure review.

## 8. Low Findings

### [LOW-01] `npm ci` emits deprecated package warnings

- Severity: Low
- Category: Dependency hygiene
- File: `package-lock.json`
- Lines: transitive dependencies
- Evidence: npm warns for deprecated `whatwg-encoding` and `node-domexception`.
- Impact: Not immediately exploitable but indicates dependency maintenance debt.
- Exploitation scenario: Deprecated transitive packages stop receiving fixes.
- Recommended fix: Update owning direct dependencies where safe.
- Fix applied: Not applied.
- Verification: `npm ci` output.

### [LOW-02] npm config warning for `http-proxy`

- Severity: Low
- Category: Environment hygiene
- File: npm environment, not repo source
- Lines: command output
- Evidence: npm warned `Unknown env config "http-proxy"`.
- Impact: Future npm versions may reject/ignore this environment setting differently.
- Exploitation scenario: CI/local install behavior diverges due unsupported npm config.
- Recommended fix: Clean CI/runner npm config if present.
- Fix applied: Not applicable to repo.
- Verification: `npm ci`, `npm outdated`, and `npm audit` output.

### [LOW-03] Proposal module code remains while feature flags disable it

- Severity: Low
- Category: Legacy/dead code
- File: `src/config/featureFlags.ts`, `server.ts`, `src/features/proposals/*`, `src/components/proposals/*`
- Lines: proposal feature flags and proposal API gateway block
- Evidence: Frontend flags set proposal module false; backend blocks `/api/proposals` unless `ENABLE_PROPOSAL_API=true`, but code remains.
- Impact: Increases bundle/maintenance surface; can be accidentally re-enabled without full review.
- Exploitation scenario: Operator sets `ENABLE_PROPOSAL_API=true` while Firestore rules/schema validation are incomplete.
- Recommended fix: Keep disabled until validated; if revived, add tests and schema validation first.
- Fix applied: `.env.example` now explicitly documents `ENABLE_PROPOSAL_API=false`.
- Verification: Source review.

### [LOW-04] Admin UI is hidden client-side, but users still see role based on profile state

- Severity: Low
- Category: Defense-in-depth / frontend authorization
- File: `src/App.tsx`, `src/components/admin/AdminWorkspace.tsx`
- Lines: admin tab/profile gating
- Evidence: UI gates admin with `profile.role === 'admin'`; backend has `requireAdmin` for actual admin APIs.
- Impact: Backend enforcement is the key control; client state should not be relied on for security.
- Exploitation scenario: A user manipulates client state to show admin UI, but backend should deny API calls.
- Recommended fix: Keep backend `requireAdmin`; add integration tests confirming non-admin 403.
- Fix applied: Not applied.
- Verification: Source review; no live Firebase token available.

### [LOW-05] Static source scan found no obvious committed private key, but only near history was checked

- Severity: Low
- Category: Secret scanning limitation
- File: repository history
- Lines: git history commands
- Evidence: `git grep` found placeholders and code references; near-history stat did not show `.env` or service account files. This was not a full entropy scanner over all blobs.
- Impact: A secret could still exist in unreachable/full history or encoded artifacts.
- Exploitation scenario: A previously committed secret remains in GitHub history outside the checked range.
- Recommended fix: Run GitHub secret scanning/gitleaks/trufflehog in CI and rotate any exposed key.
- Fix applied: Not applied.
- Verification: Manual grep/log commands.

### [LOW-06] Health endpoint returns `ok: true` even when Firestore and AI are not configured

- Severity: Low
- Category: Observability
- File: `server.ts`
- Lines: health route response
- Evidence: In this environment `/api/health` returned `ok: true` with `firestoreReady: false` and `aiConfigured: false`.
- Impact: Render startup checks pass even when dependencies are degraded; good for boot but can hide readiness issues if operators check only `ok`.
- Exploitation scenario: Deployment is marked healthy while critical features fail due missing env.
- Recommended fix: Keep `/api/health` lightweight but add `/api/ready` or admin diagnostics for dependency readiness.
- Fix applied: Not applied.
- Verification: Runtime smoke test.

## 9. Firebase Review

Client initialization:

- `src/lib/firebase.ts` reads `VITE_FIREBASE_API_KEY`, `VITE_FIREBASE_AUTH_DOMAIN`, `VITE_FIREBASE_PROJECT_ID`, `VITE_FIREBASE_STORAGE_BUCKET`, `VITE_FIREBASE_MESSAGING_SENDER_ID`, `VITE_FIREBASE_APP_ID`, and `VITE_FIRESTORE_DATABASE_ID`.
- Named Firestore DB is used when `VITE_FIRESTORE_DATABASE_ID` is set and not `(default)`.
- Missing config degrades frontend initialization instead of throwing.

Admin initialization:

- Backend reads service account JSON/base64, validates project ID where possible, falls back to application default credentials, and records `firestoreReady` after verification.
- Backend uses named DB from `VITE_FIRESTORE_DATABASE_ID` or `FIRESTORE_DATABASE_ID`.

Rules posture:

- Firestore default denies all unmatched documents.
- User data is mostly scoped under `users/{userId}` with `request.auth.uid == userId` or admin.
- `settings/aiKey` blocks direct client reads/writes; backend routes mediate status/test/save/delete.
- Storage rules default deny unmatched paths and scope `illustrations/{userId}/...` and `chatAttachments/{userId}/...` to owner/admin with size/content-type checks.

Answers to required Firebase questions:

1. User A reading/writing User B: Firestore/Storage rules generally prevent this through `isOwner(userId)` checks. Backend routes also use token UID for user paths. No proven IDOR was found.
2. Anonymous user overreach: Possible depending on production feature flags. Rules treat anonymous auth as signed-in.
3. Admin determined only by client email: Backend `requireAdmin` checks token claims and bootstrap `ADMIN_EMAILS`; UI profile role is not the backend control.
4. Backend admin check: Yes, admin routes use `requireAdmin`.
5. Storage path bound to UID: Yes for allowed `illustrations/{userId}` and `chatAttachments/{userId}` paths.
6. Upload size/content type: Yes in Storage rules; no file signature validation in rules.
7. Firestore 1 MiB risk: Some rules cap version content, but many document paths lack comprehensive field-size limits; attachment extraction truncates content to 100k chars.
8. Large sessions subcollection: Session versions exist under `users/{userId}/sessions/{sessionId}/versions/{versionId}`; this helps split history.
9. Delete document also deletes Storage object: Backend document delete attempts Storage delete if `metadata.storagePath` exists, then deletes Firestore document even if Storage delete fails.
10. Orphan risk: Yes, if Storage delete fails or client deletes direct Firestore documents where backend cleanup is not involved.

## 10. Gemini/AI Review

Strengths:

- Gemini API key is not embedded in frontend source; frontend calls backend APIs.
- Personal user API keys are stored under blocked Firestore path and encrypted server-side using AES-256-GCM when `AI_KEY_ENCRYPTION_SECRET` is present.
- AI routes use Firebase bearer tokens for most cost-bearing operations.
- Error classification distinguishes quota, invalid key, model unavailable, safety block, validation, and timeout-like failures.
- Model whitelist exists unless `ALLOW_CUSTOM_MODELS=true`.

Risks:

- Backend default model constants differ from docs/sample env.
- AI SDK calls have limited consistent timeout/cancellation/concurrency control.
- Prompt-injection is structurally possible because user documents are fed into prompts; route-specific prompt hardening varies.
- Logging avoids raw API keys, but route logs and errors should continue to be reviewed to ensure prompts/responses are not stored wholesale.
- `GOOGLE_API_KEY` fallback as a Gemini key can confuse Drive vs Gemini credential ownership.

## 11. Frontend Review

Strengths:

- Firebase tokens are sent as `Authorization: Bearer` for backend calls.
- Admin UI is gated by profile role and backend admin APIs are separately protected.
- ErrorBoundary exists.
- Heavy export libraries are partly chunked by Vite.
- Blob URL revoke patterns exist in some export panels.

Risks:

- `localStorage` stores documents/sessions/chat drafts; do not store AI keys or sensitive full-text data there. Source scan did not find raw API key storage in localStorage, but drafts/document caches can contain user content.
- Large app chunks affect mobile/iPad load.
- React app is centralized in a very large `src/App.tsx`, increasing hook/race-condition review difficulty.
- Admin/proposal/editorial modules should be route-level lazy-loaded.

Recommended code splitting:

- Lazy-load `AdminWorkspace` only for admin tab.
- Lazy-load `EditorWorkspace`, `SlideOutlineGenerator`, and proposal components.
- Dynamic import `xlsx`, `pdfmake`, `vfs_fonts`, `docx`, `pptxgenjs`, `jspdf`, and `html2canvas` only inside export/parse actions.
- Keep Firebase auth core in initial bundle but avoid loading admin/editorial/export stacks before needed.

## 12. Backend Review

Strengths:

- Helmet is configured.
- CORS middleware exists and blocks unspecified origins in production when `ALLOWED_ORIGINS` is missing.
- Rate limiting applies in production to `/api`, AI routes, task builder, and editorial image planning, with `/api/health` skipped.
- JSON body limit is `2mb`.
- Many routes use token UID rather than accepting user ID from request body.
- SSRF mitigation exists for `/api/fetch-link`: DNS lookup, private IP blocking, manual redirects, timeout, and content-length cap.

Risks:

- Validation is inconsistent: many routes manually parse bodies instead of shared Zod schemas.
- Error handler can return raw error messages from initialization/auth paths.
- No global graceful shutdown/fatal error handlers.
- Backend route and business logic are mixed in one file.
- File parsing and AI calls are synchronous within request handlers in several paths.

## 13. File Processing Review

Observed flows:

- Uploads to Storage are user-scoped by Storage rules.
- Chat attachment extraction downloads Storage object server-side, parses based on MIME/extension, truncates extracted text to 100,000 chars, and stores excerpt/status.
- Drive import/folder sync routes use Google Drive API key and store extracted metadata/content.
- Link fetch route has SSRF protections.
- Exports use frontend libraries for DOCX/PDF/PPTX and backend route for slide HTML export.

Risks by type:

| Type | Risk |
| --- | --- |
| PDF | many pages/complex structure can consume parser resources; scanned PDFs return placeholder text. |
| DOCX | zipped XML decompression expansion before truncation. |
| XLSX/XLS | many sheets/cells can expand massively through `sheet_to_csv`. |
| PPTX | export risk mostly bundle/performance; parsing should be reviewed if expanded. |
| Images | Storage rules cap 5 MB for illustrations and content type jpeg/png/webp. |
| HTML/URL | `/api/fetch-link` has SSRF controls; HTML sanitization/rendering must be route-specific reviewed. |
| Google Drive | public-link import depends on API key and Drive metadata/content handling; quota/rate limiting should be monitored. |

## 14. Dependency Review

Commands:

```bash
npm outdated
npm audit
npm ls --depth=0
npm dedupe --dry-run
```

Summary:

- `npm audit`: 22 vulnerabilities total — 0 critical, 8 high, 11 moderate, 3 low.
- Direct high-priority package: `axios@1.15.2`, patched wanted `1.18.1`.
- Transitive Firebase/Admin stack includes vulnerable `@grpc/grpc-js` and `google-gax` paths; some fixes require `firebase-admin@14.1.0` (major).
- `npm dedupe --dry-run` indicates non-breaking tree cleanup can update several transitive packages (including `@grpc/grpc-js`) without committing changes yet.

Outdated direct packages include:

| Package | Current | Wanted | Latest | Recommendation |
| --- | ---: | ---: | ---: | --- |
| axios | 1.15.2 | 1.18.1 | 1.18.1 | P1 security upgrade. |
| vite | 6.4.2 | 6.4.3 | 8.1.3 | Patch/minor first; major later. |
| firebase-admin | 13.8.0 | 13.10.0 | 14.1.0 | Patch first; evaluate major for audit fixes. |
| firebase | 12.12.1 | 12.15.0 | 12.15.0 | Patch/minor with Firebase smoke tests. |
| express | 4.22.1 | 4.22.2 | 5.2.1 | Patch only now; Express 5 separate PR. |
| xlsx | 0.18.5 | 0.18.5 | 0.18.5 | No newer direct version in npm outdated; consider alternatives/mitigations. |
| pdfmake | 0.3.9 | 0.3.11 | 0.3.11 | Patch after export tests. |
| pdfjs-dist | 5.7.284 | 5.7.284 | 6.1.200 | Major later with PDF tests. |

## 15. CI/CD Review

Current workflow:

- Triggers on push/PR to `main`/`master`.
- Uses Ubuntu latest and Node `20.x`.
- Uses npm cache.
- Runs install, lint, build.

Fix applied:

- Replaced `npm ci || npm install` with strict `npm ci`.

Remaining gaps:

- No top-level `permissions: contents: read` hardening.
- No concurrency cancellation.
- No timeout-minutes.
- No test step because no test script exists.
- No controlled `npm audit`/dependency review step.
- No artifact upload.
- No CodeQL/secret scanning/Dependabot config in repo.
- No branch protection can be verified from local clone.

Recommended minimal CI:

1. `npm ci`.
2. `npm run lint`.
3. `npm run build`.
4. Add unit/smoke tests when present.
5. Add controlled audit gate (`npm audit --audit-level=high`) once current vulnerabilities are triaged.
6. Upload build artifact for deploy debugging.
7. Avoid deployment on untrusted PRs.

## 16. Maintainability Review

Primary maintainability issue: `server.ts` is doing too much.

Recommended target structure:

```text
server/
  config/
  middleware/
  routes/
  controllers/
  services/
  repositories/
  schemas/
  utils/
  types/
```

Migration plan:

- First extract pure config parsing and validation.
- Then extract auth/admin middleware.
- Then split route groups: health, AI, Drive, documents, chat, user settings, admin.
- Add Zod schemas per route before moving business logic.
- Keep behavior identical in each extraction PR.

## 17. Dead Code and Legacy Modules

Evidence of transitional/legacy code:

- Proposal feature flags are false in frontend.
- Backend blocks proposal API unless `ENABLE_PROPOSAL_API=true`.
- Proposal components/services/routes remain in source.
- `scripts/refine-ui.ts` and `scripts/migrate-sessions.ts` are operational scripts, not necessarily dead, but need runbook ownership.

No deletion was performed because the user explicitly requested audit-first and the project instructions warn against deleting code without strong evidence.

## 18. Smoke Test Results

| Smoke test | Result | Notes |
| --- | --- | --- |
| Login page opens | Blocked | Browser/Firebase credentials not available in this environment. |
| Anonymous login flag | Source-reviewed | `VITE_ENABLE_ANONYMOUS_AUTH` controls UI behavior; live Auth not tested. |
| Google login flag | Source-reviewed | `VITE_ENABLE_GOOGLE_AUTH` controls UI behavior; live Auth not tested. |
| Create/edit/delete task | Blocked | Requires Firebase credentials/auth. |
| Create editorial session/save content | Blocked | Requires frontend browser + Firebase. |
| Upload image/PDF/DOCX/XLSX | Blocked | Requires Firebase Storage/Auth. |
| Gemini call | Blocked | Requires Gemini key and authenticated user for most routes. |
| Gemini invalid JSON handling | Source-reviewed | JSON extraction/classification exists; no live model test. |
| Export DOCX/PDF/PPTX | Build-reviewed | Libraries build; browser export not run. |
| Non-admin blocked from admin | Source-reviewed | Backend `requireAdmin` exists; no live token test. |
| Admin can access admin | Blocked | Requires custom claims/bootstrap env + auth. |
| User A cannot read User B | Source-reviewed | Rules and backend paths are user-scoped; emulator/live test not run. |
| Large file rejected | Source-reviewed | Storage size limits exist; parser-cost limits incomplete. |
| Rate limit response | Source-reviewed | Rate limiter skips non-production; production live test not run. |
| Production build runs | PASS | `NODE_ENV=production node dist/server.cjs`, `/api/health`, `/`, and SPA fallback tested. |

## 19. Recommended Roadmap

### P0 — xử lý ngay trước khi deploy

- Resolve all High dependency vulnerabilities that affect request fetching and Firebase/gRPC paths.
- Decide production anonymous-auth policy; disable or restrict anonymous cost-bearing flows.
- Add parser-cost limits/timeouts before enabling document ingestion for untrusted users.
- Standardize Gemini model defaults and verify live keys in staging.
- Add graceful shutdown/fatal-error handling.

### P1 — xử lý trước khi sử dụng nội bộ chính thức

- Add route schemas with Zod for AI/document/Drive/user-key/admin routes.
- Add authenticated readiness/diagnostics endpoint separate from public health.
- Add backend signature sniffing for uploaded files.
- Add Firestore rule field-size validation for proposal and document subcollections.
- Add controlled `npm audit`/dependency review CI after upgrading vulnerable packages.
- Add integration tests for admin/non-admin and User A/User B isolation using Firebase Emulator.

### P2 — cải thiện trong sprint tiếp theo

- Split `server.ts` into route/service modules.
- Lazy-load admin/editorial/proposal/export components.
- Add unit tests for AI JSON repair/validation, URL safety, and file metadata validation.
- Add observability: request IDs, structured logs, AI usage metrics without prompts/secrets.
- Document exact Render env and smoke-test checklist.

### P3 — tối ưu dài hạn

- Move heavy file parsing to background jobs/queue.
- Add malware scanning or external file safety service if uploads become public-facing.
- Add monitoring/alerts for Firestore/Storage/Gemini quota.
- Add caching for Drive metadata and expensive AI analysis where safe.
- Consider replacing `xlsx` with safer bounded parsing strategy for hostile spreadsheets.

## 20. Changes Applied

- `.github/workflows/ci.yml`: changed install step from `npm ci || npm install` to `npm ci`.
- `.env.example`: documented missing env vars, set named Firestore DB sample, added storage bucket and CORS/proposal/model flags, clarified `GOOGLE_API_KEY` fallback, and replaced real admin email with `admin@example.com`.
- `AUDIT_REPORT.md`: added this audit report.

No code behavior, Firebase project ID, Firestore database ID, auth provider behavior, package versions, or lockfile entries were changed.

## 21. Commands Executed

```bash
pwd
find .. -name AGENTS.md -print
git status --short --branch
git log --oneline --decorate -10
git remote -v
git branch -a
git checkout -b codex/audit-security-and-production-readiness
find . -maxdepth 2 -type f | sed 's#^./##' | sort | head -200
node -e "const p=require('./package.json'); console.log(p.scripts); console.log(Object.keys(p.dependencies||{}).length, Object.keys(p.devDependencies||{}).length)"
node --version
npm --version
npm ci
npm run lint
npm run build
find . -maxdepth 3 -type f -not -path './.git/*' -not -path './node_modules/*' | sort | sed 's#^./##' | head -300
rg -n "process\\.env|import\\.meta\\.env|VITE_|GEMINI|GOOGLE|FIREBASE|ADMIN|CORS|PORT|AI_KEY" -S --glob '!node_modules' --glob '!dist'
rg -n "app\\.(get|post|put|patch|delete|use)|router\\.(get|post|put|patch|delete)|multer|cors|helmet|rateLimit|express\\.json|verifyIdToken" server.ts src --glob '!node_modules'
nl -ba server.ts | sed -n '...'
nl -ba server/middleware/security.ts
nl -ba server/middleware/rateLimit.ts
nl -ba src/lib/firebase.ts
nl -ba src/config/featureFlags.ts
nl -ba firestore.rules
nl -ba storage.rules
nl -ba .github/workflows/ci.yml
npm outdated
npm audit --json
npm ls --depth=0
npm dedupe --dry-run
PORT=3131 NODE_ENV=production node dist/server.cjs
curl -sS -i http://127.0.0.1:3131/api/health
curl -sS -i http://127.0.0.1:3131/
curl -sS -i http://127.0.0.1:3131/some/frontend/route
git grep -n -I -E "AIza|BEGIN PRIVATE KEY|api[_-]?key|secret|password|token"
git log --all --stat --oneline -- .env "*.json" "*.key" "*.pem"
```

## 22. Remaining Unknowns

- Live Firebase Auth/Firestore/Storage behavior was not tested because no real credentials or emulator workflow was provided.
- Gemini and Google Drive API behavior was not tested with real keys.
- GitHub branch protection, secret scanning, Dependabot, CodeQL, and PR #1 metadata cannot be fully verified from this local clone because no remote is configured.
- Browser UI and iPad responsiveness were not visually tested in this CLI environment.
- No full Git history entropy scan was run; only source grep and near-history/stat checks were performed.
- No package upgrades were applied, so dependency vulnerabilities remain open for follow-up PRs.
