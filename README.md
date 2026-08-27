# Provider Cloudflare

`provider-upjet-cloudflare` is a [Crossplane](https://crossplane.io/) provider for
Cloudflare that is built using [Upjet](https://github.com/crossplane/upjet) code
generation tools and exposes XRM-conformant managed resources for the Cloudflare
API.

It is generated from the [Cloudflare Terraform
provider](https://github.com/cloudflare/terraform-provider-cloudflare), so the
set of managed resources tracks that provider's schema. The version currently
generated against is pinned as `TERRAFORM_PROVIDER_VERSION` in the `Makefile`.

## Installing

Provider packages are published to the GitHub Container Registry of the
repository owner, as `ghcr.io/<owner>/provider-upjet-cloudflare`. Install a
released version with a `Provider`:

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-upjet-cloudflare
spec:
  package: ghcr.io/crossplane-contrib/provider-upjet-cloudflare:v0.1.0
```

Substitute the owner if you are installing a package built from a fork — see
[Building and publishing a package](#building-and-publishing-a-package).

Installing the provider takes a while: the package contains a CRD for every
Cloudflare resource the Terraform provider exposes, and the API server must
accept all of them before the provider reports `Healthy`. Watch it with:

```console
kubectl get providers.pkg.crossplane.io provider-upjet-cloudflare -w
```

`Installed=False` with a registry error in the condition message means the
package reference could not be pulled, not that the provider failed to start.

### Credentials

The provider reads credentials from a Kubernetes `Secret` holding a **flat JSON
object**, not a bare token. The recognised keys mirror the Terraform provider's
own arguments — `api_token`, `api_key`, `api_user_service_key`, `email`,
`base_url`, and `user_agent_operator_suffix`. An API token (rather than the
legacy global API key plus email) is the recommended form:

```console
kubectl create secret generic cloudflare-credentials \
  --namespace crossplane-system \
  --from-literal=credentials='{"api_token":"<CLOUDFLARE_API_TOKEN>"}'
```

Scope the token to only the resources you intend to manage. For DNS records and
email routing that is: Zone → Zone: Read; Zone → DNS: Edit; Zone → Email Routing
Rules: Edit.

### ProviderConfig

Crossplane v2 supports both namespaced and cluster-scoped configuration. Managed
resources in the namespaced API groups (`*.m.crossplane.io`) reference a
namespaced `ProviderConfig` or a `ClusterProviderConfig`:

```yaml
apiVersion: cloudflare.m.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
  namespace: crossplane-system
spec:
  credentials:
    source: Secret
    secretRef:
      name: cloudflare-credentials
      namespace: crossplane-system
      key: credentials
```

The legacy cluster-scoped API group (`cloudflare.crossplane.io/v1beta1`) is also
served, for resources in the non-namespaced groups. Further examples are under
[`examples/`](examples/).

## Building and publishing a package

The package is generated code, so it only needs rebuilding when one of its
inputs changes — a new Cloudflare Terraform provider release, a new
upjet/crossplane-runtime version, or a dependency security fix. Cloudflare API
changes alone do not require a rebuild; they reach this provider only once they
appear in a Terraform provider release.

Publishing is therefore a manually triggered, two-step process.

**1. Tag the release.** Run the `Tag` workflow, which creates an annotated tag
on the ref it is dispatched against (`main` by default):

```console
gh workflow run tag.yaml -f version=v0.1.0 -f message="Release v0.1.0"
```

**2. Build and publish the package.** Run the `Publish Provider Package`
workflow **against the tag created above**, so the build is reproducible from a
fixed ref:

```console
gh workflow run publish-provider-package.yml --ref v0.1.0 -f version=v0.1.0
```

Both workflows can also be dispatched from the Actions tab in the GitHub web UI
("Run workflow"). The publish workflow accepts an optional `go-version` input if
the build needs a Go toolchain other than the default.

The publish workflow pushes to `ghcr.io/${{ github.repository_owner }}`, so a
fork publishes to its own owner's registry with no changes. It authenticates
with the run's own `GITHUB_TOKEN`; no personal access token or registry secret
is required. Mirroring to a secondary registry is skipped unless the
`XPKG_MIRROR_ACCESS_ID` and `XPKG_MIRROR_TOKEN` secrets are set.

A package pushed by `GITHUB_TOKEN` is **private on first publish**. Crossplane
pulls anonymously, so a newly published package must be made public once — under
the owner's Packages → the package → Package settings → Change visibility — or
the `Provider` will fail to install with a registry authorization error.

### Upgrading the Cloudflare Terraform provider

Bump `TERRAFORM_PROVIDER_VERSION` in the `Makefile`, then regenerate and open a
pull request. CI reports schema and CRD breaking changes on the pull request, so
review that output before tagging a release.

## Developing

Run the code-generation pipeline:

```console
go run cmd/generator/main.go "$PWD"
```

Run against a Kubernetes cluster:

```console
make run
```

Build binary:

```console
make build
```

Build and publish locally, overriding the registry organisation:

```console
make all XPKG_REG_ORGS=ghcr.io/<owner> XPKG_REG_ORGS_NO_PROMOTE=ghcr.io/<owner>
```

## Report a Bug

For filing bugs, suggesting improvements, or requesting new features, please
open an [issue](https://github.com/crossplane-contrib/provider-upjet-cloudflare/issues).
