# Providing a test262 diff for a LibJS change

LibJS is measured by [test262](https://github.com/tc39/test262), the official ECMAScript
conformance suite. A test262 diff shows which tests your change fixed or broke, and it is
the quickest way for a reviewer to see the effect of a change that touches the parser, the
runtime, or a builtin.

You do not need to run the whole suite by hand for every pull request. This page explains
what the number in a report means, how to produce one locally, and how to read the results
the CI already keeps for you.

## What a diff shows

Test262 has tens of thousands of tests and LibJS does not pass all of them yet, so the
absolute number of failures is not interesting. What matters is the change: tests that
started passing, and tests that started failing. A pull request that moves one test forward
and three backward is a regression even though the passing count went up.

## Run the suite locally

Build the runner, clone the harness, and fetch test262:

```bash
./Meta/ladybird.py build test262-runner

git clone https://github.com/LadybirdBrowser/libjs-test262.git
cd libjs-test262
pip3 install -r requirements.txt
./sync-test262.sh
```

With `LADYBIRD_SOURCE_DIR` pointing at your Ladybird checkout, `./main.py` runs the suite
against `Build/release/bin/test262-runner`. `main.py --help` lists the options that matter
for a diff run:

* `--per-file PATH` writes a per-test result file, which is what the diff reads.
* `-p GLOB` limits which test files are searched, useful while iterating on one area.
* `--ignore GLOB` skips the tests whose path matches, which is how known failures are left out.

## Produce the diff

Run the suite once on the commit before your change and once with your change applied,
writing a per-file result each time:

```bash
./main.py --per-file before.json
# rebuild with your change
./main.py --per-file after.json

python3 per_file_result_diff.py --old before.json --new after.json
```

The diff prints new tests, removed tests, and every test whose result changed. Two flags
narrow the output:

* `--regressions` shows only tests that started failing.
* `--intersection-only` ignores tests that test262 itself added or removed, so a test262
  update during your work does not look like a change in LibJS.

If rebuilding the baseline is inconvenient, use the results the CI already stores. Every
master run writes the per-file results of that run to
[libjs-data](https://github.com/LadybirdBrowser/libjs-data) as `test262/per-file-master.json`.
Take a copy of that file before you run your own, and diff against it instead of building a
baseline:

```bash
python3 per_file_result_diff.py --old per-file-master.json --new after.json
```

## What the CI does

`.github/workflows/libjs-test262.yml` runs the same suite on every pull request. It checks
out the harness, test262, and libjs-data, builds `test262-runner`, `test-js`, and `test-wasm`,
and runs them through `run_all_and_update_results.py`. Pull request runs never push results.
Master runs append their results to libjs-data, which is why the newest per-file results are
the ones to compare against.

The workflow is worth reading if a run disagrees with your local one. Failures that appear
only in the CI usually mean the local checkout of test262 has drifted, and `./sync-test262.sh`
fixes that.

## Reporting it in a pull request

Paste the summary block from the diff into the pull request description, and say which
command produced it. If the change affects only a few tests, name them. If it moves a large
number, the summary counts and the direction are enough; a reviewer can reproduce the run
from the two commands above.

The JS runtime tests in `Tests/LibJS` are separate from test262 and run with
`./Meta/ladybird.py test` as usual.
