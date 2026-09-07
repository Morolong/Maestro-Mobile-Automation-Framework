# Failure Investigation

## Deliberately introduced failure
In `tests/invalid-login.yaml`, change the asserted error string to a
value that doesn't exist in the app, e.g.:

```yaml
- assertVisible: "This account has been locked."
```

Running `maestro test tests/invalid-login.yaml` against this will fail
at that assertion.

## What failed
`assertVisible` times out and Maestro reports the element/text was not
found on screen within the default timeout, failing the test step.

## How I identified the cause
1. Read the Maestro CLI output to find the exact
   step and the exact string it was looking for. 
2. Maestro auto-saves a screenshot of the screen at the moment of
   failure. This can be used to see what the screen actually showed.
3. Open Maestro Studio's view hierarchy against the same screen state
   (or re-run with `maestro studio` live) to see the real visible
   text/labels the test expected.
4. If the screenshot looks correct but the assertion still failed,
   rerun with a longer `extendedWaitUntil` timeout to
   rule out a race condition before concluding the copy itself changed.

## How another QA would investigate
- Start from the CLI/JUnit report: which step, which assertion, which
  flow file.
- Open the attached failure screenshot first.
- Cross-reference against the flow file's git history to see if the
  assertion change recently, or did the app's copy change.
- Reproduce locally with `maestro studio` to inspect the live element
  tree.
- If the issue cannot be reproduced locally, treat it as a potential flaky test and investigate accordingly, including  reviewing timing, synchronisation, environment differences, and recent application or test changes.
