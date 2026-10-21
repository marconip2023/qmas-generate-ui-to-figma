---
name: qmas-generate-ui-to-figma
description: 'Workflow skill for generating QMAS UI pages into Figma from Claude design HTML. TRIGGER THIS SKILL WHENEVER: (1) user asks to generate/create/搬/整 a UI page into Figma from an HTML file, Claude design link, or claude.ai/design share; (2) user says "generate ui to figma" or gives a .html file + a Figma file link; (3) user asks to add screens/pages/states to the QMAS Figma file. This skill is the ORCHESTRATION layer — it works TOGETHER with the qmas-design-system skill (v14.1+), which stays MANDATORY for all token/component rules. This skill defines HOW to run the generation job: parse source → analyze states → build one frame per state → audit → screenshot proof. This is an ADD-ON that applies ONLY when the workflow ends in Figma — always used TOGETHER with qmas-design-system (which governs all tokens / components). If the UI is being built to stay in Claude Code or a Claude design / artifact and is NOT going into Figma, you do NOT need this skill — just qmas-design-system.'
---

# Generate UI To Figma — QMAS Workflow (v3)

You are converting a **Claude-designed HTML page** (standalone bundler export) into **Figma frames** inside the QMAS product file. The `qmas-design-system` skill governs every token/component decision; THIS skill governs the workflow: source parsing, state enumeration, page layout, and QA.

Working file for Product Screening: `pTcXxElQecZ60nxyuXygbj`. Token library: `T9HItYJ1nxXEJPiVF5xdxM`.

---

## STEP 1 — Parse the source HTML (never screenshot-guess)

Claude design standalone HTMLs are bundler-wrapped. The real app lives inside:

```
<script type="__bundler/template">  → JSON string of the real HTML
  └─ inside it: <script type="text/babel">  → the React JSX source
```

Extract with Python (strip `data:` URIs first — files are ~2 MB of embedded assets):

```python
import re, json
html = open(path, encoding='utf-8').read()
template = json.loads(re.search(r'<script type="__bundler/template">(.*?)</script>', html, re.S).group(1))
jsx = re.search(r'<script type="text/babel"[^>]*>(.*?)</script>', template, re.S).group(1)
```

**The JSX is the single source of truth** for layout values, copy text, token usage (`var(--color-*)`, `var(--text-*)`), and the state machine. Read it fully before building. A browser preview is optional; the JSX is not.

---

## STEP 2 — State analysis (MANDATORY before building)

> Marco's core requirement: **1 UI page = one frame PER distinct UI state**, not just the happy path.

Read the JSX `useState` variables and conditional renders. Enumerate every visually-distinct state, then present the analysis as a table (trigger → UI change) BEFORE building. Standard state checklist — check each against the source:

| State | Look for in JSX |
|---|---|
| **01 Default** | initial `useState` values (note pre-seeded values!) |
| **02 Filled / Ready** | all inputs valid, checkboxes checked |
| **03 Validation Error** | field-level `errors` object → red border (`Status=error` variant) + red `Small/Regular` message bound to `colorError` under EACH failing field; layout shifts down |
| **04 Server / Auth Error** | `authError`-style banner → Alert component `Type=Error`, fields NOT red — different error layer than 03 |
| **05 Loading / Submitting** | `submitting` flag → button label swap + `Disabled=True` on inputs/checkbox (QMAS Button has NO loading variant — use Disabled + label e.g. "Signing In…") |
| **06 Success** | success Alert / redirect — build only if user wants it |
| Empty / zero-data | list pages: empty table state |
| Overlays | Modal / Drawer → separate frame with shell underneath (design-system Rule 10) |

Trim to the states that matter (5 is typical for a form page). Micro-states (hover, focus, password-eye toggle) fold into an existing frame or skip — they live in the component library already.

**Frame naming: `PageName / 0X StateName`** — e.g. `Login / 03 Validation Error`.

---

## STEP 3 — Build the base frame (default state)

1. Frame 1440×900 (taller only if content demands), named `PageName / 01 Default`.
2. Page canvas fill: **Marco prefers `geekblue/1` (#F0F5FF)** over grey `colorBgLayout` — bind the variable, key `c67baee30b6e07c4393308c1baf141247392112d`. Login/auth pages: white `colorBgContainer` full-bleed, NO app shell (design-system Rule 10 exception). All other pages: Header + Sider required.
3. Reuse, don't rebuild: **clone the QMAS logo from an existing Header instance** on the page rather than re-drawing.
4. Bind text styles / color variables by harvesting from existing nodes on the page (`getStyleByIdAsync` / `getVariableByIdAsync` on bound nodes) — avoids re-hunting library keys.

### Verified component keys (Ant Design System - v6 (QMAS))

```
Button Primary      ea1bf5e1a1938d5470bab8292dcaa51ed0dd290a   ComponentSet
Button Default      513ffe289cdcf781eaf16e30543226c93ee91aae   ComponentSet
Button Danger       25cd90c98741332e44ef4b26c9e3b422d5dc9b92   ComponentSet
Input               0dc9d6ccf8aef53660210eeb5699d5ad99db5896   ComponentSet
Password            afac27557df3d98da9bec2bf63dda97dddb470dd   ComponentSet
Checkbox            7840ed58212302d786b54aea662e88582050eaaa   ComponentSet
Alert               2b857760f10c298235e4f180a80860fc4c6106b8   ComponentSet
Search              38e704f41fcbd4c8cf073514e70887e7f2585269   ComponentSet
Header (Top bar)    55828f969e61169f99efe12c8ac9ee29ae591d93   plain COMPONENT → importComponentByKeyAsync
Sider (side bar)    b699ab5db58bef618d8b213738d61e9409962b69   plain COMPONENT → importComponentByKeyAsync
Icon Outlined/mail  022e278b7b055ec9faa5ce0c40ea408fed803e9d   COMPONENT (in Assets page)
Icon Outlined/lock  e3d9d298691ca64a7791931d8d5f9a43e544b719   COMPONENT
Variable colorError 8f581974b3af7d77bc9833224205d4e8e209eb94
Variable geekblue/1 c67baee30b6e07c4393308c1baf141247392112d
Textarea            a494aeefb5b7a2d31b946c729c56e3524ac416f5   (value = inner TEXT node; Size=large for 40px)
Select              46d7ffc228cb7f4fb339cfe3af5ddc4b9ba36690   (display text = inner TEXT "Please Select"; caret built in)
Segmented           3159183f6713abced831d64eb1b7046a702cff06   (Block=True fills width; set each ".Segmented Item > body > Section" TEXT; first item selected by default)
Input Number        9421b0fbc2ce9fb5a9c1947ab19f0b05bc8e02d9
Tag                 b553ad232590ecf28b849edd65ab4330d76200f8   (Text#129:13 text prop, Closable#126:5 bool, Color per-hue variant)
Image               bc85110b3f8ea0aff49d13272c2f23a275aceeb6   (row thumbnail/avatar; variant Placeholder=True, 56×56)
Effect boxShadowButtonDefault  2e83ec687dcd1c3a2e3412fff002ed473fcc7d03   (use on table/filter cards — lighter than boxShadowCard)
NOTE: Input/Select/Textarea have NO value text-property — edit the inner visible TEXT node's characters (load its font first; for a filled value rebind fill to colorText, else keep the grey placeholder). Data-dense rows/fields → override that inner text style to Small/Regular (12px).

Alert sub-components (children of the Alert ComponentSet instance — use when reaching inside an Alert rather than importing separately):
.Alert Close        957804913783cc405ff129c59f1f566772a4e4ca
.Alert Title         7d4f53c9d27703b21efa30471dd2d3f8da4a02a9
.Alert Content      14bc2e9cde075e54e22da1ec4321be171286d2ff
.Alert Extra        65fb49c4a4d258e59de4169a5d069d997b430b61
```

### Hard-won gotchas (do these right the first time)

- **Button label**: text property is `'Text#73:10'`. ALWAYS pick the `Icon Only=False` variant — `Icon Only=True` shows a "+" instead of the label.
- **Button label keys differ by set**: Primary = `Text#73:10`, Default = `Text#4214:4`, Danger = `Text#4214:1`.
- **`Variant=outlined` needs a SECOND pass**: the QMAS Default button ships `Variant=solid` = a **black** button; secondary/Cancel/Add buttons must be `outlined`, but setting `Variant:'outlined'` at `createInstance` time silently doesn't stick — re-run `setProperties({Variant:'outlined'})` in a FOLLOW-UP `use_figma` call. Same timing quirk as `layoutSizingHorizontal='FILL'` on freshly created nodes (set FILL before `resize`, or re-assert it in a later pass).
- **Input prefix icons**: the `prefix` FRAME inside `.Input item` is hidden by default → set `visible = true`, then swap the inner `Icon` instance via its INSTANCE_SWAP property `'Icon#4212:96'` with an imported icon component's `.id`.
- **Input character counter**: hide the `0/200` text (`count` frame) — it's visible on some variants.
- **Input value color**: inner `input text` keeps placeholder grey; for a FILLED value, rebind its fill to `colorText`. Placeholder text stays as the component gives it.
- **Alert message**: set the inner TEXT node's characters; props `ShowIcon#150:8`, `Closable#150:7` are booleans; variant `Type=Error, Description=False, Banner=False`. Full prop map confirmed: `Type` enum = `Success/Warning/Error/Info`, `Banner`/`Closable`/`ShowIcon` booleans, `Action` variant.
- **Alert banner overlapping the title**: when retrofitting an error Alert onto an already-built state frame (not built in from scratch), it can land on top of the page title if inserted directly below it. Position the banner below the title with a proper gap first, THEN shift everything below it down — in this situation the extra push needed was ~58px, on top of the `alertHeight + 20` rule in Step 4.
- **Gradient headline** (e.g. "Welcome Back" #2BB7E4→#1677FF): gradient stops can't fully bind — bind the brand-blue stop to `colorPrimary` via `boundVariables` on the ColorStop, leave the cyan stop raw, and note the exception.
- **Icon discovery**: icons live in token-file page `↪  Assets` → `Outlined` frame (447 components, names `Outlined/<name>`). NOT on the `↪  Icon` page.
- **MCP stability**: full-file `findAll` scans of the token library CRASH the connection. Query page names first, then `loadAsync()` only the one page you need. If `use_figma` returns HTML/`connection lost`, wait and retry with a lighter query.

---

## STEP 3.5 — Auto-layout & spacing (MANDATORY — Marco's rule)

> **Everything is auto-layout. No absolute-positioned content.** Marco's #1 structural feedback: hand-placed `x/y` nodes are un-maintainable — a field can't be added or a row reordered without re-nudging everything. Every container you build MUST be a real auto-layout frame so content flows, hugs, and stays on-grid automatically.

**The rule in full:**
1. **Every container is an auto-layout frame** (`layoutMode = 'HORIZONTAL' | 'VERTICAL'`). Page shell, cards, filter bars, table, every table **row**, every **cell**, form sections, field (label+input), button rows, chips — all auto-layout. The only things positioned absolutely are true overlays (Modal, Dropdown, Toast, scrim) relative to the frame.
2. **Group atomic clusters into one auto-layout frame** so they move/scale as a unit and are easy to duplicate:
   - **Status chip** = `[icon + label]` in a horizontal AL frame (gap 6, hug) — NOT two loose nodes.
   - **Filter pill** = `[icon + text + caret]` in a horizontal AL frame.
   - **Field** = `[label (+required *) + control]` in a vertical AL frame (gap 6).
   - **Supplier chip** = `[label + close]` in a horizontal AL frame.
   - **Table row** = horizontal AL frame of fixed-width **cell** frames; the rows live in a vertical AL frame so row-to-row spacing/dividers are automatic (use per-side `strokeBottomWeight = 1` on the row for the divider, not a floating rectangle).
3. **Spacing follows the 8px grid — `4 / 8 / 16 / 24 / 32`.** Use `itemSpacing` and the four `padding*` props (never manual gaps between absolute nodes). **`4px` is allowed for dense data** (tight table cells, chip internals, icon-to-number) — reach for it only when 8 is too loose for data density; default to 8/16.
   - **Type scale tracks density too:** data-dense **table row** text and **form-field / placeholder / input value** text use **Small (12px)**, not Base (14px). Base (14px) is the default body size; drop to Small (12px) when rows or fields are data-heavy (pairs with the 4px spacing allowance). For real Input/Select/Textarea component instances, override the inner value/placeholder TEXT node's text style to `Small/Regular` — the component ships at 14px. Table column **headers** stay `Small/Strong` (12px, grey band).
4. **Sizing modes, not magic widths:** flexible columns/fields use `layoutSizingHorizontal = 'FILL'` (`layoutGrow = 1`); fixed columns use `'FIXED'`; labels/chips/pills use `'HUG'`. Row width `FIXED` to the content width; heights `HUG`. This is what makes "add a column / add a field" a one-liner.
5. **Table column alignment** comes from header cells and row cells sharing the **same fixed widths** in matching horizontal AL frames — never eyeball-align header text to row text with x coordinates.
6. **Column header = grey band:** header row gets a subtle grey fill (`colorFillTertiary` / `colorFillQuaternary`) with `colorTextSecondary` text — the standard Ant table-header look, not just grey text floating on white. **Data rows stay white** (transparent on the white card) — only the header row is grey.
7. **Card shadow for data sections:** Table cards and filter/toolbar cards use **`boxShadowButtonDefault`** (subtle) — NOT `boxShadowCard`, which is too heavy/deep for large data surfaces. (Marco's call: `boxShadowCard` looked too dark on the table + filter.)
8. **Row thumbnails / avatars use the real `Image` component** (Placeholder variant), never a hand-built frame with a picture icon.

When retrofitting an already-built absolute-positioned frame, **rebuild the container as auto-layout** rather than nudging — re-parent the existing instances/text into the new AL cells so token bindings survive.

---

## STEP 4 — Clone per state and mutate

Clone the 01 Default frame once per remaining state, then mutate each clone:

- Variant flips via `setProperties` (`Status: 'error'`, `Disabled: 'True'`, `Checked: 'True'`).
- Error messages: new TEXT bound to `Small/Regular` style + `colorError` variable, 4px below the field; shift everything below down (message ≈ +24px each) — keep the 8px grid.
- Alert banners sit between the page title and the first field (20px gaps); shift the form down by `alertHeight + 20`.

**Before positioning anything, re-read actual frame x/y from the file** — Marco moves frames between sessions; never assume the layout you left.

---

## STEP 5 — Page layout in the Figma file

- **ONE Figma page per product flow** (Marco's rule: 唔好分多 page) — all screens on one canvas.
- Main flow frames: a row at `y = 0`, `x` step = 1540 (1440 width + 100 gap).
- **Each UI page's state frames get their own row** (e.g. `y = -1000` above the main flow, or the next free row), ordered 01→0N left to right, same 1540 step.

### 5.1 Scenario / state labels (MANDATORY — Marco's rule)

Every state / scenario frame gets a canvas label above it so a programmer can tell the frames apart without opening them.

| Property | Value |
|---|---|
| Content | `S1 · <what the user did> → <result>` — e.g. `S2 · Only QR Code Name filled → warning (BUG: currently no warning)` |
| Font | Roboto SemiBold (verify the style string with `listAvailableFontsAsync` — `SemiBold` vs `Semi Bold`) |
| Size | **50pt**, line height AUTO |
| Width | same as the frame (1440), `textAutoResize = 'HEIGHT'` — long labels wrap to 2 lines, that's fine |
| Position | `x` = frame x; **bottom of label is 30px above the frame top** → `label.y = frame.y - 30 - label.height` (re-compute after wrapping) |
| Colour | bind `colorText` |
| Name | layer name `scenario-label` |

The label is a **designer annotation, not UI** → it is the one documented exception to "bind a QMAS text style" (no 50pt style exists). Do NOT Title-Case it (design-system Rule 9 annotation exception). Exclude `scenario-label` nodes from the STEP 6 binding audit.

### 5.2 Wrap a scenario set in ONE Section

- All frames of one scenario / state set (+ their labels) go into **ONE Figma Section** — never one Section per frame (Marco: "5 張一個 section 夠啦").
- Name: `<Page> / <Topic> Scenarios`, e.g. `Create QR Code / Warning Snack Bar Scenarios`.
- Padding 80px around the content; frames keep the 1540 step (100px gap) inside.
- Build: place frames + labels on the page first, compute bounds, `figma.createSection()`, `resizeWithoutConstraints`, then `appendChild` each node and convert to section-relative x/y.
- When handing off to Notion, **link the Section node** (prominent callout at the top of the task) — per-frame links go in the scenario table.

---

## STEP 6 — QA audit + proof (never skip)

1. Run the binding audit over every new frame (skip nodes inside instances): TEXT without `textStyleId`, solid fills/strokes without `boundVariables.color`, effects without `effectStyleId` → **must all be zero**.
2. Design-system checklist: exactly ONE primary CTA per frame; Title Case + acronyms; shell present (or documented exception); brand layer colors in shell only.
3. `get_screenshot` each new frame, download, and SHOW the user — report any deliberate deviations (e.g. gradient stop exception) explicitly.

---

## Deliverable summary format

Report to Marco: state analysis table → what was built (frame names + node ids) → which components/variants used → binding audit result → screenshots. In Cantonese, concise.

**Every frame you mention gets a Figma link (MANDATORY).** Whenever a reply to Marco names a
frame, section, dialog or node, write it as a clickable markdown link — never a bare node id:
`[Frame name](https://www.figma.com/design/<fileKey>/<FileName>?node-id=<id with ":" → "-">)`
e.g. `[QT · 05 Preview, then send](https://www.figma.com/design/aJIs39okLTRsQMwk96NWrC/SVSI?node-id=4990-23915)`.
In tables, put the link in the cell. This applies to every reply, not only the final summary
(Marco, 2026-10-16: 「你每次講 frame 應該要有 hyper link」).

Last synced: **2026-10-16** (v5 — every frame/node named in a reply must be a clickable Figma link, never a bare id.)

Previous: **2026-10-13** (v4 — added STEP 5.1 scenario labels: 50pt Roboto SemiBold, frame-wide, bottom 30px above the frame, bound to colorText, documented annotation exception to text-style binding + audit; STEP 5.2: one Section per scenario set (not per frame), 80px padding, link the Section in the Notion hand-off. From the QR Code warning snack bar session.)

Previous: **2026-07-17** (v3 — MAJOR update over v2. Added MANDATORY STEP 3.5 (Auto-layout & spacing): every container is a real auto-layout frame (no absolute-positioned content); group atomic clusters (status chip / filter pill / field / supplier chip / table row+cell); spacing on the 8px grid 4/8/16/24/32 (4px for dense data); FILL/FIXED/HUG sizing; shared cell widths for column alignment. Type-scale density: data-dense table rows + form field/placeholder/value text use Small (12px), not Base (14px). Table/filter cards use boxShadowButtonDefault (not boxShadowCard); data rows white, header grey band; row thumbnails use the real Image component. Form controls MUST be real Input/Select/Textarea/Segmented instances and chips real Tag (closable) — never hand-built boxes; added their component keys + 'components have no value text-prop, edit inner TEXT node'. Gotchas: per-set button label keys (Primary Text#73:10 / Default Text#4214:4 / Danger Text#4214:1), Variant=outlined needs a second setProperties pass (ships black solid), layoutSizingHorizontal=FILL must be set before resize or re-asserted. Two-skill trigger logic baked into descriptions.)

Previous: **2026-07-10** (v2 — added verified Alert sub-component keys (`.Alert Close/Title/Content/Extra`) and the full Alert prop map (`Type` enum Success/Warning/Error/Info, `Banner`/`Closable`/`ShowIcon`/`Action`); added the alert-banner-overlapping-title retrofit gotcha (~58px push). No rule changes otherwise.)

Previous: **2026-07-03** (v1 — initial version, distilled from the Product Screening + Login build sessions. Companion to qmas-design-system v14.1.)
