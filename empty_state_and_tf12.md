# Problem

## Description

The problem happens only when **all these are used**:

- Terraform 0.12
- empty tfstate (or with data missing)
- command is `terraform plan` or `terraform destroy`
- `merge()` function
- `data` source is being fed into `merge`

## Recommendation

Thus, the recommendations are (in order):

1. Strong recommendation to **move to Terraform 0.13**.
1. Alternatively, add `terraform apply -target x` to the workflow, before executing `terraform plan` or `terraform destroy` commands.
1. Another possibility, don't use `data` source, instead passing object from a `variable`.
1. Avoid `merge` in the code that is expected to handle `data` objects.
    - This is the last and the least recommended workaround, because
      it leads to constructing overly complicated objects. These decrease readibility and maintainability for all users (even those who would be never impacted by the problem at all!) and moreover the code is hard to spot and remove in future.

## Show me the code

```terraform
# Any `resource` will do:
resource google_compute_address this {
  for_each = { for k, v in local.mymap["one"].useful: k => v }
  name = each.key
}

# Any `data` item will do, does not matter which:
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
      #   { accidental_attribute = "not_a_data_item" },
      # )

      # Works, merge not used:
      # {
      #   useful = []
      #   accidental_attribute = data.google_client_config.this
      # }

      # Main take away: terraform 0.12 sometimes fails on merge() and data* combined. Do not blame flatten() or lookup() functions.
  }
}
```

General upstream issue that tracks this: https://github.com/hashicorp/terraform/issues/4149