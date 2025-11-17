# Setup Load Balancer on On-Premises Kubernetes

This project automates the setup of MetalLB and NGINX Ingress Controller on on-premises Kubernetes clusters using Ansible. It provides a complete solution for exposing services via LoadBalancer type services in bare-metal Kubernetes environments.

## Overview

This playbook installs and configures:
- **MetalLB**: A load balancer implementation for bare-metal Kubernetes clusters
- **NGINX Ingress Controller**: An Ingress controller for Kubernetes using NGINX as a reverse proxy

## Prerequisites

### On the Control Machine (where you run Ansible)

- Python 3.12 or higher
- Ansible 12.2.0 or higher
- SSH client installed (`/usr/bin/ssh`)
- SSH access to the Kubernetes master node(s)
- `uv` package manager (for dependency management)

### On the Kubernetes Master Node(s)

- Kubernetes cluster up and running
- `kubectl` configured and accessible
- SSH access enabled
- User with sudo privileges

## Installation

1. **Clone the repository** (if not already done):
   ```bash
   git clone <repository-url>
   cd setup-lb-onprem
   ```

2. **Install dependencies using uv**:
   ```bash
   uv sync
   ```

3. **Activate the virtual environment**:
   ```bash
   source .venv/bin/activate
   ```

## Configuration

### 1. Configure Inventory (`ansible/hosts.ini`)

Edit `ansible/hosts.ini` to specify your Kubernetes master node(s):

```ini
[k8s_master]
192.168.1.13 ansible_user=zafeer ansible_password=zafeer ansible_become_pass=zafeer
```

**Security Note**: For production use, consider:
- Using SSH keys instead of passwords
- Using Ansible Vault for sensitive credentials
- Using `ansible_ssh_private_key_file` for key-based authentication

### 2. Configure Variables (`ansible/configs/configs.yml`)

Edit `ansible/configs/configs.yml` to set your MetalLB IP range and versions:

```yaml
config:
  ip_range: "192.168.0.241"  # IP address or range for MetalLB (e.g., "192.168.0.240-192.168.0.250")
  metallb_version: "v0.14.8"
  ingress_version: "v1.11.0"
```

**Important**: 
- The `ip_range` should be an available IP address or range in your network
- Ensure the IP range is not used by other devices
- For a single IP, use format: `"192.168.0.241"`
- For a range, use format: `"192.168.0.240-192.168.0.250"`

### 3. Create Missing Variable Files (if needed)

The playbook references these files in `vars_files`:
- `ansible/secret.yml` (optional - create if needed)
- `ansible/configs/repos.yml` (optional - create if needed)

If you don't need these files, you can remove their references from `ansible/setup-lb.yml`.

## Usage

### Running the Playbook

From the project root directory:

```bash
ansible-playbook -i ansible/hosts.ini ansible/setup-lb.yml
```

Or from the `ansible` directory:

```bash
cd ansible
ansible-playbook -i hosts.ini setup-lb.yml
```

### What the Playbook Does

1. **Creates MetalLB namespace** (`metallb-system`)
2. **Installs MetalLB components** using the specified version
3. **Waits for MetalLB controller** to be ready
4. **Creates MetalLB IPAddressPool** with your configured IP range
5. **Creates MetalLB L2Advertisement** for Layer 2 mode
6. **Installs NGINX Ingress Controller** (bare-metal deployment)
7. **Waits for NGINX Ingress** to be ready
8. **Patches NGINX service** to use LoadBalancer type

### Verifying the Installation

After the playbook completes successfully, verify the setup:

```bash
# Check MetalLB pods
kubectl get pods -n metallb-system

# Check NGINX Ingress pods
kubectl get pods -n ingress-nginx

# Check if NGINX service got an external IP
kubectl get svc -n ingress-nginx ingress-nginx-controller
```

You should see an `EXTERNAL-IP` assigned to the `ingress-nginx-controller` service.

## Example: Deploying a Test Application

The `configs/` directory contains example Kubernetes manifests:

### 1. Deploy a test application:

```bash
kubectl apply -f configs/dep-test.yml
```

### 2. Create a service:

```bash
kubectl apply -f configs/nginx-svc.yml
```

### 3. Create an ingress rule:

```bash
kubectl apply -f configs/ingress-rule.yml
```

### 4. Access your application:

Get the external IP of the ingress controller:
```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller
```

Then access your application at: `http://<EXTERNAL-IP>/app`

## Project Structure

```
setup-lb-onprem/
├── ansible/
│   ├── ansible.cfg          # Ansible configuration
│   ├── hosts.ini            # Inventory file (configure your hosts here)
│   ├── setup-lb.yml         # Main playbook
│   └── configs/
│       └── configs.yml      # Configuration variables
├── configs/                 # Example Kubernetes manifests
│   ├── dep-test.yml        # Test deployment
│   ├── nginx-svc.yml       # NGINX service
│   ├── ingress-rule.yml    # Ingress rule example
│   └── metallb-config.yml  # MetalLB config example
├── main.py                 # Python entry point (placeholder)
├── pyproject.toml          # Python project configuration
├── uv.lock                 # Dependency lock file
└── README.md              # This file
```

## Troubleshooting

### SSH Connection Issues

If you encounter SSH errors:
- Ensure SSH is installed: `which ssh`
- Verify SSH access: `ssh ansible_user@<host_ip>`
- Check the `ansible.cfg` file has the correct SSH path

### MetalLB Not Assigning IPs

- Verify the IP range is available in your network
- Check MetalLB logs: `kubectl logs -n metallb-system -l app=metallb`
- Ensure your network supports Layer 2 mode (or configure BGP if needed)

### NGINX Ingress Not Getting External IP

- Verify MetalLB is running: `kubectl get pods -n metallb-system`
- Check service type: `kubectl get svc -n ingress-nginx ingress-nginx-controller`
- Ensure the service is patched to LoadBalancer type

### Playbook Fails on Variable Files

If you get errors about missing `secret.yml` or `repos.yml`:
- Either create these files in the `ansible/` directory
- Or remove their references from the `vars_files` section in `setup-lb.yml`

## Configuration Reference

### MetalLB Versions

Check available versions at: https://github.com/metallb/metallb/releases

### NGINX Ingress Versions

Check available versions at: https://github.com/kubernetes/ingress-nginx/releases

## Security Considerations

1. **Credentials**: Never commit passwords or sensitive data to version control
2. **SSH Keys**: Use SSH key-based authentication instead of passwords
3. **Ansible Vault**: Use Ansible Vault for encrypting sensitive variables
4. **Network Security**: Ensure MetalLB IP ranges are properly secured

## License

[Add your license here]

## Contributing

[Add contribution guidelines if applicable]

