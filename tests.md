# Testing of Terraform Modules Explained

## Are Unit Tests Viable?

A useful definition of a unit test is one which mainly spends its run time in the code that we own, and very little
time in the code owned by other organizations or teams. There are other definitions, but the usefulness of this
is dictated by tightening the loop of developer's work. The progress is faster with a five-seconds test run than with
a five-hours test run as the developer waits for the result to decide whether to go further with the work or to go
back and correct the last portion of their work. The more time the test run uses for the others' code, the less time
it spends inside our code, to catch the kind of errors *this team* is able to fix by ourselves.

How does that translate to the world of Terraform reusable modules? The full Terraform run which would use the code
of a reusable Terraform module always uses some provider, and the provider always speaks to some cloud's REST API.
Typically the provider+cloud run time is much longer, orders of magnitude longer, than the module run time. Realistic
example may be 100 seconds versus less than 1 second.

To enable running multiple test cases in much shorter time than that, one can either mock the provider
or mock the cloud's REST API. Such a "mocked" component basically either returns hardcoded results or an error
when anything unexpected happens, so it only takes microseconds or so. However such method requires either:

- the Terraform team to be versed in Go language, to be able to implement their own entire mock provider,
- or, alternatively, the Terraform team to be versed enough in some language to implement their own cloud REST API,
to have intimate knowledge of the original REST API of that cloud, and to be able to configure the provider to not to
speak to the real cloud anymore.

Such large investment appears for now to be only possible through an organization-level decision, not a team-level one.
So the default is not to do the testing this way. Instead we resort to mainly `terraform apply` to the real cloud,
which one would normally call integration testing. Just as for example Go/Python projects' integration tests which could
speak to the third-party API or spin up the database instance.

As a corollary to this, the terms "Unit Test" and "Integration Test" should be used with care with Terraform. If you
mean something else than this section, clarify exactly what are your definitions and how the distinction matters in your
context.

The only test-like action which is designed to be quite quick with Terraform is the good old `terraform plan`. It's
worth running, but it can hardly be considered an exhaustive test. The Plan does not dynamically generate the
identifiers, instead reporting them as `id = (known after apply)`, so object-to-object relations cannot be verified
at all.

## Test Mainly An Empty Terraform State

The assumption is that we mainly run tests starting from an empty Terraform [state](https://www.terraform.io/docs/language/state/index.html) file. For most kinds of tests, maintaining
any permanent cloud objects and the associated Terraform state would be quite hard. The main reason is not the cost,
but the fact that breaking changes in the code tend to destroy/recreate a lot of cloud resources. Such destroys can
quite often enter a conflict of some kind preventing any further Terraform modification to bring back the well-known
state for subsequent tests.

## Test Cases: Simple Or Complex?

Per the previous sections, we resort to using real Apply in tests, and it usually brings in some cloud cost - the Apply
means the stage after the plan has been already shown to the user, approved, and when it is being deployed to the real
world.

If quick and cheap unit tests were possible, it would be preferable to have each test case responsible for one thing
and one thing only. Such code is usually readable, small, and clear. However, it remains an open question whether
to accept the associated cost of one more Apply. In some cases we prefer to sacrifice some readability and perform
multiple checks after a single Apply run. The reason is that the tests are run more often than they're read by a human.

But if you decide to create such complex test code (one Apply with multiple checks), structure it in a way that shows
that the checks are in fact independent. Use comments as needed to make that clear. This will reduce
the ball-of-spaghetti flavor and rescue some readability. And your colleagues will not resent you as much as they used to.

## Randomization

Given the current state of various Terraform providers, one can expect plenty of small differences when trying to
repeat a test even when trying to keep exactly identical conditions. Therefore the test cases should be clearly
divided between non-random and random. The majority of tests should avoid any kind of randomness; this will hopefully
reduce false negatives (less flaky tests). Less time is spent troubleshooting and devs learn to depend on tests.

The minority of tests can be random. These explore the space of unknown unknowns and take into account very high
probability of false negative results. They can be run less often. Here, a failure is more of a curiosity than a hard
stop. As much as possible, the randomness of a test case should be determined from a single _seed_, to be able to
replicate the same pseudo-random sequence again.

## The Unknown Value Pitfall

There is a pitfall which makes Terraform tests quite different in structure and scope from tests in other
languages. The pitfall is: Terraform internally marks all values as either [unknown or known](https://github.com/zclconf/go-cty/blob/e5d3f1507f80d88b60e58c4644f462636f400c98/cty/value.go#L40-L52),
and _unknown values behave differently than known values_ during the Plan stage. Terraform does not make this fact
visible upfront, so it is up to a developer to remember to consider it when creating test cases.

What is an unknown value then? This doc, simplifying matters a bit, defines the unknown value as an attribute of
Terraform `resource` or `data` that cannot be sensibly guessed from the arguments of that `resource` or `data` before
the Apply starts. During the Apply stage the unknown values are determined one by one by the providers and cease
to be "unknowns".

For example, the `replace("mystring", "my", "new")` is a known value, because it is calculated before the Apply.

The `random_integer.this.hex` is an example of an unknown value, because it doesn't even exist before the Apply stage.

The code under test can fail the Plan stage for an unknown input although it passes exactly the same test for a known
value. The opposite is not true: any code that works for unknown values seems to always work for known
values as well. The idea is then to forcefully put unknown values in as many inputs as possible.

### Inputs

For inputs, this leads to the code pattern:

```hcl2
resource "random_pet" "this" {}

# The "u_" values are unknown during the `terraform plan`
locals {
  u_true    = length(random_pet.this.id) > 0
  u_false   = length(random_pet.this.id) < 0
  u_string  = random_pet.this.id
  u_number  = length(random_pet.this.id)
  u_cidr    = "10.0.${local.u_number}.0/24"
}
```

This way you can replace a `true` known bool value with `local.u_true` unknown bool value with ease. (The `local.u_true`
is always equal to `true`, because the `length()` is always greater than zero. But only the Apply stage determines
that, the Plan stage assumes it can be either true or false.) Same for other types of inputs: you can replace any string
by `local.u_string`, and number by `local.u_number`, and so on. This way it is possible to set every input of the module
to an unknown value in quite a readable way.

This pattern uses `random_pet`, a very simple Hashicorp-provided resource which simply generates names,
e.g. "relevant-horse". It is very fast, does not communicate to any cloud, and does not incur any cost, but otherwise
does not bring any real functionality here - the code above would be usable even if we would use `aws_vpc` instead.

### Outputs

The majority of module's outputs should be consumable by subsequent Terraform code. Only the minority of outputs
exist to be merely displayed to a user when Terraform completes. We can test for some very predictable scenarios of how
our module's output is consumed by some typical user-side code. It turns out that unknown values impact this too.

The main scenario is when an output is a `map`. There is not much one can do with a map, except to perform a series
of transformations and finally feed it to some `for_each` block at some point. But it's just impossible if the keys
of the map are an unknown set - the Plan will just fail with "Invalid for_each argument". The module theoretically works
but prevents any Lego®-style building. (Such a map output can still be displayed to users, but that's it.)

```hcl2
module "mymodule" {
  source = "../../modules/mymodule"

  /* inputs */
}

### Feed all the map outputs to a dummy for_each ###

resource "random_pet" "consume_mymodule_my_map_output" {
  for_each = module.mymodule.my_map_output

  prefix = each.key
}

resource "random_pet" "consume_mymodule_second" {
  for_each = module.mymodule.second

  prefix = each.key
}

resource "random_pet" "consume_mymodule_third" {
  for_each = module.mymodule.third

  prefix = each.key
}
```

Similarly, sometimes an output is a `list`, but a user can still be predicted to convert it to a map anyway.

Another commonly seen case is a `bool` output that will be mainly used to create resources in a conditional
manner using `count`. It also cannot be an unknown value. An example test:

```hcl2
resource "random_pet" "consume_mymodule_bool_output" {
  count = module.mymodule.bool_output ? 1 : 0
}
```
