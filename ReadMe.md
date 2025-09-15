# Kubernetes Bootstrap Ansible Role

This Ansible role bootstraps a Kubernetes cluster by installing and configuring essential plugins and services using Helm charts and raw manifests. It sets up critical components such as ingress-nginx, cert-manager, MinIO, Velero, Prometheus, Loki, and ArgoCD, along with their configurations, to prepare a Kubernetes cluster for production use.

- [Kubernetes Bootstrap Ansible Role](#kubernetes-bootstrap-ansible-role)
  - [Features](#features)
  - [Requirements](#requirements)
  - [Role Variables](#role-variables)
    - [General Variables](#general-variables)
    - [Kubernetes Variables](#kubernetes-variables)
    - [Helm Charts](#helm-charts)
    - [MinIO Configuration](#minio-configuration)
    - [Velero Configuration](#velero-configuration)
    - [Monitoring (Prometheus, Grafana, Alertmanager)](#monitoring-prometheus-grafana-alertmanager)
    - [ArgoCD](#argocd)
    - [Loki](#loki)
  - [Usage](#usage)
    - [Example Playbook](#example-playbook)
  - [Tags](#tags)
  - [Contributing](#contributing)
  - [🔗 Stay connected with DockerMe! 🚀](#-stay-connected-with-dockerme-)


## Features
- Installs and configures Helm charts for:
  - `ingress-nginx`: Ingress controller for Kubernetes.
  - `cert-manager`: Automated certificate management with Let's Encrypt.
  - `kube-prometheus-stack`: Monitoring stack with Prometheus, Grafana, and Alertmanager.
  - `minio`: Object storage solution.
  - `velero`: Backup and restore for Kubernetes resources.
  - `loki-stack`: Log aggregation with Loki and Promtail.
  - `argo-cd`: GitOps continuous delivery tool.
- Deploys raw Kubernetes manifests from a specified directory.
- Configures MinIO with a dedicated bucket, user, and policy for Velero backups.
- Supports customizable variables for domains, resource limits, and credentials.
- Applies TLS certificates via cert-manager for secure ingress.

## Requirements
- **Ansible**: Version 2.9 or higher.
- **Ansible Collections**:
  - `kubernetes.core` (for Helm and Kubernetes manifest management).
  - `amazon.aws` (for S3 bucket operations with MinIO).
- **Python Packages**:
  - `boto3`: For AWS-compatible S3 operations with MinIO.
  - `botocore`: Dependency for `boto3`.
  - `kubernetes`: For interacting with Kubernetes APIs.
- **Kubernetes Cluster**: A running Kubernetes cluster with `kubeconfig` available.
- **Helm**: Installed on the control node for Helm chart management.
- **Access to kubeconfig**: Path to the `kubeconfig` file (default: `~/.kube/config`).
- **Proxy Environment (Optional)**: Configure `proxy_env` if operating behind a proxy.

Install the required collections and Python packages:
```bash
ansible-galaxy collection install kubernetes.core
ansible-galaxy collection install amazon.aws
pip3 install boto3 botocore kubernetes
```

## Role Variables
The role uses several variables defined in `defaults/main/main.yml` and `defaults/main/vault.yml`. Below is a summary of key variables:

### General Variables
| Variable                  | Description                              | Default Value                     |
|---------------------------|------------------------------------------|-----------------------------------|
| `general.main_domain`     | Main domain for services                 | `dena.mecan.ir`                  |
| `general.pull_policy`     | Image pull policy for containers         | `IfNotPresent`                   |
| `general.manifest_directory` | Directory containing Kubernetes manifests | `manifests` |

### Kubernetes Variables
| Variable                     | Description                              | Default Value           |
|------------------------------|------------------------------------------|-------------------------|
| `kubernetes.context`         | Kubernetes context to use                | `dena`                  |
| `kubernetes.kubeconfig_path` | Path to kubeconfig file                  | `~/.kube/config`        |

### Helm Charts
The `kubernetes_helm_charts` list defines Helm charts to install. Each chart includes:
- `name`: Chart name (e.g., `ingress-nginx`, `cert-manager`).
- `repository_url`: Helm repository URL.
- `chart`: Chart reference.
- `namespace`: Target namespace.
- `chart_version`: Specific chart version.
- `values_file_path_jinja`: Path to the Jinja2 template for Helm values.
- `values_file_path_yaml`: Destination for rendered Helm values file.
- Example:
  ```yaml
  kubernetes_helm_charts:
    - name: ingress-nginx
      repository_url: https://kubernetes.github.io/ingress-nginx
      chart: ingress-nginx
      namespace: ingress-nginx
      create_namespace: true
      state: present
      chart_version: "v4.13.0"
      wait: true
      update_repo_cache: true
      replace: true
      values_file_path_jinja: values-ingress.yaml.j2
      values_file_path_yaml: /tmp/values-ingress.yaml
  ```

### MinIO Configuration
| Variable                       | Description                              | Default Value                     |
|-------------------------------|------------------------------------------|-----------------------------------|
| `minio.image.repo`            | MinIO image repository                   | `quay.io/minio/minio`            |
| `minio.image.tag`             | MinIO image tag                          | `RELEASE.2025-09-07T16-13-09Z`   |
| `minio.endpoint.api`          | MinIO API endpoint                       | `object`                         |
| `minio.endpoint.console`      | MinIO console endpoint                   | `minio`                          |
| `minio.alias_name`            | MinIO alias name                         | `dena`                           |
| `minio_root_username`         | MinIO root username (vault)              | Sensitive (vault)                |
| `minio_root_password`         | MinIO root password (vault)              | Sensitive (vault)                |

### Velero Configuration
| Variable                        | Description                              | Default Value                     |
|--------------------------------|------------------------------------------|-----------------------------------|
| `velero.minio.bucket_name`     | Bucket for Velero backups                | `velero-backup`                  |
| `velero.minio.username`        | Velero MinIO user                        | Sensitive (vault)                |
| `velero.minio.password`        | Velero MinIO password                    | Sensitive (vault)                |
| `velero.minio.policy_name`     | Velero MinIO policy name                 | `velero`                         |

### Monitoring (Prometheus, Grafana, Alertmanager)
| Variable                              | Description                              | Default Value                     |
|--------------------------------------|------------------------------------------|-----------------------------------|
| `monitoring.prometheus.endpoint`      | Prometheus endpoint                      | `prometheus`                     |
| `monitoring.prometheus.pvc_capacity`  | Prometheus PVC size                      | `25Gi`                           |
| `monitoring.grafana.endpoint`         | Grafana endpoint                         | `grafana`                        |
| `monitoring.grafana.pvc_capacity`     | Grafana PVC size                         | `15Gi`                           |
| `grafana_root_username`              | Grafana admin username (vault)           | Sensitive (vault)                |
| `grafana_root_password`              | Grafana admin password (vault)           | Sensitive (vault)                |

### ArgoCD
| Variable                    | Description                              | Default Value                     |
|----------------------------|------------------------------------------|-----------------------------------|
| `argocd.endpoint`          | ArgoCD endpoint                          | `argocd`                         |
| `argocd.image.repo`        | ArgoCD image repository                  | `quay.io/argoproj/argocd`        |
| `argocd.image.tag`         | ArgoCD image tag                         | `v2.12.13`                       |
| `argocd_admin_password`    | ArgoCD admin password (bcrypt) (vault)   | Sensitive (vault)                |

### Loki
| Variable                 | Description                              | Default Value                     |
|-------------------------|------------------------------------------|-----------------------------------|
| `loki.pvc_capacity`     | Loki PVC size                            | `25Gi`                           |
| `loki.image.repo`       | Loki image repository                    | `grafana/loki`                   |
| `loki.image.tag`        | Loki image tag                           | `2.9.3`                          |

## Usage
1. Clone the role to your Ansible project or install it from Ansible Galaxy (once published).
2. Configure the necessary variables in `defaults/main/main.yml` and `defaults/main/vault.yml` files.
3. Create a playbook to use the role.

### Example Playbook
```yaml
---
- name: bootstrap kubernetes cluster
  hosts: localhost
  become: false
  gather_facts: false
  vars:
    proxy_env:
      http_proxy: http://<Your_Proxy_address>:<Port>
      https_proxy: http://<Your_Proxy_address>:<Port>
  roles:
    - bootstrap-kubernetes

```

4. Run the playbook:
```bash
ansible-playbook playbook.yml 
```

## Tags
The role supports the following tags for selective task execution:
- `bootstrap_kubernetes`: Runs all tasks.
- `setup_plugins`: Installs and configures Helm charts.
- `add_helm_repo`: Adds Helm repositories.
- `render_values_file`: Renders Helm values files from templates.
- `deploy_manifest`: Deploys raw Kubernetes manifests.
- `minio_configuration`: Configures MinIO (bucket, user, policy).

Example:
```bash
ansible-playbook playbook.yml --tags "setup_plugins,minio_configuration"
```

## Contributing
Contributions are welcome! Please submit issues or pull requests to the repository.

## 🔗 Stay connected with DockerMe! 🚀

**Subscribe to our channels, leave a comment, and drop a like to support our content. Your engagement helps us create more valuable DevOps and cloud content!** 🙌

[![Site](https://img.shields.io/badge/Dockerme.ir-0A66C2?style=for-the-badge&logo=docker&logoColor=white)](https://dockerme.ir/) [![linkedin](https://img.shields.io/badge/linkedin-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/ahmad-rafiee/) [![Telegram](https://img.shields.io/badge/telegram-0A66C2?style=for-the-badge&logo=telegram&logoColor=white)](https://t.me/dockerme) [![YouTube](https://img.shields.io/badge/youtube-FF0000?style=for-the-badge&logo=youtube&logoColor=white)](https://youtube.com/@dockerme) [![Instagram](https://img.shields.io/badge/instagram-FF0000?style=for-the-badge&logo=instagram&logoColor=white)](https://instagram.com/dockerme)