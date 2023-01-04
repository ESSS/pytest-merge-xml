# pytest-update-test-results

Pytest plugin to update test results on a test report from previous run.

The primary purpose of this plugin is to avoid flaky tests to break a CI
build by executing them again after a failed run. The plugin does not
reruns tests by itself, but allow the user to leverage the pytest
`--last-failed` option to execute failed tests in a separete run and
provides `--update-xml` option so if previously failed tests
passes, the XML test report will be updated (and as result, your CI
would flag the test as successfull).

In summary, your CI script would have something like:

```bash
#/bin/bash
pytest tests/my-test-suite --junix-xml pytest.xml

if [ $? -eq 2 ]  # pytest exit code for test failure
then
    # re-run failed test and update previous run report.
    pytest tests/my-test-suite --last-faield --update-xml pytest.xml
fi
```

## Alternatives

In our particular cases, most of the flakiness comes from using a
shared storage on our virtualized on-premise CI cluster, which made for
random file access times to varies in order of 10x-100x depending on
the cluster loading. This do not affect small unit tests, but greatly
affects execution time of integration and end-to-end tests, resulting
in flaky tests due timeout or race condition.

In this scenario, marking individual tests as flaky (using [flaky]) was
like beatign a dead horse. Hence, we tried `--reruns` option from **pytest-rerunfailres** (which re-run all test failures), but that didn't
work due the way **pytest-rerunfailres** [deal with session fixtures](1).

So we opted to develop this plugin to re-run tests in a separate pytest
execution and update an existing XML report. Besides working just fine with session fixtures, re-running failed tests in a separate process has other advantages:

1. By starting a separate process, we make sure no "test contamination"
occurs (memory leaks vanish, objects that live during the entire test
session are reset).
1. Since we heavily parallelize the test suite (using **pytest-xdist**),
some flaky tests can occur due high CPU/IO loading. In the re-run
we can reduce number of parallel processes to make more resources
available for each test.

[flaky]: https://github.com/box/flaky
[pytest-rerunfailres]: https://github.com/pytest-dev/pytest-rerunfailures
[1]: https://github.com/pytest-dev/pytest-rerunfailures/issues/51
