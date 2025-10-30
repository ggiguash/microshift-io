# Run MicroShift Upstream

MicroShift can be run on the host or inside a Bootc container.

## MicroShift RPM Packages

### Install RPM

Run the following command to install MicroShift RPM packages from the local
repository copied from the build container image.
See [Create RPM Packages](../docs/build.md#create-rpm-packages) for more information.

```bash
RPM_REPO_DIR=/tmp/microshift-rpms

sudo ./src/rpm/create_repos.sh -create "${RPM_REPO_DIR}"
sudo dnf install -y microshift microshift-kindnet
sudo ./src/rpm/create_repos.sh -delete
```

The following optional RPM packages are available in the repository. It is
mandatory to install either `microshift-kindnet` or `microshift-networking`
to enable the Kindnet or OVN-K networking support.

| Package               | Description                | Comments |
|-----------------------|----------------------------|----------|
| microshift-kindnet    | Kindnet CNI                | Overrides OVN-K |
| microshift-networking | OVN-K CNI                  | Uninstall Kindnet to enable OVN-K |
| microshift-topolvm    | TopoLVM CSI                | Install to enable storage support |
| microshift-olm        | Operator Lifecycle Manager | See [Operator Hub Catalogs](https://okd.io/docs/operators/) |

### Start MicroShift Service

Run the following commands to configure the minimum required firewall rules,
disable LVMS, and start the MicroShift service.

```bash
sudo ./src/rpm/postinstall.sh
sudo systemctl start microshift.service
```

Verify that all the MicroShift pods are up and running successfully.

```bash
mkdir -p ~/.kube
sudo cat /var/lib/microshift/resources/kubeadmin/kubeconfig > ~/.kube/config

oc get pods -A
```

## MicroShift DEB Packages

### Install DEB

Run the following command to install MicroShift DEB packages from the local
repository copied from the build container image.
See [Create DEB Packages](../docs/build.md#create-deb-packages) for more information.

```bash
DEB_REPO_DIR=/tmp/microshift-rpms/deb
sudo ./src/deb/install.sh "${DEB_REPO_DIR}"
```

The following optional DEB packages are available in the repository.

| Package            | Description                | Comments |
|--------------------|----------------------------|----------|
| microshift-topolvm | TopoLVM CSI                | Install to enable storage support |
| microshift-olm     | Operator Lifecycle Manager | See [Operator Hub Catalogs](https://okd.io/docs/operators/) |

> Note: All of the optional packages are installed by default.

### Start MicroShift Service

Run the following command to start the MicroShift service. All the necessary system
configuration was performed during the installation step.

```bash
sudo systemctl start microshift.service
```

Verify that all the MicroShift pods are up and running successfully.

```bash
mkdir -p ~/.kube
sudo cat /var/lib/microshift/resources/kubeadmin/kubeconfig > ~/.kube/config

oc get pods -A
```

## MicroShift Bootc Image

### Start the Container

Run `make run` to start MicroShift inside a Bootc container.

The following options can be specified in the make command line using the `NAME=VAL` format.

| Name              | Required | Default  | Comments
|-------------------|----------|----------|---------
| LVM_VOLSIZE       | no       | 1G       | TopoLVM CSI backend volume
| ISOLATED_NETWORK  | no       | 0        | Use `--network none` podman option

This step creates a single-node MicroShift cluster. The cluster can be extended using `make add-node` to add one node at a time.

This step includes:
* Loading the `openvswitch` module required when OVN-K CNI driver is used
  when compiled with the non-default `WITH_KINDNET=0` image build option.
* Preparing a TopoLVM CSI backend (default 1GB, configurable via `LVM_VOLSIZE`) on the host to be used by MicroShift when
  compiled with the default `WITH_TOPOLVM=1` image build option.
* Creating a podman network for easier multi-node cluster support with name resolution.

```bash
make run
```

> Specify the `ISOLATED_NETWORK=1` make option to run MicroShift inside a Bootc
> container without Internet access.
>
> Such a setup requires a MicroShift Bootc image built with `make image EMBED_CONTAINER_IMAGES=1`.
> This ensures all the required container image runtime dependencies are embedded
> and the operating system network settings are adjusted to allow a successful
> MicroShift operation.
>
> See the [config_isolated_net.sh](../src/config_isolated_net.sh) script for more
> information.

### Container Login

Log into the container by running the following command. The commands for doing so are displayed as
part of the summary from `make run` and `make add-node`.

For example, the first node in a cluster is named `microshift-okd-1`:
```bash
sudo podman exec -it microshift-okd-1 /bin/bash -l
```

Verify that all the MicroShift services are up and running successfully.
```bash
export KUBECONFIG=/var/lib/microshift/resources/kubeadmin/kubeconfig
oc get nodes
oc get pods -A
```

### Start the Container

If you have stopped the MicroShift cluster, you can start it again using the following command.

```bash
make start
```

### Stop the Container

Run the following command to stop the MicroShift Bootc container.

```bash
make stop
```

### Add Node to Cluster

To create a multi-node cluster, you can add additional nodes after creating the initial cluster with `make run`.

```bash
make add-node
```

> Note: The `add-node` target requires a non-isolated network (`ISOLATED_NETWORK=0`). Each additional node will be automatically joined to the cluster.

### Check Cluster Status

Run the following commands to check the status of your MicroShift cluster.

```bash
# Wait until the MicroShift service is ready (checks all nodes)
make run-ready

# Wait until the MicroShift service is healthy (checks all nodes)
make run-healthy

# Show current cluster status including nodes and pods
make run-status
```

## Cleanup

### RPM

Run the following commands to delete all the MicroShift data and uninstall the
MicroShift RPM packages.

```bash
echo y | sudo microshift-cleanup-data --all
sudo dnf remove -y 'microshift*'
```

### DEB

Run the following commands to delete all the MicroShift data and uninstall the
MicroShift DEB packages.

```bash
echo y | sudo microshift-cleanup-data --all
sudo apt purge -y 'microshift*'
```

### Bootc Containers

Run the following command to stop the MicroShift Bootc container and
clean up the LVM volume used by the TopoLVM CSI backend.

```bash
make clean
```

Run the following command to perform a full cleanup, including the
MicroShift Bootc images.

```bash
make clean-all
```
