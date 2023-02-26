---
title: "terraform Cheat Sheet"
tags:
  - terraform
comments: true
---

# terraform Cheat Sheet

## Expressions

### [Conditional Expressions](https://developer.hashicorp.com/terraform/language/expressions/conditionals)

```hcl
locals {
  multi   = true
  tenancy = local.multi ? "foo,bar" : "foo"
}

output "tenancy" {
  value = local.tenancy
}
```

equal to:

```python
# check if multi is True
if multi:
  tenancy = "foo,bar"
# assume that multi is False
else:
  tenancy = "foo"
```
