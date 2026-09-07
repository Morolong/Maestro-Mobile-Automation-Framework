# Maestro Mobile Automation Suite

This repo automates the login → product → cart → checkout journey in the
Sauce Labs My Demo App (Android) using the [Maestro](https://maestro.mobile.dev/)
test automation framework.

## Setup

1. Install Android Studio: https://developer.android.com/studio/install
2. Install the Maestro CLI: https://maestro.mobile.dev/getting-started/installing-maestro
   (this also gives you `maestro studio` for interactive selector inspection)
3. Download the demo app APK from the
   [releases page](https://github.com/saucelabs/my-demo-app-android/releases)
   and install it on an emulator or device:
   ```
   adb install My-Demo-App-Android.apk
   ```
4. Confirm the installed app's package name matches the `appId` used
   throughout this suite: `com.saucelabs.mydemoapp.android`

## Project structure

```
maestro-suite/
├── .maestro/
│   └── config.yaml                
│
├── flows/                         
│   │                               
│   ├── authentication/
│   │   ├── navigate-to-login.yaml
│   │   ├── login.yaml
│   │   └── logout.yaml
│   ├── products/
│   │   ├── navigate-to-product.yaml
│   │   ├── increase-product-count.yaml
│   │   └── add-to-cart.yaml
│   └── checkout/
│       ├── go-to-cart.yaml
│       ├── proceed-to-checkout.yaml
│       ├── fill-valid-shipping-address.yaml
│       ├── proceed-to-payment.yaml
│       ├── fill-valid-payment-details.yaml
│       ├── fill-invalid-payment-details.yaml
│       ├── review-order.yaml
│       ├── place-order.yaml
│       └── continue-shopping.yaml
│
├── tests/                          
│   │                               
│   ├── authentication/
│   │   ├── successful-login.yaml
│   │   ├── invalid-login.yaml
│   │   └── logout.yaml
│   ├── products/
│   │   └── product-journey.yaml
│   └── checkout/
│       ├── valid-checkout-journey.yaml
│       └── invalid-payment-method-test.yaml
│
└── evidence/
    └── failure-investigation.md   
                                    
```

`tests/` is what CI points at; `flows/` is what `tests/` is built from. This
split is what keeps a login regression failing the login test specifically,
instead of cascading through every downstream test that also needs a
logged-in user.

## Running the tests

Run everything:
```
maestro test .maestro/config.yaml
```

Run one test:
```
maestro test tests/authentication/successful-login.yaml
```

## Test data

Test data is currently defined inline inside each test. With more time, I'd
extract this into a single config/fixture file so credentials, addresses,
and card details are managed from one source, which would make the suite
easier to maintain and safer to update (e.g. rotating a test card number in
one place instead of N).

## Selector strategy

The most reliable selectors are those that use the field's `id`, and that's
the preferred approach here. The app doesn't expose all fields via an id
selector, so some selectors fall back to accessibility label, index, or
placeholder text, which is more prone to flakiness if the app's copy or
layout shifts. Where that happens it would be best to work with the Dev
team to get stable resource ids added for those components rather than
keep working around it on the test side.

## Stability

- `scrollUntilVisible` is used anywhere a target might be off-screen.
- `extendedWaitUntil` (10s) is used after actions that trigger client-side
  validation or a screen transition, instead of a fixed sleep.
- `launchApp: clearState: true` is only ever used at the *start* of a
  Journey, never mid-flow.
- Flows are single-purpose and don't assert business outcomes themselves,
  so a flaky *action* and a wrong *assertion* fail at different, clearly
  distinguishable points.
- Tests never `runFlow` another test file, so a login regression fails the
  login test specifically instead of cascading false failures through
  every downstream test.

## Planned CI/CD Execution

### When it runs

| Trigger | Scope | Purpose |
|---|---|---|
| Pull request touching app code or `tests/`/`flows/` | Smoke subset: `successful-login`, `invalid-login`, `product-journey` | Fast (<10 min) sanity check before merge |
| Merge to `main` | Full suite (`.maestro/config.yaml`) | Confirms the merged state is healthy |
| Nightly schedule | Full suite, against the latest APK build | Catches regressions and environment drift that PR-scoped runs miss |
| Manual (`workflow_dispatch`) | Full suite | Pre-release confidence check, or re-running against a specific APK/build number |

The CI/CD pipeline **will be configured to execute the Maestro suite against a headless Android emulator** or, where appropriate, a device cloud such as Sauce Labs. This will allow the tests to execute unattended across the defined CI/CD triggers rather than relying on a physical device.

### What happens when a critical test fails

"Critical" here means anything in the login → product → cart → checkout
core journey (i.e. everything currently wired into `.maestro/config.yaml`) -
as opposed to a scenario from the "beyond the happy path" backlog once
those are automated and could reasonably be marked non-blocking.

1. **The pipeline fails the build/job**, not just an individual step, so it
   can't be missed in a green checkmark.
2. **On a PR**, this blocks merge - the branch protection rule requires the
   Maestro smoke job to pass.
3. **On main/nightly**, there's nothing left to block, so instead:
   - A notification fires to a chat channel or ticketing system (the
     README's "known limitations" section flags this as not yet wired up -
     today this step is a manual write-up, e.g.
     `evidence/failure-investigation.md`).
   - The failure is **not auto-retried blindly**. A single automatic retry
     is acceptable to filter out obvious infra/emulator flakiness, but a
     failure that reproduces on retry is treated as real and left failing,
     not silently re-run until green.
   - If it's a release-blocking run (manual trigger ahead of a release),
     the release is held until the failure is triaged.
4. **Triage follows the same steps as `evidence/failure-investigation.md`**:
   read the CLI/JUnit output for the exact step and string, check the
   failure screenshot, cross-reference the flow file's git history against
   recent app changes, and reproduce locally with `maestro studio` before
   deciding whether to fix the test or file a bug against the app.

### What test evidence is retained

- **JUnit XML report** from the Maestro run - this is what feeds the CI
  system's pass/fail UI and history/trend view.
- **Full CLI/console log** for the run.
- **Screenshots Maestro captures automatically at the point of failure**
  for every failed step.
- All of the above are uploaded as CI artifacts (already partially wired
  up per "known limitations" below) with a retention window (e.g. 14-30
  days) that's long enough to investigate but doesn't accumulate storage
  indefinitely.
- Nightly/full-suite runs additionally keep a lightweight pass/fail history
  per test over time, so a test that fails intermittently (candidate for
  the "treat as flaky" path in `evidence/failure-investigation.md`) is
  visible as a pattern rather than a one-off.

Example CI shape (GitHub Actions), for illustration:

```yaml
name: maestro-suite
on:
  pull_request:
    paths: ["tests/**", "flows/**", ".maestro/**"]
  push:
    branches: [main]
  schedule:
    - cron: "0 2 * * *"   # nightly
  workflow_dispatch: {}

jobs:
  maestro:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Start Android emulator
        run: ./ci/start-emulator.sh
      - name: Install demo app APK
        run: adb install My-Demo-App-Android.apk
      - name: Run Maestro suite
        run: |
          maestro test .maestro/config.yaml \
            --format junit \
            --output evidence/report.xml \
            --debug-output evidence/
      - name: Upload evidence
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: maestro-evidence
          path: evidence/
          retention-days: 21
      - name: Notify on failure
        if: failure() && github.event_name != 'pull_request'
        run: ./ci/notify-failure.sh
```

## Known limitations / what I'd improve with more time

- Confirm every `ASSUMPTION FLAGGED` string against a live emulator - this
  suite has still not been run against one.
- Replace the accessibility-label/index and placeholder-text selectors with
  real resource ids once the app team can provide them.
- Implement CI/CD integration - configure the Maestro test suite to execute automatically within a CI/CD pipeline, including smoke-test execution on pull requests, full-suite execution on merges, scheduled nightly execution, and manual workflow execution for release validation.
- Implement JUnit reporting - configure Maestro test execution to generate JUnit-compatible test reports and integrate these reports into the CI/CD pipeline so that test results, pass/fail status, and execution history can be surfaced through the CI/CD platform.
- Implement evidence/ artifacts - configure the CI/CD pipeline to collect and retain screenshots, CLI logs, and other relevant test evidence generated during failed executions. evidence/failure-investigation.md will continue to provide the documented approach for investigating test failures.
- Implement automated failure notifications - add a ticketing or chat notification step for nightly and full-suite failures so that relevant failures can be surfaced automatically to the appropriate team.
- Centralize test data (see "Test data" above).

## Beyond the happy path

Additional scenarios I'd test if time allowed, beyond what's automated:

1. Removing an item from the cart and confirming the total/quantity
   updates.
2. Attempting checkout with an empty cart.
3. Entering a shipping address with an unsupported/invalid postal code
   format for that country.
4. Killing/backgrounding the app mid-checkout and confirming cart/form
   state on relaunch.
5. Network interruption during "Place Order" - does the app show an error
   and avoid double-charging or duplicate order submission on retry?

## Failure investigation (spec item 9)

See `evidence/failure-investigation.md` for the full write-up. Summary:

**Deliberately introduced failure** - in `tests/authentication/invalid-login.yaml`,
change the asserted string to one that doesn't exist in the app, e.g.:

```yaml
- assertVisible: "This account has been locked."
```

Running `maestro test tests/authentication/invalid-login.yaml` against this
fails at that assertion: `assertVisible` times out and Maestro reports the
element/text was not found on screen within the timeout.

**How it was diagnosed:** read the CLI output top to bottom (it names the
exact step and string), open the auto-saved failure screenshot, cross-check
the live element tree in `maestro studio`, and - if the screenshot looks
correct but the assertion still failed - rerun with a longer
`extendedWaitUntil` to rule out a timing/race issue before concluding the
copy itself changed.

## Defect found: invalid credentials do not block app access

**Summary:** When logging into the app with invalid credentials, the user
is still able to access the app, browse products, add items to the cart,
and complete checkout.

**Severity/Priority:** Critical - authentication is not enforced, so any
functionality behind login is reachable without a valid account. This
would be a release blocker in a real product.

**Steps to reproduce:**
1. Launch the app to the login screen.
2. Enter an invalid username/password combination (e.g.
   `incorrect user` / `wrongPassword`, as used in
   `tests/authentication/invalid-login.yaml`).
3. Submit the login form.
4. Observe that the app allows navigation into the Products screen and
   beyond, rather than blocking access.

**Expected result:** Login should be rejected, an appropriate error should
be shown, and the user should remain on the login screen with no access to
authenticated screens.

**Actual result:** The user reaches the Products screen and can select
items, add them to the cart, and complete checkout - the same as an
authenticated user would.

**Impact:** Authentication is effectively bypassed for the demo app's core
flows, which is a functional security defect, not just a UX/copy issue.

**Suggested next step:** File this as a bug against the app (not the test
suite) with the reproduction steps above, and add a regression test once
fixed that explicitly asserts the user is *blocked* from Products after an
invalid login attempt - `tests/authentication/invalid-login.yaml` currently
only asserts the Login screen re-appears, not that authenticated screens
are unreachable; that assertion gap should be closed alongside the app fix.
