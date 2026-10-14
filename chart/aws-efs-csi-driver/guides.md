## Installing the chart

### Prerequisites

- Kubernetes 1.17+
- Helm 3.8+
- An Amazon EFS file system reachable from the cluster nodes
- IAM permission for the driver to manage EFS access points, granted through EKS Pod Identity, an IAM role for the
  controller service account, or the node instance profile. See
  [Set up driver permission](https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/master/docs/install.md#set-up-driver-permission).

### Installation steps

All examples in this guide use the public chart and images. If you've mirrored the repository for your own use (for
example, to your Docker Hub namespace), update your commands to reference the mirrored chart instead of the public one.

For example:

- Public chart: `dhi.io/<repository>:<tag>`
- Mirrored chart: `<your-namespace>/dhi-<repository>:<tag>`

For more details about customizing the chart to reference other images, see the
[documentation](https://docs.docker.com/dhi/how-to/customize/).

#### Step 1: Optional. Mirror the Helm chart and/or its images to your own registry

To optionally mirror a chart to your own third-party registry, you can follow the instructions in
[How to mirror an image](https://docs.docker.com/dhi/how-to/mirror/) for either the chart, the image, or both.

The same `regctl` tool that is used for mirroring container images can also be used for mirroring Helm charts, as Helm
charts are OCI artifacts.

For example:

```console
 regctl image copy \
     "${SRC_CHART_REPO}:${TAG}" \
     "${DEST_REG}/${DEST_CHART_REPO}:${TAG}" \
     --referrers \
     --referrers-src "${SRC_ATT_REPO}" \
     --referrers-tgt "${DEST_REG}/${DEST_CHART_REPO}" \
     --force-recursive
```

#### Step 2: Create a Kubernetes secret for pulling images

The Docker Hardened Images that the chart uses require authentication. To allow your Kubernetes cluster to pull those
images, you need to create a Kubernetes secret with your Docker Hub credentials or with the credentials for your own
registry.

Follow the [authentication instructions for DHI in Kubernetes](https://docs.docker.com/dhi/how-to/k8s/#authentication).

The upstream chart installs the driver into the `kube-system` namespace, so create the secret there:

```console
kubectl create secret docker-registry helm-pull-secret \
  --namespace kube-system \
  --docker-server=dhi.io \
  --docker-username=<Docker username> \
  --docker-password=<Docker token> \
  --docker-email=<Docker email>
```

#### Step 3: Install the Helm chart

To install the chart, use `helm install`. Make sure you use `helm login` to log in before running `helm install`.

```console
helm install aws-efs-csi-driver oci://dhi.io/aws-efs-csi-driver-chart --version <version> \
  --namespace kube-system \
  --set "imagePullSecrets[0]=helm-pull-secret"
```

Replace `<version>` accordingly. If the chart is in your own registry or repository, replace `dhi.io` with your own
registry and namespace. Replace `helm-pull-secret` with the name of the image pull secret you created earlier.

If you already created the controller service account with an IAM role, as described in
[Create an IAM policy and role](https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/master/docs/iam-policy-create.md),
add the following arguments so the chart uses it instead of creating one:

```console
  --set controller.serviceAccount.create=false \
  --set controller.serviceAccount.name=efs-csi-controller-sa
```

#### Step 4: Verify the installation

The `efs-csi-controller` Deployment and one `efs-csi-node` pod per Linux node should become ready:

```console
kubectl get pod -n kube-system \
  -l "app.kubernetes.io/name=aws-efs-csi-driver-chart,app.kubernetes.io/instance=aws-efs-csi-driver"
```

Each node then lists the driver in its CSINode object:

```console
kubectl get csinode -o jsonpath='{range .items[*]}{.metadata.name}{": "}{.spec.drivers[*].name}{"\n"}{end}'
```

### Provision a volume dynamically

Dynamic provisioning creates one EFS access point per persistent volume claim. Create a StorageClass that names your
file system, either with the chart's `storageClasses` value or directly:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: <file system ID>
  directoryPerms: "700"
```

Then request storage from it:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-claim
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

EFS file systems are elastic, so the requested capacity is required by Kubernetes but not enforced. For every
StorageClass parameter, see the upstream
[parameters documentation](https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/master/docs/parameters.md).

### What this chart provisions

- Deployment `efs-csi-controller` (two replicas by default): containers `efs-plugin` (`dhi/aws-efs-csi-driver`),
  `csi-provisioner` (`dhi/csi-provisioner`) and `liveness-probe` (`dhi/livenessprobe`)
- DaemonSet `efs-csi-node` on every Linux node except Fargate and hybrid nodes: containers `efs-plugin`
  (`dhi/aws-efs-csi-driver`), `csi-driver-registrar` (`dhi/csi-node-driver-registrar`) and `liveness-probe`
  (`dhi/livenessprobe`)
- CSIDriver `efs.csi.aws.com`, created by a Helm hook and kept on uninstall
- ServiceAccounts `efs-csi-controller-sa` and `efs-csi-node-sa`, with their ClusterRoles, ClusterRoleBindings and a
  RoleBinding
- Optional: StorageClasses from the `storageClasses` value, a PodDisruptionBudget when
  `controller.podDisruptionBudget.enabled=true`, and a metrics Service when `controller.enableMetrics=true`, with a
  ServiceMonitor when `controller.serviceMonitor.enabled=true` as well

### Differences from the upstream chart

- The `efs-plugin` containers keep the upstream security posture: both pods run as root and the container is privileged,
  because the node plugin mounts file systems on the host and the controller manages access point directories. The
  Docker Hardened Images default to a nonroot user; the chart's pod security context runs them as root.
- The `csi-provisioner`, `csi-driver-registrar` and `liveness-probe` sidecars additionally drop all Linux capabilities
  and use the `RuntimeDefault` seccomp profile.
