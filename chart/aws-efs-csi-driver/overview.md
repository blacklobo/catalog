## About this Helm chart

This is an AWS EFS CSI Driver Docker Hardened Helm chart built from the upstream AWS EFS CSI Driver Helm chart and using
a hardened configuration with Docker Hardened Images.

The following Docker Hardened Images are used in this Helm chart:

- `dhi/aws-efs-csi-driver`
- `dhi/livenessprobe`
- `dhi/csi-node-driver-registrar`
- `dhi/csi-provisioner`

To learn more about how to use this Helm chart you can visit the upstream documentation:
[https://github.com/kubernetes-sigs/aws-efs-csi-driver/tree/master/charts/aws-efs-csi-driver](https://github.com/kubernetes-sigs/aws-efs-csi-driver/tree/master/charts/aws-efs-csi-driver)

### About AWS EFS CSI Driver

The Amazon EFS Container Storage Interface (CSI) driver allows Kubernetes clusters running on AWS to mount Amazon EFS
and Amazon S3 file systems as persistent volumes. It supports static provisioning of an existing file system and dynamic
provisioning, where a persistent volume claim creates an access point in a file system. The driver packages the
`efs-utils` mount helper, which mounts file systems with encryption in transit enabled by default.

The chart deploys the `efs-csi-controller` Deployment, which provisions volumes, and the `efs-csi-node` DaemonSet, which
mounts volumes on every node, together with the `efs.csi.aws.com` CSIDriver object.

For more information and documentation see https://github.com/kubernetes-sigs/aws-efs-csi-driver.

## About Docker Hardened Images

Docker Hardened Images are built to meet the highest security and compliance standards. They provide a trusted
foundation for containerized workloads by incorporating security best practices from the start.

### Why use Docker Hardened Images?

These images are published with near-zero known CVEs, include signed provenance, and come with a complete Software Bill
of Materials (SBOM) and VEX metadata. They're designed to secure your software supply chain while fitting seamlessly
into existing Docker workflows.

## Trademarks

Kubernetes® is a registered trademark of The Linux Foundation. Amazon Web Services, AWS, Amazon EFS, and Amazon S3 are
trademarks or registered trademarks of Amazon.com, Inc. or its affiliates. All rights in these marks are reserved to
their respective owners. Any use by Docker is for referential purposes only and does not indicate sponsorship,
endorsement, or affiliation.
