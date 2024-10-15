# Pipeline Tests

We have smoke tests and regression tests in our deployment pipelines. 
This explains how to run, skip tests, or run the test but ignore the results.

If you do not have a pin, it will always reset to normal `run-tests` mode.
This is so if someone wants to override one off, they don't have to remember to switch it back.

If smoke or regressions tests are failing, but we still want to deploy, we can pin the toggles
for the `smoke-test-{std|prod}-{region}-toggle` or `regression-test-{region-group}-toggle` resources:

| state          | description                                                         |
|----------------|---------------------------------------------------------------------|
| ignore-results | run the test, but ignore the results, always pass the concourse job |
| skip-tests     | skip the tests, pass the job                                        |
| run-tests      | run the test, and fail the concourse job if tests fail              |

1. Click on the skip resource you wish to enable or disable or bypass
2. Pin either `ignore-results`, `skip-tests`, or `run-tests`
3. Re-run the smoke test or regression test
4. Un-pin the resource after the test runs so next time so it's not always disabled or enabled

![bypass smoke test image](../img/bypass-smoke-test.png)
