# Problem

## Description

The problem happens only when **all these are used**:

- Terraform 0.12
- empty tfstate (or with data missing)
- `merge()` function
- `data` source is being fed into `merge`

## Recommendation

Thus, the recommendations are (in order):

1. Strong recommendation to **move to Terraform 0.13**.
1.1. Alternatively, add `terraform apply --target x` to the workflow (and document it), before the general untargeted `terraform apply` or `terraform plan` commands.
    - This is exactly the first thing Terraform suggests on version 0.12:
      > The "for_each" value depends on resource attributes that cannot be determined
      > until apply, so Terraform cannot predict how many instances will be created.
      > To work around this, use the -target argument to first apply only the
      > resources that the for_each depends on.

1.2. Another possibility, don't use `data` source, instead passing object from a `variable`.
2. Avoid `merge()` function in the code that is expected to handle `data` sources.
    - This is the last and the least recommended workaround, because
      it leads to constructing overly complicated objects. These decrease readibility and maintainability for all users (even those who would be never impacted by the problem at all!) and moreover the code is hard to spot and remove in future.

## Show me the code

```terraform
# Any `resource` will do:
resource google_compute_address this {
  for_each = { for k, v in local.mymap["one"].useful : k => v }
  name     = each.key
}

# Any `data` source will do, does not matter which:
data google_client_config this {}

locals {
  mymap = {
    for k in ["one"] : k =>
    # Error: Invalid for_each argument (only when doing `tf plan` or `tf destroy` on an empty state)
    merge(
      { useful = [] },
      { accidental_attribute = data.google_client_config.this },
    )

    # Works:
    # merge(
    #   { useful = [] },
    #   { accidental_attribute = "not_a_data_source" },
    # )

    # Works, merge not used:
    # {
    #   useful = []
    #   accidental_attribute = data.google_client_config.this
    # }

    # Main take away: terraform 0.12 sometimes fails on merge() and data.* combined. Do not blame flatten() or lookup() functions.
  }
}
```

General upstream issue that tracks this: [https://github.com/hashicorp/terraform/issues/4149](https://github.com/hashicorp/terraform/issues/4149)
