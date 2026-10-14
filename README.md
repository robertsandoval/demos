# demos

Umbrella repository for curated OpenShift customer demos, organized by product/category.

## Layout

```text
<category>/
  <demo-name>/
    README.md          # How to run the demo
    ...                # Manifests, GitOps apps, examples
```

## Categories

### ACM ([acm/](acm/))

| Demo | Description |
|------|-------------|
| [baremetal-host-inventory](acm/baremetal-host-inventory/) | Provision a spoke with ACM Host Inventory (Redfish/BareMetalHost) and install operators via OpenShift GitOps |

## Conventions

- Create new demos under the relevant category folder (for example `acm/`).
- Keep secrets out of git; ship `.example` files and document local setup in each demo README.
- Prefer kustomize for transparent, demo-friendly manifests.
