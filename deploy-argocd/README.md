# ArgoCD Setup with Ansible

This repository contains an Ansible playbook to automate the installation and configuration of ArgoCD in a Kubernetes cluster on multiple master nodes at once.

## Overview

This playbook automates the complete setup of ArgoCD on your Kubernetes cluster, including:

- Creating the ArgoCD namespace
- Installing ArgoCD from the official manifests
- Exposing ArgoCD server via NodePort (accessible on port 30080)
- Retrieving and displaying the initial admin password
- Logging into ArgoCD CLI
- Adding a private GitHub repository to ArgoCD

## Prerequisites

- A running Kubernetes cluster with `kubectl` configured
- Python 3.x installed
- `uv` package manager
- `argocd` CLI tool installed at `/usr/local/bin/argocd`
- GitHub personal access token (for private repository access)

## Project Setup

1. **Initialize the Python environment:**
   ```bash
   cd deploy-argocd
   uv sync
   ```
   This creates a virtual environment and installs all required dependencies (primarily Ansible).

2. **Activate the virtual environment:**
   ```bash
   source .venv/bin/activate
   ```

## Configuration

### Hosts Inventory

Create or edit `ansible/hosts.ini` with your Kubernetes master node details:

```ini
[k8s_master]
192.168.0.146 ansible_user=host1 ansible_password=password ansible_become_pass=password
192.168.0.141 ansible_user=host2 ansible_password=password ansible_become_pass=password
```

Replace the IP addresses and usernames with your actual Kubernetes node information.

### Secrets Configuration

Create an encrypted secrets file to store sensitive information:

```bash
ansible-vault create asnible/secrets.yml
```

You'll be prompted to create a vault password. Remember this password as you'll need it to run the playbook.

**The `secrets.yml` file should contain:**

```yaml
github_token: "your_github_personal_access_token"
github_user: "your_github_username"
```

To generate a GitHub personal access token:
1. Go to GitHub Settings → Developer settings → Personal access tokens
2. Create a token with `repo` access
3. Copy the token and add it to your `secrets.yml`


### Repo Configurations

Edit configs/repos.yml file to store add all repos:

```yaml
- name: infra-repo
   url: https://github.com/<username>/<repo-name>
   username: "{{ github_username }}"
   password: "{{ github_token }}"
   project: default   
```

The username and token will be taken from secret.yml at runtime.


## Usage

Run the playbook with:

```bash
ansible-playbook -i ansible/hosts.ini ansible/setup-argocd.yml --ask-vault-pass
```

**Command breakdown:**
- `-i ansible/hosts.ini`: Specifies the inventory file
- `--ask-vault-pass`: Prompts for the Ansible Vault password to decrypt `secrets.yml`

## What the Playbook Does

1. **Validates Environment**: Checks that `kubectl` is installed and accessible

2. **Creates ArgoCD Namespace**: Sets up a dedicated `argocd` namespace in your cluster

3. **Installs ArgoCD**: Applies the official ArgoCD installation manifest from the stable release

4. **Exposes ArgoCD Server**: Patches the ArgoCD server service to use NodePort, making it accessible at `http://<node-ip>:30080`

5. **Retrieves Admin Credentials**: Fetches and decodes the initial admin password from the Kubernetes secret

6. **Logs into ArgoCD CLI**: Authenticates the ArgoCD CLI with the admin credentials

7. **Adds GitHub Repository**: Registers your private GitHub repository (`https://github.com/masterworks-engineering/bb-infra`) with ArgoCD using your credentials

## Accessing ArgoCD

After successful execution:

1. Access the ArgoCD UI at: `http://<your-node-ip>:30080`
2. Login credentials:
   - Username: `admin`
   - Password: (displayed in the playbook output)

## Customization

You can modify the following variables in the ansible/conigs/configs.yml:

- `argocd_namespace`: ArgoCD installation namespace (default: `argocd`)
- `argocd_nodeport`: External port for accessing ArgoCD (default: `30080`)
- `repo_url`: GitHub repository URL to add to ArgoCD

## Troubleshooting

- **kubectl not found**: Ensure kubectl is installed and in your PATH
- **Connection refused**: Verify your Kubernetes cluster is running and kubectl is properly configured
- **Authentication failed**: Check that your GitHub token has the correct permissions
- **NodePort not accessible**: Verify your firewall allows traffic on port 30080

## Security Notes

- The `secrets.yml` file is encrypted with Ansible Vault for security
- Never commit unencrypted secrets to version control
- Consider using a secrets management solution for production environments
- The initial admin password should be changed after first login

## Project Structure

```
bb-argocd/
├── ansible/
│   ├── hosts.ini          # Inventory file with K8s node details
│   └── setup-argocd.yml   # Main playbook
├── secrets.yml            # Encrypted vault file (create with ansible-vault)
├── pyproject.toml         # Python project configuration
├── uv.lock               # Dependency lock file
├── .venv/                # Virtual environment (created by uv sync)
└── README.md             # This file
```

## Quick Start

```bash
# 1. Setup environment
uv sync
source .venv/bin/activate

# 2. Create secrets file
ansible-vault create secrets.yml
# Add your github_token and github_user

# 3. Configure hosts.ini with your K8s nodes

# 4. Run the playbook
ansible-playbook -K -i ansible/hosts.ini ansible/setup-argocd.yml --ask-vault-pass
```