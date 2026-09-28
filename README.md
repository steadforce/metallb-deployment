# MetalLB Deployment

This repository is an umbrella chart that packages and configures the upstream [MetalLB][metallb] Helm chart, the
load balancer implementation of the Steadforce Kubernetes clusters. It announces `LoadBalancer` service IPs from
a fixed address pool per environment in layer 2 mode.

> [!IMPORTANT]
> Never install the content of this repository on a cluster manually. This is all done by ArgoCD.

## Overview

- **Umbrella chart**: [`Chart.yaml`](Chart.yaml) declares `metallb` from `https://metallb.github.io/metallb` as
  its only dependency, and [`Chart.lock`](Chart.lock) pins the exact resolved version and digest. The matching
  subchart archive is committed in [`charts/`](charts).
- **Address pool**: [`templates/address-pool.yaml`](templates/address-pool.yaml) and
  [`templates/l2-advertisement.yaml`](templates/l2-advertisement.yaml) create an `IPAddressPool` and a matching
  `L2Advertisement` from `addressPool.name` and `addressPool.addresses`, with ArgoCD sync wave `1`. Both render
  only when the `metallb.io/v1beta1` API is available.
- **Namespace**: [`templates/namespace.yaml`](templates/namespace.yaml) creates the release namespace with the
  `privileged` Pod Security level for `enforce`, `audit`, and `warn`, which the MetalLB speaker needs.
- **Separated override values**: subchart overrides live in their own value file, apart from the umbrella chart's
  own values (see [Value Files](#value-files)).

## Prerequisites

Commands in this document run from the repository root, in one of two ways:

- From the `SteadOps-Steadies-K8s-Workplace` workbench, which already provides `helm`, `yq`, `kubectl`,
  `hetzner-k3s`, and `act`. Tool commands run directly from the workbench shell.
- From any machine with Docker installed, using the containerized examples.

## Repository Layout

| Path | Purpose |
| --- | --- |
| [`Chart.yaml`](Chart.yaml) | Umbrella chart metadata and the `metallb` dependency. |
| [`Chart.lock`](Chart.lock) | Committed lock file pinning the resolved dependency version and digest. |
| [`charts/`](charts) | Committed subchart archive matching `Chart.lock`. |
| [`values.yaml`](values.yaml) | Umbrella chart defaults (local address pool), always applied. |
| `values-*.yaml` | Subchart overrides and environment-specific overrides (see [Value Files](#value-files)). |
| [`templates/`](templates) | Address pool, L2 advertisement, and release namespace. |
| [`tests/`](tests) | helm-unittest suites. |
| [`.github/workflows/`](.github/workflows) | CI workflows calling reusable workflows. |
| [`renovate.json`](renovate.json) | Renovate dependency update rules. |

`_local/`, `tests/__snapshot__/`, and `test-output.xml` are generated locally and gitignored.

## Value Files

| File | Scope | Purpose |
| --- | --- | --- |
| [`values.yaml`](values.yaml) | All environments | Address pool `default` with the local IP range. |
| [`values-subchart-overrides.yaml`](values-subchart-overrides.yaml) | All environments | Subchart overrides. |
| [`values-local.yaml`](values-local.yaml) | Local clusters | Near-zero resource requests and limits. |
| [`values-development.yaml`](values-development.yaml) | Development clusters | Development IP range. |
| [`values-production.yaml`](values-production.yaml) | Production clusters | Production IP range. |

`values-subchart-overrides.yaml` sets, under the top-level `metallb:` key, the resource requests and limits of the
controller and of the speaker containers, a `timeoutSeconds` of `3` for the speaker's liveness and readiness
probes, and disables FRR, which is only needed for BGP mode.

> [!NOTE]
> Subchart overrides live in their own file because Helm does not allow disabling the use of `values.yaml`.
> Keeping them separate lets the unit tests catch incompatible changes in the subchart's values.

## IP Ranges

To avoid problems inside the Steadforce network, the load balancers use fixed IP ranges known by our Sys-Team.
The first host of each block is reserved for the gateway, so the address pool starts at the second host.

| Environment | Network | Gateway (HostMin) | Broadcast | Address pool |
| --- | --- | --- | --- | --- |
| Local | `10.172.242.240/28` | `10.172.242.241` | `10.172.242.255` | `10.172.242.242-10.172.242.254` |
| Development | `10.195.5.240/28` | `10.195.5.241` | `10.195.5.255` | `10.195.5.242-10.195.5.254` |
| Production | `10.193.5.240/28` | `10.193.5.241` | `10.193.5.255` | `10.193.5.242-10.193.5.254` |

> [!WARNING]
> [`values-development.yaml`](values-development.yaml) currently configures the address pool
> `10.195.6.242-10.195.6.254`, which lies outside the documented development block `10.195.5.240/28` and does not
> include the development Istio gateway IP below. Confirm the correct block with the Sys-Team before relying on
> either value.

### Used IPs

| Environment | Service | IP |
| --- | --- | --- |
| Local | Istio gateway | `10.172.242.242` |
| Development | Istio gateway | `10.195.5.242` |
| Production | Istio gateway | `10.193.5.242` |

## Setup

`Chart.lock` and the matching subchart archive in `charts/` are committed, so a fresh clone can be rendered and
tested right away. After a pull that changes `Chart.lock`, or when `charts/` got out of sync, rebuild it with
`helm dependency build`, which downloads exactly the pinned subchart. The dependency repositories are derived
from `Chart.yaml`, the same way the pipeline does it, because `helm dependency build` only resolves registered
repositories.

From the workbench:

```shell
 yq 'explode(.) | .dependencies[] | select(.repository == "http*") | .name + " " + .repository' Chart.yaml |
   while read -r name repo; do helm repo add --force-update "$name" "$repo"; done
 helm dependency build .
```

Containerized:

```shell
 docker run \
   -e HOME=/tmp \
   --entrypoint sh \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm -c '
     yq "explode(.) | .dependencies[] | select(.repository == \"http*\") | .name + \" \" + .repository" Chart.yaml |
       while read -r name repo; do helm repo add --force-update "$name" "$repo"; done &&
     helm dependency build .
   '
```

## Rendering

This renders the chart for a local cluster into `_local/local/`. The `-a metallb.io/v1beta1` flag declares the
MetalLB API, since no cluster is queried while rendering; without it, the address pool and the L2 advertisement
are not rendered. For another environment, replace `values-local.yaml` with its value file.

From the workbench:

```shell
 helm template metallb . \
   -a metallb.io/v1beta1 \
   -f values-subchart-overrides.yaml \
   -f values-local.yaml \
   --include-crds \
   -n metallb \
   --output-dir _local/local \
   --skip-tests
```

Containerized:

```shell
 docker run \
   -e HOME=/tmp \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm template metallb . \
   -a metallb.io/v1beta1 \
   -f values-subchart-overrides.yaml \
   -f values-local.yaml \
   --include-crds \
   -n metallb \
   --output-dir _local/local \
   --skip-tests
```

## Testing

The suites in [`tests/`](tests) test the subchart's controller `Deployment` and speaker `DaemonSet` for the local,
development, and production clusters. They assert the container resources, the speaker probe timeouts, and that
FRR is disabled, and take a snapshot of each rendering.

```shell
 docker run \
   -e HELM_CACHE_HOME=/tmp/helm/.config \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   helmunittest/helm-unittest .
```

helm-unittest writes XUnit by default. To get a JUnit report in `test-output.xml`, as the pipeline does, add
`-t JUnit -o test-output.xml` before the chart path. Inside the workbench, run `helm unittest .` directly.

To update the test snapshots, add `-u`:

```shell
 docker run \
   -e HELM_CACHE_HOME=/tmp/helm/.config \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   helmunittest/helm-unittest -u .
```

> [!WARNING]
> Snapshots live in `tests/__snapshot__/`, which is gitignored and rebuilt locally on every run. They catch broad
> side effects but prove nothing on their own, because a regression is silenced by simply refreshing them. Any
> behavior that must not change belongs in a direct assertion.

### Run GitHub Workflows Locally

From the workbench, run `act` in the repository root. On first execution you are asked which flavor of the `act`
image to use; the default `medium` is a good starting point. Under `act`, the unit test workflow skips publishing
test results and sending notifications.

## CI/CD

All workflows in [`.github/workflows/`](.github/workflows) call reusable workflows from
[`steadforce/steadops-workflows`][steadops-workflows]. The unit test workflow is pinned to `v4.2.0`, the
Trufflehog workflow to `v3.0.0`:

| Workflow | Trigger | Purpose |
| --- | --- | --- |
| [`helm-unittest.yaml`](.github/workflows/helm-unittest.yaml) | Every push | Unit tests, lint, notifications. |
| [`trufflehog.yaml`](.github/workflows/trufflehog.yaml) | Push/PR to `main`, manual | Scans commits for secrets. |

- **Unit tests** register the chart's dependency repositories and run `helm dependency build` against
  `Chart.lock` (falling back to `helm dependency update` with a warning when the lock file is missing). They then
  run `helm unittest` including subchart tests, publish the JUnit results, and run `helm lint`.
- **Notifications**: on `renovate/` branches, the unit test result is posted to MS Teams. Successes go to the
  channel of the `STEADOPS_HELM_RENOVATION_MS_TEAMS_WEBHOOK` secret, failures to the separate error channel of the
  `STEADOPS_HELM_RENOVATION_ERROR_MS_TEAMS_WEBHOOK` secret. When the error secret is not set, failures go to the
  regular channel; when neither is set, no notification is sent.
- **Trufflehog** scans the pushed or pull request commit range and fails when the scan finds secrets.

## Dependency Updates

Dependency updates are managed by [Renovate][renovate] (see [`renovate.json`](renovate.json)) with its recommended
preset and a dependency dashboard. No update is merged automatically: every update, including the `metallb`
subchart and GitHub Actions, is opened as a pull request for manual review. Renovate also updates the subchart
archive in `charts/` (`helmUpdateSubChartArchives`).

To change the subchart version manually, edit the dependency version in `Chart.yaml`, then regenerate
`Chart.lock` and the archive in `charts/`:

```shell
 docker run \
   -e HOME=/tmp \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm dependency update .
```

Inside the workbench, run `helm dependency update .` directly. It replaces the old archive in `charts/` with the
new one. Commit both changes together with `Chart.yaml` and the new `Chart.lock`, otherwise the unit tests fail
because `Chart.lock` and `Chart.yaml` disagree. See the [Helm docs][helm-dependencies] for details.

[helm-dependencies]: https://helm.sh/docs/topics/charts/#chart-dependencies
[metallb]: https://metallb.io
[renovate]: https://docs.renovatebot.com
[steadops-workflows]: https://github.com/steadforce/steadops-workflows
