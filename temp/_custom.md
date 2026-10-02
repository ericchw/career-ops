# Custom Instructions -- career-ops

<!-- ============================================================
     THIS FILE IS YOURS. It will NEVER be auto-updated.

     Put your own house rules, custom workflows, and automations
     here -- anything you want the agent to ALWAYS do (or never do).

     This is for PROCEDURAL rules ("HOW I want things done").
     For WHO you are (archetypes, narrative, comp, negotiation),
     use modes/_profile.md instead. Keeping the two separate keeps
     each one readable.

     The agent reads this file alongside the system instructions;
     your rules here take precedence over the defaults, as long as
     they don't break the Data Contract (your files are never
     touched, and we never auto-submit an application for you).

     Because this is a user-layer file, anything you write here
     survives `node update-system.mjs`. Put customizations HERE,
     not in CLAUDE.md / modes/_shared.md / other system files --
     those get overwritten on update.
     ============================================================ -->

## House Rules

<!-- Rules the agent should always follow. Examples:
     - Always write evaluation summaries in British English.
     - Never include a photo in my CV (US / ATS-first market).
     - Cap each batch run at 20 listings unless I say otherwise.
     - If a report scores below 6, skip the cover letter. -->

- In CVs, cover letters, application answers, and outreach, bold only the most important quantified proof-point metrics, and bold those metrics every time they appear. Keep the surrounding wording unbolded. Priority metrics include **400+ embedded devices**, **$80K annually**, **300+ VMware VMs**, **33% cost reduction**, **8 global regions**, and **$550K engagement**. Do not bold routine numbers, dates, years of experience, compensation expectations, or JD figures unless emphasis is specifically useful in that document.
- **Application form answers:** Short form fields (motivation, why us, why this role, additional information, "tell us about a project") are **1-2 sentences** and straight to the point. No second paragraph for location, cert recap, or cooldown unless the question asks. Cover letters keep the existing cover-letter structure. Example (Smile #767, "What motivates you to work with us?"): "Clinical data must be available at all times. Smile runs a FHIR platform in 20+ countries, so AKS here is not a side cluster for a demo app, and I want to own the Azure that other teams ship on, with hardening and on-call that match that bar."
- **CV summary cert line:** Use full credential names in prose (e.g. "DevOps Engineer Expert"), never exam codes alone (e.g. "AZ-400"). When listing **CKS**, do not also list **CKA** in the same summary line — CKA is a prerequisite of CKS. Keep a single trailing "certified".
- **Portal scan list after evaluation:** After every completed evaluation (auto-pipeline / `oferta`), if the score meets `auto_portal_track_score_threshold` (same gate as PDF by default — see `config/profile.yml` / `node pipeline-thresholds.mjs`), add the **named end employer** career page to `portals.yml` via:

  ```bash
  node track-portal-company.mjs --report reports/{NNN}-*.md
  ```

  Confirm the company appears under `tracked_companies` with a real `careers_url` (prefer ATS board / company careers site over a single LinkedIn job URL). **Skip** when Company is `?`, Via-only agencies/staffing with no disclosed client, or the posting is closed/filled before a report is kept. If auto-track was missed earlier in the session, backfill before ending: `node track-portal-company.mjs --backfill-recent --days 1` (or `--report` per missed eval). Caught 2026-08-23 when Fleetworthy / Univerus / Octave / Capco were evaluated but not added to the scan list.
- **CV PDF template:** Always build via **`build-cv-html.mjs`** so template resolution hits `config/profile.yml` → `cv.template_path` → `config/custom-cv-template.html` (grey contact panel + burnt-orange `#D85A30`). Keep `cv.template_path: config/custom-cv-template.html` in profile. After any tailored CV regen, confirm HTML has `.contact-panel`, `--accent: #D85A30`, `Liberation Sans` — not the system teal/purple `--accent-color: hsl(187…)` look. Then `node generate-pdf.mjs` (Post-update checklist §1 for top-bleed). Quick check: `node -e "import('./build-cv-html.mjs').then(m=>import('js-yaml').then(y=>import('fs').then(fs=>console.log(m.resolveCvTemplatePath(y.load(fs.readFileSync('config/profile.yml','utf8')))))))"` must end in `config/custom-cv-template.html`.
- **CV contact icons:** After every tailored CV PDF, confirm LinkedIn/GitHub icons render (HTML must use `data:image/png;base64,…`). Files live at `config/assets/logo-linkedin.png` + `logo-github.png` — restore there if missing after an update.
- **CV bullet HTML:** When passing `opts.jobs` into `buildCvBaseHtml()`, bullets must be **plain text** (or `**markdown**`). Never embed `<strong>` / HTML tags — `inlineMarkdown()` escapes them to literal `&lt;strong&gt;` then `emphasizeMetricsInHtml()` adds real bold, so the PDF shows broken tags (caught 2026-08-24 on #744 Confluence / #747 FGF). Metrics bold via `cv.metric_emphasis` only.
- **Cover letter PDF entrypoint:** Always use **`node build-cover-letter-pdf.mjs --payload …`** (JSON payload per `modes/cover.md`). It resolves `config/custom-cover-letter-template.html` (grey band + burnt-orange accent) and merges the Verra recommendation on page 2. **Never** use `node generate-cover-letter-pdf.mjs` — that script has an inline Times New Roman builder and silently ignores the custom template (caught 2026-08-23 on Fleetworthy #732). After render, confirm page-1 HTML has `.top-band` / `#D85A30` / Liberation Sans, not Times New Roman as live CSS. See Post-update checklist §5.
- **Tailored CV PDF is mandatory at score ≥ 4.0 (verify, don't assume):** every evaluation scoring **4.0 or above** must end with a *tailored* CV PDF actually rendered, not just a path written into the report header. The `auto_pdf_score_threshold` (3.0) already permits it, so the failure mode is a silent skip, not a gate. Before closing out an eval at ≥ 4.0, confirm all three:
  1. the file exists at the `**PDF:**` path in the report header (`ls -l <path>`),
  2. a row for that report number exists in `data/pdf-index.tsv` (`generate-pdf.mjs` writes it only after a successful render — pass `--report={NNN}`),
  3. the tracker PDF column is ✅ only when 1 and 2 both hold.

  **"Tailored" means per-bullet JD keyword matching, not merely "a PDF exists"** — all three checks above can pass on an untailored base export, so they are necessary, not sufficient. Every ≥ 4.0 PDF must go through exactly one of these two routes, and you must be able to say which:

  - **Automatic ranking:** pass `jdText` + `score` to `buildCvBaseHtml()` and let `cv-jd-bullet-rank.mjs` reorder each role's bullets by JD keyword score (auto-enabled at score ≥ 4.0). Note it **reorders only, never rewrites**.
  - **Manual curation:** pass explicit `opts.jobs`. This **bypasses the automatic ranker and the bullet budget** (`build-cv-html.mjs` guards both on `!opts.jobs?.length`), so the JD matching becomes your responsibility per bullet — selection, order, and JD vocabulary injected into the wording per `modes/pdf.md` Step 14 (reformulate real `cv.md` facts, never invent). Bypassing the budget does **not** lift the caps — you now enforce `modes/_profile.md` § Page & bullet budget by hand (Verra 3-5, GTI 3-4, Macroview 1-2, max 10 total, 2 pages).

  Either way, before rendering, sanity-check the match with `node cv-jd-bullet-rank.mjs --jd-file jds/{slug}.md --score {score} --json` and confirm the top-ranked bullets are the ones that survived. A base export with default profile bullet order at ≥ 4.0 is a bug, not a shortcut.

  Never write ✅ or a `**PDF:**` path optimistically before the render succeeds. **Two legitimate exceptions**, both of which must say so explicitly in the report header instead of naming a fresh path: an explicit **reuse** decision (`reuse \`output/…pdf\` — not regenerated`, per `modes/pdf.md` reuse rules) and a user-directed hold. Batch/parallel runs are the highest-risk path — check each worker's output, not just the last one. Caught 2026-08-24: #702 CIBC (4.0, Applied) carried a `**PDF:**` path with no file and no manifest row, and #601–#692 had index rows whose files were later pruned from gitignored `output/`.
- **Dashboard URL open (WSL):** Use **wslview** (wslu) only. Do not fall back to Chromium/Chrome, xdg-open, or cmd.exe for `o` / `d` openers. Re-apply after `/career-ops update` if `dashboard/openurl.go` reverts (see Post-update checklist §4).
- **Dashboard PDF open (WSL) → Windows `file:///` URL:** The `d` hotkey (and post-generate PDF open) must pass wslview a Windows browser URL, not a `/mnt/c/...` path. Convert `/mnt/c/Users/LiamNg/Documents/GitHub/career-ops/output/cv-liam-ng-bmo-2026-09-01.pdf` to `file:///C:/Users/LiamNg/Documents/GitHub/career-ops/output/cv-liam-ng-bmo-2026-09-01.pdf` via `dashboard/internal/fileurl` (`WindowsBrowserFileURL`) before `openURLOnWSL`. Same conversion for OSC 8 PDF links in the report viewer. HTTP job URLs (`o`) stay unchanged. Re-apply after `/career-ops update` if `dashboard/openurl.go` or `dashboard/internal/ui/screens/viewer.go` reverts (see Post-update checklist §4).
- **Archive mode — prune stale Evaluated postings:** whenever running **`/career-ops archive`** or `node archive-output.mjs`, run **Step 0** by default (`archive-liveness-prune.mjs`): liveness-check every **Evaluated** row with a report `**URL:**`; **`expired` → `Discarded`** via `set-status.mjs` with note `Posting expired (archive liveness prune)`; **`uncertain`/no URL → leave**. Then archive closed-role PDFs/HTML as today. Preview with `--dry-run`; skip prune with `--no-prune`. Same API-first / Playwright-fallback ladder as `check-liveness.mjs`. User-layer files: `archive-liveness-prune.mjs`, `archive-output.mjs` hook, this rule — survive `node update-system.mjs`.
- After **`/career-ops update`**, run the **Post-update checklist** below before the next PDF regen or `node test-all.mjs`.

## Post-update checklist (local patches — system files get overwritten)

Run this after every `node update-system.mjs apply` (or when the user says they updated career-ops). `modes/_custom.md`, `config/custom-cv-template.html`, `config/custom-cover-letter-template.html`, `build-cv-html.mjs`, `build-cover-letter-html.mjs`, `build-cover-letter-pdf.mjs`, **`archive-liveness-prune.mjs`**, and **`archive-output.mjs`** are **user/local** and survive updates; **`generate-pdf.mjs`**, **`generate-cover-letter.mjs`**, **`generate-cover-letter-pdf.mjs`**, and **`test-all.mjs`** are **system layer** and may be reverted.

### 0. Archive Evaluated liveness prune (`archive-output.mjs` + `archive-liveness-prune.mjs`)

**Symptom if missing:** `/career-ops archive` moves closed-role PDFs but leaves **Evaluated** rows pointing at dead postings.

**Required:** `archive-output.mjs` calls `pruneExpiredEvaluatedRows()` from `archive-liveness-prune.mjs` before file moves unless `--no-prune`. Expired Evaluated rows → `Discarded` via `set-status.mjs --row N`.

**Verify:** `rg -n "pruneExpiredEvaluatedRows" archive-output.mjs` returns a hit; `node archive-output.mjs --dry-run` prints an "Evaluated liveness prune" section when Evaluated rows have report URLs.

### 1. Contact panel top bleed (`generate-pdf.mjs`)

**Symptom if broken:** white gap above the grey `.contact-panel` on page 1 of tailored PDFs.

**Required behavior in `injectPrintPageCss()`:** if the HTML already contains `@page` rules (from `config/custom-cv-template.html`), **return html unchanged** — do **not** inject `@page { margin: 0.6in }`, which overrides `@page :first { margin-top: 0 }`.

```javascript
if (/@page\b/i.test(html)) {
  return html;
}
```

**Pair with:** `preferCSSPageSize: true` and Playwright `margin: { top/right/bottom/left: '0' }` in `page.pdf()` (already required). Layout contract: comment block above `.header` in `config/custom-cv-template.html` and `modes/_profile.md` § CV PDF preferences.

**Verify:** regen any tailored PDF (e.g. `#424`) and confirm the grey panel touches the top edge; or grep `generate-pdf.mjs` for the `@page` early-return above. Also confirm the HTML came from the custom template (`.contact-panel` / `#D85A30`) — see House Rules → **CV PDF template**.

### 2. Skills JD filter tests (`test-all.mjs` section **7b2**)

**Symptom if missing:** `node test-all.mjs` has no `7b2. CV skills JD filter` block; skills filtering in `build-cv-html.mjs` is untested after update.

**Required:** section `// ── 7b2. CV SKILLS JD FILTER ──` immediately before `// ── 7c. UPDATER DASHBOARD REBUILD ──`, importing `buildSkillsPool`, `filterSkillsByJd`, `skillsPoolToRows` from `build-cv-html.mjs` and asserting:

1. `buildSkillsPool` merges `cv.md` skills with `profile.yml` → `tech_stack`
2. `filterSkillsByJd` keeps JD-matching categories and drops weak fits
3. `skillsPoolToRows` produces category/value rows
4. `filterSkillsByJd` leaves skills unchanged when `jdText` is empty

**Verify:** `node test-all.mjs 2>&1 | rg "7b2|skills JD"` — all four checks should pass.

**Note:** JD-filtered Skills logic lives in local `build-cv-html.mjs` (not overwritten by update); only the **tests** in `test-all.mjs` need re-application if upstream overwrote the file.

### 3. Cert JD reorder tests (`test-all.mjs` section **7b3**)

**Symptom if missing:** `node test-all.mjs` has no `7b3. CV cert JD reorder` block; cert light-tailor in `cv-jd-cert-rank.mjs` / `build-cv-html.mjs` is untested after update.

**Required:** section `// ── 7b3. CV CERT JD REORDER ──` immediately after `// ── 7b2. CV SKILLS JD FILTER ──`, importing `reorderCertsByJd`, `lightTailorCerts`, `lightTailorAdditionalSummary` from `cv-jd-cert-rank.mjs` and asserting:

1. `reorderCertsByJd` puts CKA/Terraform ahead of GCP PCA on a platform JD
2. `lightTailorCerts` caps dated rows at 6
3. `lightTailorAdditionalSummary` drops Power Platform block when JD lacks Power Platform signals
4. `buildCvBaseHtml` with `score: 4.2` + `jdText` emits reordered cert HTML (integration smoke)

**Update impact:** `cv-jd-cert-rank.mjs` is **system layer** (may be lost on update unless upstream merges). `build-cv-html.mjs` wiring is **local** per this checklist. Re-verify cert order on any regen after `node update-system.mjs apply`.

**Verify:** `node test-all.mjs 2>&1 | rg "7b3|cert JD"` — all four checks should pass.

### 4. Dashboard: WSL open URL via wslu only (`dashboard/openurl.go` + `open_linux.go`)

**Symptom if broken:** pressing `o` (open job URL) or `d` (open CV PDF) launches Linux Chromium instead of the Windows default browser, or fails with `xdg-open` exit status 4. Root cause: `dashboard/` is system layer, so `node update-system.mjs apply` can revert opener logic.

**Required behavior:**

1. `dashboard/open_linux.go` — `openWithDefaultApp` must delegate to `openURL` (not bare `xdg-open`).
2. `dashboard/openurl.go` — on WSL, `openURLOnWSL` uses **wslview only** (no cmd.exe/explorer.exe chain, no `linuxOpeners` fallback). Chromium/Chrome must not appear in `linuxOpeners`.
3. `dashboard/internal/fileurl` — `WindowsBrowserFileURL` converts WSL `/mnt/<drive>/...` and Windows `C:\...` PDF paths to `file:///C:/...` (example: `file:///C:/Users/LiamNg/Documents/GitHub/career-ops/output/cv-liam-ng-bmo-2026-09-01.pdf`). `openURLOnWSL` must call it after `normalizeURL`. Report-viewer OSC 8 PDF links in `dashboard/internal/ui/screens/viewer.go` must use the same helper.

**Paired test files (also system layer, also revertable):** `dashboard/open_command_test.go` must be `//go:build darwin` only; `dashboard/open_linux_test.go` must assert error when `PATH` is empty; `TestLinuxOpeners_excludesChromium` must pass; `dashboard/internal/fileurl/fileurl_test.go` must assert the BMO PDF example path.

**Verify:** `cd dashboard && go build ./... && go test . ./internal/fileurl -run 'TestOpenWithDefaultApp|TestLinuxOpeners|TestWindowsBrowserFileURL' -v` — grep `openURLOnWSL` for `WindowsBrowserFileURL` and confirm no `chromium` loop in `linuxOpeners`.

### 5. Cover letter template wiring (`build-cover-letter-pdf.mjs` / `generate-cover-letter.mjs`)

**Symptom if broken:** cover letter PDFs silently render with the plain system look (Times New Roman, no grey header / orange accent) instead of `config/custom-cover-letter-template.html` — no error, just the wrong look.

**Always generate via (user/local, survives updates):**

```bash
node build-cover-letter-pdf.mjs --payload /tmp/cover-payload-{slug}.json
```

That entrypoint imports `buildHtml` from **`build-cover-letter-html.mjs`**, which resolves: `cover_letter.template_path` → `config/custom-cover-letter-template.html` → `templates/cover-letter-template.html`.

**Do not use for Liam's letters:**

- `node generate-cover-letter-pdf.mjs` — system markdown→PDF path with an **inline Times New Roman `buildHtml`**; ignores the custom template and recommendation merge contract above. Caught on Fleetworthy #732 (2026-08-23).
- Prefer not relying on `generate-cover-letter.mjs` alone for final PDFs unless it still re-exports `buildHtml` from `build-cover-letter-html.mjs` (see below). Use `build-cover-letter-pdf.mjs` for page-1 custom theme + page-2 Verra recommendation merge.

**If `generate-cover-letter.mjs` is used at all:** it is **system layer** and may revert to an inline `buildHtml` after `node update-system.mjs apply`. Re-wire:

```javascript
import { buildHtml } from "./build-cover-letter-html.mjs";
```

Delete any reverted inline `buildHtml` copy in that file.

**Verify:** after any cover letter regen, page-1 HTML must include `.top-band`, `--accent: #D85A30`, and `Liberation Sans` as live CSS (Times New Roman only OK inside comments). Or: `node -e "import('./build-cover-letter-html.mjs').then(m => console.log(m.resolveCoverLetterTemplatePath({})))"` should end in `config/custom-cover-letter-template.html`.

### 6. Tailoring cross-references still resolve (`modes/pdf.md`, `build-cv-html.mjs`)

**Why:** the House Rule *Tailored CV PDF is mandatory at score ≥ 4.0* points at two **system-layer** anchors that updates overwrite. If upstream renumbers or renames them, the rule keeps reading fine but silently points at nothing.

**Check all three after each update:**

1. **`modes/pdf.md` Step 14 is still the keyword-injection step** — `rg -n "^14\." modes/pdf.md` should return the "Inject keywords naturally into existing achievements (NEVER invent)" line. If it moved, update the step number in the House Rule (and in §Manual curation) to the new one.
2. **The `opts.jobs` bypass guard still exists** — `rg -n "opts.jobs\?\.length" build-cv-html.mjs` should show it gating both the JD bullet ranking and `applyWorkExpBulletBudget()`. If upstream changes that behavior (e.g. ranking starts applying to explicit `opts.jobs`), the manual-curation branch of the House Rule is wrong and must be rewritten, not just renumbered.
3. **The ranker CLI still runs** — `node cv-jd-bullet-rank.mjs --jd-file jds/{any}.md --score 4.1` should print per-role bullet scores. It is the sanity-check the House Rule mandates before rendering any ≥ 4.0 PDF.

**Also user-layer, so it does not need re-checking but must stay in sync by hand:** the bullet caps live in `modes/_profile.md` § Page & bullet budget (Verra 3-5, GTI 3-4, Macroview 1-2, max 10, 2 pages). If you change them there, update the House Rule's manual-curation line to match.

## Custom Workflows

<!-- Multi-step routines you run often, given a short name. Examples:
     - "weekly review": scan my saved portals, evaluate the new roles,
       then give me a one-paragraph summary of the top 3.
     - "prep <company>": pull the JD, generate STAR stories from
       article-digest.md, and draft 5 likely interview questions. -->

(none yet -- add yours above)

## Output Preferences

<!-- How you like results formatted. Examples:
     - Reports: lead with the score and the one-line verdict.
     - Show the per-step token breakdown after a batch run.
     - Save PDFs date-first: YYYY-MM-DD-company.pdf -->

- **Interview-practice Q&A reference:** For the human-readable Q&A session record, use a compact format: write the interviewer question as plain text, followed immediately by the candidate's concise reference answer as a Markdown blockquote (`> ...`). Keep answers direct and technically precise. This preference applies only to the Q&A reference; retain any machine-readable transcript fields required by the interview-practice session contract.

## Off-Limits

<!-- Things the agent must never do for you. Examples:
     - Never auto-fill or submit an application without showing me first.
     - Never edit a system file to customize my setup -- put it here. -->

(none yet -- add yours above)
