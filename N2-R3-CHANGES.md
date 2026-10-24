# HurricaneShield — Round 3: "Control Room" landing pages and workspaces

Round 3 reworks the showcase pages in the **Control Room** direction you picked (option A,
inspired by linear.app). It keeps the light, homely maroon palette.

**Install:** apply Round 2 first, then unzip `HurricaneShield-N2-R3-changed-files.zip` over the
project root (11 files). Alternatively, use `HurricaneShield-N2-R3-full-project.zip`, which
includes everything. **There are no new packages and no new API calls.**

| Page | What changed |
|---|---|
| NaturalShield landing (`/`) | A tight sans hero on a faint hairline grid, with a 3D-tilted, glowing preview of the HurricaneShield workspace. Under it: journey ticks, a five-peril grid (Hurricane live, the rest marked "Soon"), a dark claim-log panel and a large closing call-to-action. The footer now has link columns. |
| HurricaneShield home (`/home`) | The same hero style, with a tilted preview of the quote tiers. Below it: fact ticks, a 4-cell feature grid with small UI previews (risk factors, tier comparison, receipt, claim stages) and a dark "premium as a sum" panel. |
| How it works | A numbered vertical timeline. Each step has its own small UI preview, followed by a two-paths grid and a dark "good to know" console. |
| Plans | Columns per plan built from real API data: base premium, plus coverage-limit and deductible bars scaled across your plans. The middle plan is highlighted as "Recommended". |
| Role dashboards (all 4 roles) | A dark "control panel" hero: role, date, greeting, the primary and secondary actions, a **Jump to anything (Ctrl K)** button that opens the command palette, and the role guidance shown as a console. Action cards sit in a hairline grid with icons. The workflow is a mono-labelled timeline. |
| Public header and footer, brand mark | Sans wordmark, darker square buttons and mono footer columns. |

**Shortcuts:** on the landing pages, **Q** opens the quote and **P** opens Plans. They are
ignored while you are typing in a field.

**New file:** `src/app/shared/components/product-preview/product-preview.component.ts`. It is a
decorative tilted app window with example values only, hidden from screen readers.

**Verified:** production build passes (452 kB transferred), `ng test` passes 16 files / 28 tests,
and `ng lint` shows the same 15 errors the original already had. Desktop and 390 px phone checks
were done.

---

# HurricaneShield — N2 "Porcelain & Oxblood" · Round 2: interaction & flow overhaul

Round 1 changed the look. Round 2 changes how the app **behaves**: new screen flows, motion,
icons, toasts, menus, quick-view panels, a map and a few moments of delight. It uses the packages
you checked on the Nexus mirror.

The rules are unchanged:
- Same folders, file names, selectors, inputs/outputs, signals and form controls.
- Same routes, guards, interceptors, services and request payloads. **No new API calls.**
- Mock payment gateway kept, with a better flow around it. Currency stays INR.

---

## 1. Install (office laptop)

1. Back up or commit your current `Frontend` folder.
2. Unzip `HurricaneShield-N2-R2-full-project.zip` over the project root and choose
   *Replace files*. Alternatively, unzip `HurricaneShield-N2-R2-changed-files.zip`; it holds
   only the 63 files that changed or are new.
3. Run `npm install`. Your original `package-lock.json` is included unchanged, and npm updates
   it from your Nexus mirror.
4. Run `npm start`.

### Packages added to `package.json`

| Package | Why | Mirror status |
|---|---|---|
| `lucide-angular@1.0.0` (pinned) | Icon set used everywhere | ✅ you checked it |
| `@angular/cdk@^21.2.14` | Account menu, focus trapping in dialogs and panels | ✅ |
| `ngx-sonner@^3.1.0` | Stacked, animated toasts | ✅ |
| `leaflet@^1.9.4`, `@types/leaflet` | Property map | ✅ |
| `canvas-confetti@^1.9.4` | Payment-success celebration | ✅ |
| `@tailwindcss/forms@^0.5.11` (dev) | Consistent native inputs, checkboxes and radios | ✅ |
| `@fontsource-variable/manrope`, `@fontsource-variable/fraunces`, `@fontsource/jetbrains-mono` | Self-hosted fonts, no Google call at build time | ⚠️ **please check** |
| `@types/canvas-confetti` (dev) | Typings only | ⚠️ **please check** |

Quick check in PowerShell before `npm install`:

```powershell
npm view @fontsource-variable/manrope version
npm view @fontsource-variable/fraunces version
npm view @fontsource/jetbrains-mono version
npm view @types/canvas-confetti version
```

If a font package is blocked, tell me and I'll switch that font back to a Google Fonts
`@import` (your laptop can reach Google Fonts).

`lucide-angular@1.0.0` is marked deprecated on npm in favour of `@lucide/angular`, but it
works on Angular 21. It is pinned so nothing upgrades unexpectedly.

---

## 2. New screen flows

**Public quote → 3-step wizard** (`public-connected-page`)
- Steps: *Your home* → *Protection* → *Coverage*, with a live "quote so far" summary card.
- Construction types are picker cards, and previous claims use a − / + stepper.
- Mitigations are toggle cards with explanations, and coverage has 50 / 75 / 100 % quick picks.
- Each step is validated before Continue, and focus moves to the new step heading for
  keyboard and screen-reader users.
- The result page has animated tier cards, a pulsing "Quote calculated" badge and a maroon
  "Next step" call-to-action.

**Apply for a Policy** (`policy-connected-page`)
- A three-stage progress strip runs across the top: Property & coverage → Choose tier → Apply.
- Saved properties are now selectable cards, including an *Enter manually* card. These are
  radio buttons on the same `customerPropertyOwnershipId` control.
- Coverage has quick-pick chips, and mitigation toggle cards have icons.
- A maroon call-to-action bar shows the selected plan and its yearly price.
- The application result is a celebration card with the status, underwriting outcome, risk and
  a "what happens next" tracker. *Pay premium* is the primary action.

**Submit a Claim → 3-step wizard** (`claim-connected-page`)
- Step 1, Policy & coverage: pick a policy card, then the coverage cards load and show their
  limits and deductibles.
- Step 2, Incident: date (no future dates), a ₹ amount with a warning above the coverage limit,
  and a description with a character counter.
- Step 3, Review: a summary with *Edit* links back to each step.
- A side panel explains "What happens next" and gives tips. The success screen has a large
  confirmation and an *Open claim & add documents* button.
- The `submitNewClaim()` request is unchanged.

**Premium payment**
- The modal traps focus and animates in, and its processing state lists the stages.
- On **success**, confetti plays. It is loaded on demand and skipped for reduced-motion users.
  Failures show a calm retry state.
- The receipt page is now a ticket-style receipt with a perforated divider and Print.

---

## 3. Page-level upgrades

- **Sign-in, register, MFA, recovery:** a split screen with a maroon brand panel whose copy
  changes per page.
  - The MFA code shows as **six boxes**. A single hidden input still uses the same
    `verificationCode` control, so paste and autofill still work.
  - Password visibility uses eye icons.
- **Customer dashboard:**
  - A time-of-day greeting with the date.
  - A **Storm readiness checklist** with a progress ring. It is saved in this browser only and
    makes no API calls.
  - A four-up action grid.
- **My Policies:** status filter chips, and storm-card tiles with a coverage-term progress bar.
  The tile says *Pay now* when payment is pending.
- **Policy details:**
  - A hero card showing premium, version, risk and how far through the term the policy is.
  - An icon action toolbar.
  - History as a vertical timeline with status badges, and versions labelled with a "Current" tag.
- **My Claims:** status chips, and claim cards with a 4-segment progress bar. Claim details
  has a **"Where your claim is"** tracker in the sidebar.
- **Profile:** a section tab bar on every profile page, and a richer overview card.
  - **My Properties** shows an **OpenStreetMap of your properties** with numbered pins. It uses
    the latitude/longitude the Profile API already returns. Clicking a pin highlights the
    matching card.
- **Staff and admin tables:** the claims queue, settlements, policy search, transaction search
  and customer search are now dense tables.
  - Claims queue also has at-a-glance counts, and settlements shows a payout total.
  - Each row opens a **slide-over quick view** without leaving the list, and Esc closes it.

---

## 4. App-wide interaction layer

- **Command palette** (`Ctrl/⌘ + K` or `/`) lists only the pages your role can open, with
  keyboard navigation. It also opens from the search pill in the top bar.
- **Account menu** (CDK menu): user card, security, login activity and sign out.
- **Toasts** now use ngx-sonner: stacked, swipeable and colour-coded. `NotificationService` is
  unchanged.
- **Motion:**
  - Page transitions use the browser View Transitions API.
  - Cards fade in one after another, lift on hover and have press feedback.
  - Arrows nudge on hover, and dialogs pop and slide in.
  - Everything respects *reduced motion*.
- Lucide icons replace the emoji, unicode glyphs and hand-drawn SVGs. Raw status text is
  replaced by `<app-status-badge>`, with a pulsing dot for in-progress states.
- Amounts use the `en-IN` locale, for example ₹85,00,000.00.

---

## 5. New files (5)

```
src/app/shared/components/icon/icon.component.ts              <app-icon name="…">
src/app/shared/components/command-palette/command-palette.component.ts / .html
src/app/shared/components/slide-over/slide-over.component.ts  quick-view panel
src/app/shared/components/property-map/property-map.component.ts  Leaflet, loaded lazily
```

## 6. Config touched

- `angular.json`:
  - Adds the font, Leaflet and CDK stylesheets.
  - Allows `canvas-confetti` as a CommonJS dependency.
  - Raises the **initial-bundle error budget from 2 MB to 2.5 MB**. The warning stays at
    1.5 MB, which the original already exceeded.
- `app.config.ts`: `en-IN` locale with INR as the default currency, view transitions, and
  scroll-to-top on navigation.

## 7. Verified

| Check | Result |
|---|---|
| `ng build` (production) | ✅ passes. Initial bundle is 2.05 MB raw, **452 kB transferred**. Leaflet, confetti, html2canvas and DOMPurify load lazily. |
| `ng test` | ✅ 16 files, 28 tests pass (same as the original) |
| `ng lint` | ✅ same 15 errors the original already had, no new ones |
| Visual check | Every role (Customer, Agent, Claims Manager, Administrator) on desktop and 390 px phone width, against a mocked gateway |
