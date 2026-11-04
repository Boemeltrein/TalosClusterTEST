# ClusterTool v5.0: 

**Native Talos configuration**
This major version uses native Talos 1.14 configuration documents instead of
Talhelper. Use the new layout below; configuration from older ClusterTool versions
is not converted automatically. Run the commands from your cluster repository root.

**Flux Operator**
Flux Operator replaces the previous Flux manifest installation. A FluxInstance
defines the Flux controllers and the Git repository they synchronize.

## Before you start

**We strongly recommend a completely fresh cluster bootstrap for this major version.**

Before starting, verify that your VolSync backups and CloudNativePG (CNPG)
backups are available and accessible, and that you have the credentials required
to restore them.

This procedure creates a new cluster; it does not migrate an existing running
cluster or its storage data.

## Required Longhorn and OpenEBS values

**Your Longhorn and OpenEBS HelmRelease values must match the new Talos storage
paths.** The supplied templates already contain these settings. If you keep your
own HelmReleases, update the following values before using the new storage layout.

Longhorn — `kubernetes/apps/system/longhorn/app/helm-release.yaml`:

```yaml
spec:
  values:
    defaultSettings:
      defaultDataPath: /var/mnt/longhorn
```

For multi-node clusters, increase the Longhorn replica counts as indicated by the comments in its HelmRelease.

OpenEBS — `kubernetes/apps/system/openebs/app/helm-release.yaml`:

```yaml
spec:
  values:
    localpv-provisioner:
      localpv:
        basePath: /var/mnt/openebs
```

Merge these values into your existing HelmReleases; keep their other settings.
Custom OpenEBS StorageClasses with an explicit `BasePath` must also use the intended
path. Flux/Helm must apply the changed values; `clustertool genconfig` alone does not
update an installed chart.

The Talos patch `patches/all/31-storage.yaml` creates directory-type user volumes
at these paths on the existing EPHEMERAL filesystem. They do not create separate
partitions or reserve disk space. Talos makes these directories available to
kubelet, replacing the old `machine.kubelet.extraMounts` configuration.

**Changing these defaults does not move existing data.** Existing Longhorn disks
and OpenEBS persistent volumes retain their recorded paths. If they contain data,
plan that storage change separately before replacing the old mounts.

## Initialize and edit

Run `clustertool init`, fill in `kubernetes/config/cluster-secrets.sops.yaml`, then run
`clustertool init` again to complete setup. Existing Talos secrets are retained.

Init blocks legacy Talhelper-configured clusters. Initialize ClusterTool 5 in a new folder.

`RUNAGAIN` remains until init completes successfully, so a failed setup can be
retried. If Talos secrets already exist and `RUNAGAIN` is absent, init warns before
decrypting or changing repository files. Prefer initializing in a new folder and comparing
it with your existing configuration.

```text
talos/
├── clustertool.yaml
├── secrets.sops.yaml
├── patches/
│   ├── all/
│   ├── control-plane/
│   ├── worker/
│   └── nodes/
│       └── control-1/
├── example-patches/
└── generated/
```

- `clustertool.yaml`: cluster name, Talos/Kubernetes versions and the list of nodes.
- `patches/all/`: shared Talos settings.
- `patches/control-plane/` and `patches/worker/`: settings for that role.
- `patches/nodes/<name>/`: a node's hostname, network, disk and installer image.
- `example-patches/`: inactive examples; copy the required YAML into a patch directory.
- `generated/`: generated node configurations and client configuration; do not edit.
- `secrets.sops.yaml`: the cluster's keys, certificates and tokens; retain this file
  when regenerating configuration or adding nodes.

Patches are applied in that order: shared, role, then node. Within each directory,
YAML files are read in filename order. Later patches can override earlier settings.
Encrypt secrets with `clustertool encrypt` before committing them to Git.

## Configure the cluster name, nodes and versions

`clustertool.yaml` is required even for a single node:

```yaml
apiVersion: clustertool/v1
kind: ClusterConfig
clusterName: main
# renovate: datasource=github-releases depName=siderolabs/talos
talosVersion: v1.14.0
# renovate: datasource=docker depName=ghcr.io/siderolabs/kubelet
kubernetesVersion: v1.37.0
nodes:
  - name: control-1
    role: control-plane
    address: ${CONTROL1IP}

  # - name: control-2
  #   role: control-plane
  #   address: ${CONTROL2IP}
  # - name: worker-1
  #   role: worker
  #   address: ${WORKER1IP}
```

`clusterName` is required and sets the Talos cluster name and generated Talos client
context. The template uses `main`; choose your name before generating and applying
the configuration. It does not affect repository paths or Cilium settings.
ClusterTool no longer injects a `CLUSTERNAME` environment variable.

`apiVersion` identifies the ClusterTool file format. ClusterTool automatically
uses the alphabetically first control-plane node by `name` for bootstrap, not as
a permanent leader. A node's `name` matches its patch directory; its actual hostname is set in `HostnameConfig`.

Set bare IP addresses in `kubernetes/config/cluster-secrets.sops.yaml`, for example
`CONTROL1IP: 192.168.20.210`. Specify the subnet prefix in the node's
network patch, for example `address: ${CONTROL1IP}/24`.

Network configuration uses a logical `mgmt` interface alias defined through
`LinkAliasConfig`, rather than relying on a fixed interface name such as `eth0`.
The default selector uses `match: true`, which is intended for single-NIC
setups. On systems with multiple network interfaces, configure a more specific
selector to ensure the correct management interface is selected.

`LinkConfig` references the `mgmt` alias for network addresses and routes.
The shared `patches/control-plane/40-vip.yaml` assigns the Layer 2 VIP
to the same alias on control-plane nodes. Worker nodes do not receive
the VIP configuration.

ClusterTool substitutes variables as entered; it does not create `_IP`,
`_CIDR` or `_NETMASK` variants. The VIP must differ from the node addresses.

To add a node, add its entry and address variable, then create
`patches/nodes/<name>/` using the starter node files. Adjust its hostname,
IP, network alias selector, disk selector and installer image.

Use three control planes if you need control-plane availability during a reboot.

`kubernetesVersion` sets the generated Kubernetes component images unless a custom
patch overrides an image. Installer patches use `${TALOS_VERSION}` from
`talosVersion`. This does not change the Talosctl bundled with ClusterTool:
a different major/minor version produces a warning and asks whether to continue.
Changing version values alone does not perform an upgrade.

## Generate and apply

```sh
clustertool genconfig
clustertool talos apply
# Apply to one configured node:
clustertool talos apply control-2
```

`genconfig` generates the node configurations. `talos apply` regenerates them first,
then applies all nodes sequentially, control planes before workers. A node name
or IP selects one node. Do not run generation and apply concurrently.

For a new cluster, apply asks before bootstrapping. Initial setup applies the
configuration to all nodes, bootstraps one control plane, then installs Cilium
and the CSR approver. It waits for the remaining nodes to become ready before
continuing with the other charts and Flux.

If setup is interrupted, run apply again. When a matching
`.bootstrap-in-progress.json` exists, ClusterTool asks whether to resume.
The file is removed after successful setup. Keep the original cluster secrets.

## Restore the Git hook

Run `clustertool githook` from your cluster repository root after switching
environments or rebuilding a devcontainer. ClusterTool refreshes its embedded
tools in the local cache and replaces `.git/hooks/pre-commit` with a hook pointing
to the current environment's precommit binary. Commits remain blocked if that
binary is missing or fails. `init` and `genconfig` also install the hook and now
report installation errors instead of reporting success.

## Generate Kustomizations

`init` prepares Flux placeholders and Kustomizations. After init, maintain these
files yourself; `genconfig` and `talos apply` no longer update them.
To optionally create missing `ks.yaml` files and add resources to
`kustomization.yaml` files under `kubernetes/apps`, run:

```sh
clustertool genks
```

`kustomizations` is an alias for `genks`. Existing `ks.yaml` files are preserved.
Review the generated changes before committing. `genks` only processes
`kubernetes/apps/`. Keep optional user components in `kubernetes/components/`;
reference them from application Kustomize files. Shared configuration, repositories,
Flux entry and components are not processed by `genks`.

## Flux Operator

Set `GITHUB_REPOSITORY` in `kubernetes/config/cluster-secrets.sops.yaml` before completing
`init`. Add the generated `ssh-public-key.txt` to that repository's deploy keys. 
Run `clustertool genconfig` and `clustertool encrypt`, then commit and push 
the configuration before accepting the Flux bootstrap prompt.

At the end of a new Talos bootstrap, ClusterTool asks whether to bootstrap Flux.
It applies the required secrets, installs the Flux Operator chart and waits for
the operator to be ready, then installs the FluxInstance chart. To start this
separately on an existing cluster:

```sh
clustertool flux bootstrap
```

The configuration is in:

- `kubernetes/apps/flux-system/flux-operator/app/helm-release.yaml`:
  operator settings.
- `kubernetes/apps/flux-system/flux-instance/app/helm-release.yaml`:
  Flux controllers and `spec.values.instance.sync` settings. The default Git branch
  is `refs/heads/main`; change it if your repository uses another branch.
- `kubernetes/flux-entry/ks.yaml`: entry points for shared configuration, repositories and
  applications.
- `kubernetes/repositories/oci/flux-operator.yaml` and `flux-instance.yaml`: chart versions
  in `spec.ref.tag`.

`init` creates `kubernetes/apps/flux-system/flux-instance/app/deploy-key.sops.yaml`.
It holds the Git deploy key and must be encrypted before committing.
`kubernetes/apps/flux-system/sops-age.sops.yaml` holds the private age key and must also be encrypted
before committing. It is excluded from Kustomizations and applied directly during
Flux bootstrap. Keep the original `age.agekey` separately to decrypt a fresh clone.
ClusterTool reads bootstrap and Talos variables from `stringData` in
`kubernetes/config/cluster-secrets.sops.yaml`. Quote numeric and boolean
values so they remain strings. `clustertool encrypt` checks that all `stringData`
values in `cluster-secrets.sops.yaml` are strings before encrypting the file.
Bootstrap applies this Secret directly; Flux
manages the same file through `flux-entry-secrets` and uses the
`cluster-secrets` Secret for substitutions. There is no generated settings copy.

After chart installation, Flux continues synchronizing in the background. Check
its status with:

```sh
kubectl -n flux-system get fluxinstances
kubectl -n flux-system get gitrepositories,kustomizations,helmreleases
```

## Change an image or upgrade

Each node can use its own Image Factory schematic in `00-install.yaml`:

```yaml
installer:
  image: factory.talos.dev/metal-installer/<schematic-id>:${TALOS_VERSION}
```

Register your customization with Image Factory and replace the schematic ID.
Keep the extensions and kernel arguments that node needs.
`genconfig` and `apply` update configuration but do not replace the running image.
To activate a new schematic or Talos version on one node:

```sh
clustertool talos upgrade control-2 --talos-only
```

The node reboots and ClusterTool waits for recovery. Without `--talos-only`, the
command also performs one cluster-wide Kubernetes upgrade, even when you select
only one node. Use `clustertool talos upgrade all` to upgrade all configured nodes.

A two-control-plane cluster cannot keep etcd quorum during a reboot: rolling
upgrades are refused and apply is restricted to changes that do not reboot.
A single-control-plane upgrade causes temporary control-plane downtime.
Tuppr independently uses each node's running schematic; it does not read
`clustertool.yaml`.

## Optional configuration

For registry authentication, copy `example-patches/44-registry-auth.yaml` into the
appropriate patch directory and configure `DOCKERHUB_USER` and
`DOCKERHUB_PASSWORD`. Setting the variables alone does not enable authentication.

For NVIDIA, copy `example-patches/50-nvidia.yaml` to the relevant node patch directories
and use an image schematic with the required NVIDIA extensions.
