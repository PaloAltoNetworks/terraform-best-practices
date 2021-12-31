# Testing of Terraform Modules Explained

## How To Run The Tests

Quick start:

1. Install Go at the latest 1.* version: https://golang.org/

2. Install Terraform at the specific version that you'd like to test. Put it in your PATH.

3. Set environment variables for cloud authentication, e.g. `ARM_CLIENT_SECRET` or similar.

4. Run: go test -count 1 ./tests/...

The tests should normally run in parallel.

Cloud resources are destroyed automatically after the test, no cleanup is normally required.

VScode users should keep `Go: Test On Save` at the default false value, and not
set to true. This option is spelled `go.testOnSave` in settings.json.

Do not however run `go test .` or similar without `-count 1`. Specifying a
package (that extra dot) enables caching, which is incompatible with Terraform.

## Naming Of Test Cases

The test cases are all placed in directory `tests` at the root of the
repository, as subdirectories. We use these suffixes for the names of the
test-case subdirectories:

  - `_read`: Test of reading pre-existing resources ("brownfield" resources).
  - `_mod`: Test of modification of configuration. Applies a configuration and then tries to apply a slightly changed configuration.
  - `_e2e`: End-to-end test, which besides applying resources to the cloud, actually uses them.
  - `_plan`: Plan-only test, which is not intended to apply anything useful to the cloud. See [unit tests](./unit_tests.md).

## Test Mainly An Empty Terraform State

The assumption is that we mainly run tests starting from an empty Terraform
[state](https://www.terraform.io/docs/language/state/index.html) file. For most
kinds of tests, maintaining any permanent cloud objects and the associated
Terraform state would be quite hard. The main reason is not the cost, but the
fact that breaking changes in the code tend to destroy/recreate a lot of cloud
resources. Such destroys can quite often enter a conflict of some kind
preventing any further Terraform modification, thus preventing the recovery of
the well-known state for subsequent tests.

## Randomization

Given the current state of various Terraform providers, one can expect plenty of
small differences when trying to repeat a test even when trying to keep exactly
identical conditions. Therefore the test cases should be clearly divided between
non-random and random. The majority of tests should avoid any kind of
randomness; this will hopefully reduce false negatives (the tests will be less
flaky). Less time is spent troubleshooting and devs become confident that the
test suite is their own tool, not an obstacle.

The minority of tests can be random. These explore the space of unknown unknowns
and take into account very high probability of false negative results. They can
be run less often. Here, a failure is more of a curiosity than a hard stop. As
much as possible, the randomness of a test case should be determined from a
single _seed_, to be able to replicate the same pseudo-random sequence again.
