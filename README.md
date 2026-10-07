# Setup MAAS GitHub Action

![Tests](https://github.com/canonical/setup-maas/actions/workflows/test.yml/badge.svg)

A GitHub Action for installing and configuring MAAS in an Ubuntu LXD container.

Requires an Ubuntu runner with snapd, passwordless sudo, and LXD support. The
action configures LXD automatically and runs MAAS and its test database in a
container named `maas-backend`, independently of the runner's Ubuntu release.
Use `lxc exec maas-backend -- maas ...` for MAAS commands instead of `sudo maas`.

## Inputs

| Input | Default | Description |
| --- | --- | --- |
| `branch` | `master` | MAAS branch: `master`, `3.6`, `3.7`, or `3.8`. Accepts UI branch `main` as an alias for `master`. |
| `build-from-source` | `false` | `false`: install the published MAAS snap. `true`: clone and build the backend branch. A path: build an existing backend checkout. |
| `channel` | Derived from `branch` | Full Snap Store channel override for MAAS and `maas-test-db`. With source builds, applies only to `maas-test-db`. |
| `maas-url` | `http://localhost:5240/MAAS` | URL used to initialize MAAS. |
| `use-maasdb-dump` | `false` | Restore a test database dump. |
| `maasdb-dump-url` | MAAS master dump (see `action.yml`) | URL of a database dump compatible with the selected backend. |

| Branch input | Backend source branch | Default snap channel |
| --- | --- | --- |
| `master` (default) | `master` | `latest/edge` |
| `main` | `master` | `latest/edge` |
| `3.6` | `3.6` | `3.6/edge` |
| `3.7` | `3.7` | `3.7/edge` |
| `3.8` | `3.8` | `3.8/edge` |

The runtime image follows the snap base:

| Snap base | Ubuntu container image | Store-mode branches |
| --- | --- | --- |
| `core24` | `ubuntu:noble` (24.04) | `3.6`, `3.7` |
| `core26` | `ubuntu:resolute` (26.04) | `master`, `main`, `3.8` |

For source builds, the base is read from the checkout's `snap/snapcraft.yaml`
rather than assumed from `branch`. The container waits for cloud-init and snap
seeding before installing MAAS. Core26 uses candidate base snaps and beta snapd
to support the newer Ubuntu release; core24 uses stable channels.

Snap Store installation is the default. Set `build-from-source: true` explicitly
when the published MAAS snap is stale or broken; there is no automatic fallback
on a store failure. With `true`, source builds clone `canonical/maas` and its
pinned submodules into a temporary directory without replacing the consuming
workflow's checkout.

Alternatively, set `build-from-source` to an absolute backend checkout path or a
path relative to `github.workspace`. The action builds that checkout in place,
without cloning, switching branches, resetting, or updating submodules. Its
checkout determines the source revisions; `branch` still determines the default
`maas-test-db` channel. Missing or invalid checkouts fail without falling back to
cloning. Submodules must be initialized and match their recorded revisions.
Commit any UI submodule pointer update locally before invoking the action, since
the backend UI build reads that pointer from backend `HEAD`. Building generates
artifacts in the checkout and runs `make snap-clean` afterward.

Source builds bypass the published **MAAS** snap, not the entire Snap Store:
`maas-test-db`, LXD, Snapcraft, base snaps, and build dependencies still require
network access. Fresh clones use the UI revision pinned by the backend branch;
path-based builds use the UI revision recorded in the supplied backend checkout.

The action waits for the MAAS API after initialization, including the final
restart when restoring a database dump. It polls `<maas-url>/api/2.0/version/`
and requires HTTP 200, retrying connection failures and other HTTP responses
up to 30 times with 5-second pauses. Each request has a 5-second connection
timeout and a 10-second total timeout. The probe bypasses proxies to contact
MAAS directly and fails the action if readiness is not reached. Consumers do
not need a separate API wait step; this does not check UI readiness.

## Runtime access and outputs

An LXD proxy forwards the runner's `127.0.0.1:5240` to the container's HTTP port,
preserving the default `http://localhost:5240/MAAS` URL. The container remains
running for subsequent workflow steps. The name `maas-backend` and host port
5240 must be available; the action does not replace existing containers.
Custom `maas-url` values must be reachable from the runner for the readiness
probe; changing the URL does not change the proxy's port.

| Output | Description |
| --- | --- |
| `container-name` | `maas-backend`, for subsequent `lxc` commands. |
| `maas-ip` | The container's IPv4 address, for direct access or additional forwarding. |

TLS configuration, certificate trust, HTTPS port forwarding, and Cypress
remain the consuming workflow's responsibility. The action forwards only HTTP
5240, so existing workflow forwarding of HTTPS 5443 does not conflict. If a
workflow enables TLS after setup, it should wait for the HTTPS endpoint
separately.

## Usage

### Setup

#### Commands

##### MAAS with empty test database

```yaml
- name: Setup MAAS
  uses: canonical/setup-maas@main
```

##### Build the backend snap for each UI branch

```yaml
jobs:
  test:
    runs-on: ubuntu-24.04
    strategy:
      fail-fast: false
      matrix:
        branch: ["main", "3.6", "3.7", "3.8"]
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ matrix.branch }}
      - name: Setup MAAS from source
        uses: canonical/setup-maas@main
        with:
          branch: ${{ matrix.branch }}
          build-from-source: true
```

##### Build a backend checkout with an updated UI submodule

After updating the UI submodule and committing its pointer locally in the
backend repository, pass that checkout to the action:

```yaml
- name: Setup MAAS from the workflow checkout
  uses: canonical/setup-maas@main
  with:
    branch: ${{ matrix.maas_branch }}
    build-from-source: ${{ github.workspace }}
```

For a backend checked out in a subdirectory, use a relative path such as
`build-from-source: ./maas-backend`. The values `true` and `false` are reserved;
use `./true` or `./false` if a directory has one of those names.

##### MAAS with pre-populated test database

```yaml
- name: Setup MAAS
  uses: canonical/setup-maas@main
  with:
    use-maasdb-dump: true
```

For release branches, supply a compatible `maasdb-dump-url`; the default dump
targets backend `master` and is not guaranteed to restore on older releases.

#### Create a new user

To add a new user, run `maas createadmin` inside the container in the next step:

```yaml
- name: Create MAAS admin
  run: lxc exec maas-backend -- maas createadmin --username=admin --password=test --email=fake@example.org
```
