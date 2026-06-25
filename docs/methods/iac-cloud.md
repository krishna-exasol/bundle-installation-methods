# IaC & Cloud Marketplace (Terraform/OpenTofu, AMI/Azure images, deploy buttons)

> **One-liner:** Distribute the bundle as cloud infrastructure — a Terraform/OpenTofu module, a prebaked marketplace image (AMI/Azure/GCP), or a one-click deploy button — so users provision a running instance in their own cloud account.

| | |
|---|---|
| **Category** | Infrastructure-as-Code module + cloud marketplace / one-click deploy |
| **Platforms** | Cloud providers (AWS, Azure, GCP, …); not laptops/desktops |
| **Prerequisites** | A cloud account with credentials/permissions and a payment method; `terraform`/`tofu` CLI (for modules); for marketplace, a packaged image (built with Packer) and a vendor listing |
| **Handles** | acquire ✓ / verify ? (image hashing/marketplace vetting) / place ✓ / configure ✓ (variables/user-data) / start ✓ (instances/services) / update ? (re-apply / new image version) / uninstall ✓ (`terraform destroy`) |
| **Maturity fit** | Growth → Mature (for a cloud/server audience; overkill pre-product) |
| **Trust model** | Provider IAM + signed provider plugins; marketplace vendor vetting + image hashes/owner IDs; module source pinning by version/commit; remote state integrity. |

## How it works
There are three related cloud delivery shapes. A **Terraform/OpenTofu module** is reusable declarative code that, on `apply`, provisions the bundle's infrastructure — compute, network, storage, and the services themselves (often via a container runtime, a managed Kubernetes deployment, or `user_data`/cloud-init that installs and starts the components). A **cloud marketplace image** (an AWS **AMI**, Azure managed image, or GCP image, typically built with **Packer**) is a prebaked OS disk with the whole bundle already installed; the user launches an instance from it through the provider's marketplace, sometimes with metered/subscription billing. A **one-click deploy button** (CloudFormation "Launch Stack", Azure "Deploy to Azure" ARM/Bicep, or a vendor's deploy link) wraps a template behind a single URL that drops the user into the provider console pre-filled.

For the **bundle**, IaC fits a **server deployment**: provision a VM or cluster, run the **Exasol Personal/Nano** container (pinned by digest), start the **MCP server**, and place the `json-tables` CLI on the host. This is the natural home for multi-node or always-on scenarios — and notably, **Exasol Personal itself uses OpenTofu for its cloud presets**, so an IaC module is idiomatic for the DB layer. The flip side: all of this is **massive overkill for a developer's laptop**, where Docker Compose or a local installer is far simpler.

## Example
```hcl
# ---------- Consume a published Terraform/OpenTofu module ----------
# main.tf
module "exa_bundle" {
  source  = "example/exa-bundle/aws"   # registry module
  version = "1.2.0"                     # PIN — never float on latest

  instance_type   = "t3.large"
  ami_id          = "ami-0abc123exabundle"   # marketplace/prebaked image (owner-verified)
  db_image_digest = "exasol/personal@sha256:abc123…"  # digest-pinned DB container
  enable_mcp      = true
}

output "endpoint" { value = module.exa_bundle.public_endpoint }
```
```bash
# Provision / update / tear down (Terraform or the OpenTofu CLI `tofu`)
terraform init        # downloads + verifies signed provider plugins, locks versions
terraform plan        # preview the change set
terraform apply       # creates the instance, runs the DB + MCP server
terraform destroy     # full uninstall — removes all provisioned resources

# ---------- Build a marketplace image with Packer (producer side) ----------
packer build exa-bundle.pkr.hcl     # bakes an AMI/Azure image with the bundle preinstalled
# then list it in AWS/Azure/GCP Marketplace (owner ID + version become the trust anchor)

# ---------- One-click deploy button (in your README) ----------
# [![Deploy to AWS](launch-stack.svg)](https://console.aws.amazon.com/cloudformation/home#/stacks/new?templateURL=https://example.com/exa-bundle.yaml)
# [![Deploy to Azure](deploy-azure.svg)](https://portal.azure.com/#create/Microsoft.Template/uri/<urlencoded-ARM-template>)
```

## Pros
- Provisions a **fully running** deployment — compute, network, the DB, and the MCP server — not just files on disk.
- Reproducible and version-controlled: the same module yields the same environment across dev/stage/prod.
- Clean teardown (`terraform destroy`) is a genuine, complete uninstall.
- Marketplace listings add discovery, vendor vetting, and integrated/metered billing.
- One-click buttons give a near-zero-friction "try it in my cloud" path.
- Idiomatic for the Exasol DB layer, which already ships OpenTofu cloud presets.

## Cons
- **Cloud-only and overkill for a laptop** — there is no local/offline story here.
- Requires a cloud account, IAM permissions, and (often) real spend; provisioned resources cost money until destroyed.
- Steep prerequisite knowledge: Terraform/OpenTofu, provider quirks, networking, state management.
- Marketplace publishing is a heavyweight vendor process (listing, legal, image scanning, per-provider effort).
- Prebaked images go stale (OS patches, pinned image drift) and must be rebuilt and re-listed regularly.

## Security considerations
Pin everything: the **module `version`**, provider versions (committed `.terraform.lock.hcl`), the **AMI/image by owner ID + ID**, and the DB **container by digest** — floating any of these lets supply-chain drift in. Protect **remote state** (it can contain secrets): use an encrypted backend with locking and least-privilege access. Run `apply` with scoped, short-lived cloud credentials, ideally OIDC from CI rather than long-lived keys. Verify provider plugin signatures (default in `init`) and prefer modules from a trusted registry. For marketplace images, rely on the provider's owner verification and rebuild images promptly for CVE patches. See [Security](../cross-cutting/security.md) and [Versioning & pinning](../cross-cutting/versioning-pinning.md).

## Approval & maintenance burden
A **self-published module** (Terraform Registry or a Git source) needs no approval — just versioned tags and docs. A **cloud marketplace listing** is the opposite: a substantial per-provider onboarding (vendor account, legal/billing terms, security scanning, image validation), repeated for each cloud you target. Ongoing maintenance includes rebuilding images for OS/CVE patches, tracking provider API changes, and keeping example modules working against new Terraform/OpenTofu releases. Budget marketplace presence as a product line, not a one-off upload.

## Best for / Avoid when
**Best for:** server, multi-node, or always-on cloud deployments; teams that already live in Terraform/OpenTofu; offering customers a "run it in your own account" option with clean teardown; the Exasol DB layer, which is OpenTofu-native. **Avoid when:** the target is an individual developer's laptop or an offline/air-gapped box; you have no cloud-account audience; or you cannot justify the marketplace onboarding effort. For local use, prefer [Docker Compose](./docker-compose.md) or [Native installers](./native-installers.md).

## Real-world examples
- HashiCorp and community **Terraform Registry** modules (`terraform-aws-modules/*`) provision full app stacks via `module {}` + `apply`.
- GitLab, Bitnami (now Tanzu Application Catalog), and many ISVs publish prebaked **AWS AMIs** and **Azure images** in the cloud marketplaces.
- AWS "Launch Stack" CloudFormation buttons and Azure "Deploy to Azure" ARM buttons are common one-click README paths.

## See also
- [Comparison matrix](../02-comparison-matrix.md)
- [Docker Compose](./docker-compose.md)
- [Helm / Kubernetes](./helm-kubernetes.md)
- [Native OS installers](./native-installers.md)
- [Security](../cross-cutting/security.md)
- [Versioning & pinning](../cross-cutting/versioning-pinning.md)
