# Huawei_MDF_Assistant
A Manifest V3 extension for Chrome and Edge that drives Huawei Cloud MDF forms from a structured JSON payload. TypeScript, Vite, a Tailwind side panel, a persisted service-worker coordinator, and an isolated-world content script. No backend or API key is needed.
# Huawei MDF Assistant

A Manifest V3 extension for Chrome and Edge that drives Huawei Cloud MDF forms from a structured JSON payload. TypeScript, Vite, a Tailwind side panel, a persisted service-worker coordinator, and an isolated-world content script. No backend or API key is needed.

**Release status:** runnable implementation with automated fixture coverage. The authenticated Huawei portal DOM has **not** been inspected or tested in a live partner account. The selector map contains candidate mappings, not certified Huawei selectors. Calibrate it in your account before enabling production submissions. The test suite never submits a real MDF application.

## Build and install

Use Node.js 22.12+ (or a supported newer Node release).

```sh
npm install
npm run build
```

For reproducible installs after the initial setup, use `npm ci`. The committed lockfile pins the tested dependency tree.

1. Open `chrome://extensions` in Chrome or `edge://extensions` in Edge.
2. Enable **Developer mode**.
3. Choose **Load unpacked**, then select this project's **dist/** directory. Do not select the source directory.
4. Sign in yourself and open the Huawei Cloud **Partner Center**.
5. Click the extension toolbar icon to open its side panel. If the browser's sidebar integration is unavailable, open the extension's **Options** page; it uses the same interface in a tab.
6. Import a JSON payload or edit the example. Select the portal tab explicitly.
7. Run **Inspect only** first. For a filling mode, read the payment agreement in the portal and set `agreeToTerms` to `true` only if you accept it.
8. Use **Fill only** to review actual field mappings. Enable **Fill and submit** only after reviewing the payload and checking the explicit submission authorization.

The root `manifest.json` is a build input. The build emits a module service worker, a classic self-contained content script, and local UI assets. There is no remote code, `eval`, or inline executable extension-page script. Minimum Chrome version is 116; use a current Edge version with the MV3 Side Panel API. Edge-specific behavior still needs a manual smoke test.

## Modes

| Mode | Behavior |
| --- | --- |
| `DRY_RUN` | Validates data and inspects currently mounted controls. No clicks, field writes, uploads, agreement changes, or navigation. Later wizard pages are explicitly logged as unverified. |
| `FILL_ONLY` | Fills fields, uploads files, checks the consented agreement, and advances through form pages. Stops before the final save/submit action. The portal itself may autosave as you edit. |
| `DRAFT` | Fills and clicks Save Draft, then waits for a saved confirmation. Fails visibly if Save Draft is unavailable. |
| `SUBMIT` | Fills, clicks Submit, handles a submission confirmation dialog, and waits for success. Requires explicit authorization in the panel. |

Pause and Resume are available in the panel and the on-page overlay. **Esc** pauses/resumes; **Shift+Esc** aborts. Abort stops subsequent actions; it cannot undo uploads, portal autosaves, or a request already sent. One run is allowed at a time and is bound to one tab.

## Workflow and the supplied manual

The [Huawei MDF manual](https://support.huaweicloud.com/intl/en-us/usermanual-bpconsole/apb_060001.html) documents **Benefits → Market Development Fund (MDF) → MDF Quota → Submit Application**, followed by an activity-plan page and agreement, an objectives page, a budget page, and a final confirmation. This implementation uses that order for `layout: "wizard"`.

The brief's alternate labels, such as Marketing and Apply for MDF, are included in the selector dictionary. `layout: "single"` follows the brief's plan → budget → objectives ordering without Next clicks. The manual does not establish the presence of Save Draft, the illustrative payload categories, all proposed fields, or a readable application ID. Those require account-specific verification.

Huawei also documents a 10-working-day application lead time and quota constraints. These remain portal/business checks. The extension does not guess local holiday calendars, convert currencies, infer available MDF quota, or override validation. All amounts must use the currency displayed by your account.

## Payload

See `src/types/mdf-payload.ts` for interfaces and `examples/mdf-application.json` for an inspectable example. The example keeps consent false intentionally. Its dates, audience, category, and location labels are illustrative and must be replaced as appropriate.

- `layout`: `wizard` or `single`.
- Optional `partner`: verifies the displayed account or selects it if it is editable. Omit only when the signed-in account already identifies the intended organization.
- `activity`: name, type, objective, audience array, real ISO start/end dates, country, city, and optional late-submission reason.
- `budgetItems`: 1–50 rows with category, description, total cost, requested MDF, and optional percentage ratio (0–100). If supplied, the ratio must agree with the requested amount to two-decimal rounding; final budget checks detect frontend recalculations that change your values.
- `kpis`: integer leads/opportunities, estimated revenue, and agenda.
- `attachments`: up to 10 files. Each file has a base64 body, name, MIME type, `target` (`proposal` or `quotation`), and `section` (`plan`, `objectives`, or `budget`).
- `agreeToTerms`: explicit boolean consent; all filling modes require `true`.

The panel can import/export JSON and attach local files. It converts selected files to raw base64 (without the `data:` prefix). Limit: **2 MiB per file, 4 MiB total** to remain within local storage limits. JSON import is limited to 7 MiB. Do not put credentials or access tokens in a payload. Editor changes are not autosaved; export JSON if you want to retain an unstarted template. A started run stores its own immutable payload snapshot, which is restored in the editor when the panel reopens. Edits apply only to the next run; Resume keeps the saved payload.

## Recovery model

1. The service worker serializes state transitions and persists a run in `chrome.storage.local` before returning each claimed action.
2. The claim includes a unique token and the browser's document ID. Acknowledgements from stale documents, other tabs, subframes, and cancelled claims are rejected.
3. The content script executes one step, verifies its observable outcome, and acknowledges it before claiming the next.
4. On a new document, the current form section is replayed using idempotent operations: field assignment, row-count reconciliation, checked-state checks, selected-option checks, and uploaded-filename checks. A Next action first checks whether its destination has already mounted.
5. The extension assumes the portal preserves earlier wizard sections across navigation. If a reload resets the wizard to an earlier page, required current-section controls will be missing and the run stops. It does not silently claim those values survived. Return to the saved section or abort and restart from the plan after reviewing the portal.
6. Before any final click, its intent is persisted. If a Submit, Save Draft, or Confirm action is interrupted or loses its acknowledgement, the run enters **attention** and is not automatically replayed. **Check result** only reads a visible confirmation in the bound tab. It never resubmits. If no confirmation is visible, inspect Application History manually before clearing the run.

These are conservative duplicate-prevention measures, not a guarantee of server-side exactly-once delivery. The portal offers no idempotency key to this DOM client. A success message without a visible reference is reported as successful with an unavailable reference; no ID is fabricated. Arbitrary table rows are not scraped as proof of a new application.

A closed portal tab pauses the run. Reopen the portal and review any saved/submitted work before clearing and starting a new run. The extension does not silently move a run to another tab or bypass authentication. A service-worker restart reloads state from storage; request counters are kept in session storage across worker suspension.

## Selector calibration

All Huawei-specific selectors, widget classes, labels, and confirmation patterns are maintained in `src/config/selectors.config.ts`.

1. Open an unsubmitted test application in your signed-in account.
2. Use DevTools to inspect stable `data-testid`, `formcontrolname`, `name`, accessible names, or associated label IDs.
3. Update the relevant field's `css` array. Put the most precise known selector first. Avoid hashed styling classes or translated text fragments that also match other controls.
4. Calibrate both the `budgetTable` container and row-scoped field selectors. The Add button must be scoped within that container. Preflight rejects extra expense rows.
5. For portal dropdowns, map overlay/option classes and confirm that selected state is visible through ARIA, a selected label, or a rendered tag. Do not substitute a click-only implementation that assumes success.
6. For uploads, map the hidden file input, its container, upload progress/error elements, and the exact filename badge. Setting `input.files` alone is not considered success.
7. Verify both wizard Next transitions and the exact final dialog. Keep Submit scoped away from the application-list creation button.
8. Rebuild, click Reload on the browser's extensions page, reload the portal tab, and run Inspect then Fill only.

Ambiguous matches are errors rather than arbitrary first-match clicks. Missing required fields stop the workflow; optional payload fields are skipped only if omitted from the payload. To support a materially different account schema, update its payload/step mapping as well as selectors.

### Supported and bounded adapters

- Native setters and bubbling input/change events support ordinary controlled inputs and textareas. Contenteditable fields receive text, not HTML.
- Native selects and visible custom option lists are supported, including body-mounted popovers; `aria-controls` is preferred.
- Date fields support native dates and editable text inputs using the configured date format (ISO by default). A read-only calendar, a virtualized option list, iframe-hosted form, or closed shadow-root widget requires a dedicated adapter; the extension stops rather than claiming it was filled.
- Browser-generated events have `isTrusted === false`. Applications requiring trusted events or blocking programmatic file assignment cannot be bypassed by this extension.
- Network idle tracks fetch/XHR requests observable under the Huawei host permission, plus busy/progress indicators. Requests to unrelated CDN/service domains and requests that predate extension installation are not observable. Add only verified, necessary hosts if your account needs them. Continuously polling pages may time out; refine request filtering after inspecting the portal rather than globally disabling waits.
- The default element timeout is 15 seconds; uploads use 60 seconds and confirmation capture uses 30 seconds. Action jitter (150–400 ms) follows readiness checks and is not used as the primary synchronization method.

## Privacy and permissions

No analytics, server calls, credential collection, or external uploads are performed by the extension itself. Payload files are uploaded through the portal's own file controls only during an authorized filling run. Data is stored unencrypted in your local browser profile until you clear it or uninstall the extension. The editor and downloaded JSON may include attachment contents. Local execution logs may include file names or option labels in diagnostic messages.

| Permission | Purpose |
| --- | --- |
| `storage` | Durable run checkpoints and bounded logs; local storage is restricted to trusted extension contexts. Content scripts access it through a validated message bridge. |
| `activeTab`, `scripting` | Connect to a user-selected portal tab, including a tab opened before installation. |
| `sidePanel` | Persistent application editor and live log. The options-page fallback also works in a browser tab. |
| `webRequest` | Observe request start/end for network readiness. No request headers, credentials, bodies, or URLs are stored. |
| `https://*.huaweicloud.com/*` | HTTPS portal automation and network observation. Documentation/login pages are excluded from automation. |

The extension injects no main-world network hooks and exposes no web-accessible resources. It does not intercept passwords. Clear local run removes the cached payload, file bytes, checkpoints, and logs; transient request IDs expire with the browser session or tab closure.

## Development and verification

```sh
npm run typecheck
npm test
npm run build
npx playwright install chromium
npm run test:e2e
```

Unit tests exercise validation, domain checks, controlled input dispatch, exact/ambiguous labels, asynchronous mounting, cancellation, native selection, network idle, and recovery rules. Browser tests load the built MV3 extension in a temporary Chromium profile against an intercepted Huawei-shaped fixture. They cover read-only inspection, custom dropdowns, cascading values, uploads, totals, multiple budget rows, draft saving, confirmed submission, pause/reload recovery, and uncertain-confirmation reconciliation.

The fixture's selectors intentionally match the candidate configuration; passing these tests validates the extension machinery, not the real portal DOM. No live Huawei request is needed for the fixture tests.

```text
manifest.json
src/
  types/mdf-payload.ts         Payload interfaces and example
  config/selectors.config.ts  Site mappings and widget vocabulary
  shared/                     Validation, step plans, protocol, recovery rules
  background/service-worker.ts
  content/
    content.ts                Content lifecycle and control bridge
    dom-engine.ts             Framework input and wait adapters
    mdf-runner.ts             Workflow step executor
    overlay.ts                Highlight and emergency controls
  sidepanel/                  HTML, TypeScript UI, Tailwind styles
scripts/build.mjs             Multi-entry build without remote runtime assets
examples/mdf-application.json
tests/                        Unit tests and intercepted browser fixture
```

## Implementation references

- [Huawei MDF application workflow](https://support.huaweicloud.com/intl/en-us/usermanual-bpconsole/apb_060001.html)
- [Chrome Side Panel API](https://developer.chrome.com/docs/extensions/reference/api/sidePanel)
- [Chrome service-worker lifecycle](https://developer.chrome.com/docs/extensions/develop/concepts/service-workers/lifecycle)
- [Chrome webRequest API](https://developer.chrome.com/docs/extensions/reference/api/webRequest)
