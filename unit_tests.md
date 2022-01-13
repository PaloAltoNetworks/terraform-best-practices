# Unit Tests For Terraform Modules

## Unit Tests Do Not Use Apply

A useful definition of a unit test is one which mainly spends its run time in
the code that we own, and very little time in the code owned by other
organizations or teams. There are other definitions, but the usefulness of this
is dictated by tightening the loop of developer's work. The progress is faster
with a five-seconds test run than with a five-hours test run as the developer
waits for the result to decide whether to go further with the work or to go back
and correct the last portion of their work. The more time the test run uses for
the others' code, the less time it spends inside our code, to catch the kind of
errors *this team* is able to fix by ourselves.

How does that translate to the world of Terraform reusable modules? The full
Terraform run which would use the code of a reusable Terraform module always
proceeds to the Apply stage in which a provider speaks to some cloud's REST API.
The API roundtrips are what constitutes the main part of the delay.

The only test-like action which is designed to be quite quick with Terraform is
the good old `terraform plan` which only runs the Plan stage but not the Apply
stage. It's worth running, but it can hardly be considered an exhaustive test.
The Plan does not dynamically generate the identifiers, instead reporting them
as `id = (known after apply)`, so object-to-object relations cannot be verified
at all. So, is there any test that could work in the Plan stage as a truly
useful unit test? It turns out there is: the test against the unknown value
pitfall.

## The Unknown Value Pitfall

The pitfall is: Terraform internally marks all values as either [unknown or
known](https://github.com/zclconf/go-cty/blob/e5d3f1507f80d88b60e58c4644f462636f400c98/cty/value.go#L40-L52),
and _unknown values behave differently than known values_ during the Plan stage.
Terraform does not make this fact visible upfront, so it is up to a developer to
remember to consider it when creating test cases.

What is an unknown value then? This doc, simplifying matters a bit, defines the
unknown value as an attribute of Terraform `resource` or `data` that cannot be
sensibly guessed from the arguments of that `resource` or `data` before the
Apply starts. During the Apply stage the unknown values are determined one by
one by the providers and cease to be "unknowns".

For example, the `replace("mystring", "my", "new")` is a known value, because it
is calculated before the Apply.

The `random_integer.this.hex` is an example of an unknown value, because it
doesn't even exist before the Apply stage.

The code under test can fail the Plan stage for an unknown input although it
passes exactly the same test for a known value. The opposite is not true: any
code that works for unknown values seems to always work for known values as
well. The idea is then to forcefully put unknown values in as many inputs as
possible.

Since the distinction apparently becomes irrelevant after the Plan succeeds,
this category of tests are written to only execute the Plan stage; the Apply
never runs. This greatly simplifies the setup code, because the module under
test does not need any real dependencies and all the unknown inputs may use
dummy values.

### Inputs

For inputs, this leads to the code pattern:

```hcl2
resource "random_pet" "this" {}

# The "u_" values are unknown during the `terraform plan`.
locals {
  u_true    = length(random_pet.this.id) > 0
  u_false   = length(random_pet.this.id) < 0
  u_string  = random_pet.this.id
  u_number  = length(random_pet.this.id)
  u_cidr    = "10.0.${local.u_number}.0/24"
}
```

This way you can replace a `true` known bool value with `local.u_true` unknown
bool value with ease. (The `local.u_true` is always equal to `true`, because the
`length()` is always greater than zero. But only the Apply stage determines
that, the Plan stage assumes it can be either true or false.) Same for other
types of inputs: you can replace any string by `local.u_string`, and number by
`local.u_number`, and so on. This way it is possible to set every input of the
module to an unknown value in quite a readable way.

This pattern uses `random_pet`, a very simple Hashicorp-provided resource which
simply generates names, e.g. "relevant-horse". It is very fast, does not
communicate to any cloud, and does not incur any cost, but otherwise does not
bring any real functionality here - the code above would be usable even if we
would use `aws_vpc` instead.

To take the idea further, there is not much point to use
known values in tests where an unknown value would work. The reality is that
inputs are divided - an input can either accept an unknown/known value or only
known. The test cases should show that divide in an obvious manner, so that when
a reader sees a known value used (like `true`) they can immediately infer that
this particular input is necessarily-known.

### Outputs

The majority of module's outputs should be consumable by subsequent Terraform
code. Only the minority of outputs exist to be merely displayed to a user when a
Terraform run completes. We can test for some very predictable scenarios of how
our module's output is consumed by some typical user-side code. It turns out
that unknown values impact this too.

The main scenario is when an output is a `map`. There is not much one can do
with a map, except to perform a series of transformations and finally feed it to
some `for_each` block at some point. But it's just impossible if the keys of the
map are an unknown set - the Plan will just fail with "Invalid for_each
argument". The module theoretically works but in a way which prevents any
Lego®-style building (such a map output can still be displayed to users however
and occasionally that's all what's needed).

```hcl2
module "mymodule" {
  source = "../../modules/mymodule"

  /* inputs */
}

### Feed all the map outputs to a dummy for_each ###

resource "random_pet" "consume_mymodule_my_map_output" {
  for_each = module.mymodule.my_map_output
}

resource "random_pet" "consume_mymodule_second" {
  for_each = module.mymodule.second
}

resource "random_pet" "consume_mymodule_third" {
  for_each = module.mymodule.third
}
```

On Terraform 0.13 and above condense that to a single block:

```hcl2
resource "random_pet" "consume_maps" {
  for_each = merge(
    module.mymodule.my_map_output,
    module.mymodule.second,
    module.mymodule.third,

    module.mymodule_alt.my_map_output,
    module.mymodule_alt.second,
    module.mymodule_alt.third,
  )
}
```

The `_alt` above refers to an alternate call to the same module, with a
different set of known inputs. For example, if the `vpc` module has the input
`create_vpc` which is true by default, an alternate call would have it false.
This would test Plan on multiple execution paths within the same code.

When output is a `list` there is a bit of gray area. Lists are often used as
resource arguments in practice, which are immune to unknown values. For example
`aws_instance.vpc_security_group_ids` expects a list of strings and works fine
whether the list is known or unknown, so it has completely different behavior
than `for_each`. Map arguments would be immune for the same reason, it's just
that they are rarely seen in popular providers except for the ubiquitous `tags`
map. In general, a developer judges whether an output really needs to be tested
against unknown values, it is by no means an automatic decision.

In some cases however you can judge to test whether module's users will be able
to build their resources for each element of the list. This needs `for_each`
testing quite similar to `map`:

```hcl2
resource "random_pet" "consume_mymodule_my_list" {
  for_each = { for v in module.mymodule.my_list : v => true }
}
```

On Terraform 0.13 and above condense multiple outputs to a single block:

```hcl2
resource "random_pet" "consume_lists" {
  for_each = merge(
    { for v in module.mymodule.my_list : v => true },
    { for v in module.mymodule.my_second_list : v => true },
    { for v in module.mymodule_alt.my_list : v => true },
    { for v in module.mymodule_alt.my_second_list : v => true },
  )
}
```

Another imaginable test case is a `bool` output. Sometimes you can judge that
module's users will likely need to create a resource in a conditional manner
using `count`. Such `bool` output also cannot be an unknown value. An example
test:

```hcl2
resource "random_pet" "consume_mymodule_bool_output" {
  count = module.mymodule.bool_output ? 1 : 0
}

resource "random_pet" "consume_bools" {
  count = contains([
    module.mymodule.bool_output1,
    module.mymodule.bool_output2,
    module.mymodule.bool_output3,
  ], false) ? 1 : 0
}
```
