# Slinky Slurm on OpenShift Tutorial

## Project Overview
Step-by-step tutorial for deploying and using Slinky (the Slurm operator) on OpenShift.

## Key Details
- Helm chart: `oci://ghcr.io/slinkyproject/charts/slurm` version `1.0.1`
- Images: `quay.io/slinky-on-openshift/*` with tag `25.11.1-centos9-ohpc`
- Namespace: `slurm` (operator runs in `slinky` namespace)
- SCC: privileged SCC is required for the `default` service account
- Shared storage: PVC `shared-home` with `ReadWriteMany`, `1Gi`, storageClass `coe-netapp-nas`
- SSH access: Via `oc exec` ProxyCommand with `socat`

## Conventions
- Tutorial is written in `README.md`
- All commands use `oc` (OpenShift CLI) and `helm`
- SSH key used is `id_ed25519` (not `id_rsa`)
- Batch scripts saved to `/home` (shared storage mount point)

## Credits
- [Slinky on OpenShift](https://github.com/redhat-hpc/slinky-on-openshift)
- [Intro to Slinky (Slurm on Kubernetes!)](https://youtu.be/YBPVzde3glw)
- [Slurm Introduction (Jobs, Partitions, Nodes, and Concepts)](https://youtu.be/hhIlmi_6E7U)
