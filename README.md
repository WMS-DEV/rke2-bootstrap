# rke2-bootstrap

Bootstrap engine for RKE2 clusters on Proxmox. Consumed as a submodule by [infra-iac](https://github.com/WMS-DEV/infra-iac).

## Contents

### `terraform/`

Reusable Terraform module (`modules/rke2-cluster`) that provisions cluster VMs on Proxmox via cloud-init. Consumed by per-cluster Terraform configs in `infra-iac/clusters/`.

See `terraform/cloud-init/README.md` for cloud-init multipart details.

### `ansible/`

Ansible roles and playbooks that bootstrap RKE2 onto provisioned VMs — installing and configuring the control plane, joining workers, and enrolling clusters into ArgoCD.

See [`ansible/README.md`](ansible/README.md) for architecture details.

## Usage

See [cluster-bootstrap](https://github.com/WMS-DEV/infra-wiki/blob/main/docs/infra/cluster-bootstrap.md) in infra-wiki.
