# Using `data` object that includes `depends_on`

## Description

The problem happens where:

- compatibility is required with both Terraform 0.12.x and 0.13.x
- `data {}` block requires an attribute from a `resource {}` block, like this:

```hcl
resource google_compute_network initial {
  name                    = "example-vpc"
  auto_create_subnetworks = false
}

data google_compute_network passthrough {
  name = google_compute_network.initial.name

  # tf-0.12: Do not use depends_on!
  # tf-0.13: Needs depends_on.
  # tf-0.14: Is okay both with and without depends_on.
  depends_on = [google_compute_network.initial]
}

resource google_compute_subnetwork this {
  name          = "example-subnet"
  network       = data.google_compute_network.passthrough.self_link
  ip_cidr_range = "192.168.55.0/24"
}
```

The behavior of the example is:

- For Terraform-0.12, the consequences are quite dire and need to be documented in the corresponding README document:

  - Never run `terraform refresh`, it forces a subsequent destroy/recreate of multiple resources. Recover from it by restoring
  the tfstate from backup. No workaround available, which means that the production environment can never be refreshed to fully
  re-read the currently existing cloud resources.
  - The `apply` will also destroy/recreate of multiple resources.
  - The workaround is to always use: `terraform apply --refresh=false`
  - For planning, use: `terraform plan --refresh=false`
  - Actually tested on 0.12.29.

- For Terraform-0.13 (actually tested 0.13.6), all works well.
- For Terraform-0.14+, all is good in any variant.

## Without `depends_on`

You can simply remove the `depends_on` above.

- For Terraform-0.12 (actually tested 0.12.29), all works well.
- For Terraform-0.13:

  - The `terraform apply` just fails (except when the resource exists already).
  - The `terraform apply --refresh=false` fails in an identical manner.
  - Only some of the types of `data` objects are impacted, not sure which (e.g. it happens with `data.google_compute_network`
  but not with `data.google_compute_subnetwork`).
  - The workaround is: `terraform apply --target google_compute_network.initial`
  - The workaround is user-visible and needs to be documented in the corresponding README document.
  - Actually tested on 0.13.6.

- For Terraform-0.14+, all is good with any variant.

## Root Cause

Unsuccessful attempt at explaining the root cause: [this support thread](https://discuss.hashicorp.com/t/terraform-0-13-handling-of-data-source-data-resource-reads-can-no-longer-be-disabled-by-refresh-false/18457)
for 0.13 suggests that the behavior exists because the attribute is marked *non-computed*, however in our case the attribute
`data.google_compute_network.passthrough.self_link` is seemingly marked *computed* per the
[source code](https://github.com/hashicorp/terraform-provider-google/blob/75fb8de218b7abaa85328db31ab16af6351d0c7e/google/data_source_google_compute_network.go#L31).

## Recommendation

Thus, the alternative recommendations are (sorted starting from the best one):

1. Strong recommendation to **require the latest GA version of Terraform** enforcing it with `required_version` mechanism.
2. Test the code with Terraform 0.13.x (a `data {}` block requiring an attribute coming from a `resource {}` block), if impacted:

    - avoid this code pattern altogether,
    - alternatively, keep `depends_on` but drop the 0.12.x compatibility with `required_version = ">= 0.13"`
    - alternatively, [remove `depends_on`](#without_depends_on) from code and accept that 0.13 will not run cleanly; document the
    precise workaround commands for users.
