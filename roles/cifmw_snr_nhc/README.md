# cifmw_snr_nhc

Apply Self Node Remediation and Node Health Check Custom Resources on OpenShift.

## Overview

This Ansible role automates the deployment and configuration of:
- **Self Node Remediation (SNR)** - Automatically remediates unhealthy nodes
- **Node Health Check (NHC)** - Monitors node health and triggers remediation

The role creates the necessary operators, subscriptions, and custom resources to enable automatic node remediation in OpenShift clusters.

## Privilege escalation

None - all actions use the provided kubeconfig and require no additional host privileges.

## Parameters

* `cifmw_snr_nhc_kubeconfig`: (String) Path to the kubeconfig file.
* `cifmw_snr_nhc_kubeadmin_password_file`: (String) Path to the kubeadmin password file.
* `cifmw_snr_nhc_namespace`: (String) Namespace used for SNR and NHC resources. Default: `openshift-workload-availability`
* `cifmw_snr_nhc_cleanup_before_install`: (Boolean) Whether to clean up existing installation before installing. Default: `false`
* `cifmw_snr_nhc_cleanup_namespace`: (Boolean) Whether to remove the namespace during cleanup. Default: `false`

## Role Tasks

The role performs the following tasks in sequence:

1. **Create Namespace** - Creates the target namespace if it doesn't exist
2. **Create OperatorGroup** - Sets up the OperatorGroup for operator deployment
3. **Create SNR Subscription** - Deploys the Self Node Remediation operator
4. **Wait for SNR Deployment** - Waits for the SNR operator to be ready
5. **Create NHC Subscription** - Deploys the Node Health Check operator
6. **Wait for CSV** - Waits for the ClusterServiceVersion to be ready
7. **Create NHC CR** - Creates the NodeHealthCheck custom resource

## Examples

### Basic Usage

```yaml
- name: Configure SNR and NHC
  hosts: masters
  roles:
    - role: cifmw_snr_nhc
      cifmw_snr_nhc_kubeconfig: "/home/zuul/.kube/config"
      cifmw_snr_nhc_kubeadmin_password_file: "/home/zuul/.kube/kubeadmin-password"
      cifmw_snr_nhc_namespace: openshift-workload-availability
```

### Custom Namespace

```yaml
- name: Configure SNR and NHC in custom namespace
  hosts: masters
  roles:
    - role: cifmw_snr_nhc
      cifmw_snr_nhc_kubeconfig: "/path/to/kubeconfig"
      cifmw_snr_nhc_kubeadmin_password_file: "/path/to/password"
      cifmw_snr_nhc_namespace: custom-workload-namespace
```

## Cleanup Functionality

The role includes comprehensive cleanup functionality to remove all deployed resources.

### Cleanup Options

1. **Cleanup before install**: Automatically clean up existing resources before installing
2. **Standalone cleanup**: Run only cleanup tasks without installing
3. **Selective cleanup**: Clean up specific components (SNR, NHC, or both)
4. **Namespace preservation**: Option to keep or remove the namespace

### Cleanup Examples

#### Fresh Install with Cleanup

```yaml
- name: Clean install SNR and NHC
  hosts: masters
  roles:
    - role: cifmw_snr_nhc
      cifmw_snr_nhc_kubeconfig: "/path/to/kubeconfig"
      cifmw_snr_nhc_cleanup_before_install: true
      cifmw_snr_nhc_cleanup_namespace: false  # Keep namespace
```

#### Cleanup Only

```bash
# Run only cleanup tasks
ansible-playbook your-playbook.yml --tags cleanup

# Clean up specific components
ansible-playbook your-playbook.yml --tags snr-cleanup
ansible-playbook your-playbook.yml --tags nhc-cleanup

# Clean up including namespace removal
ansible-playbook your-playbook.yml --tags cleanup -e cifmw_snr_nhc_cleanup_namespace=true
```

#### Creating a Custom Playbook

```bash
# Create your own cleanup playbook
cat > my-cleanup.yml << 'EOF'
---
- name: SNR/NHC Cleanup
  hosts: localhost
  connection: local
  gather_facts: false
  vars:
    cifmw_snr_nhc_kubeconfig: "{{ lookup('env', 'KUBECONFIG') | default('~/.kube/config') }}"
    cifmw_snr_nhc_namespace: "openshift-workload-availability"
    cifmw_snr_nhc_cleanup_before_install: false
    cifmw_snr_nhc_cleanup_namespace: false
  tasks:
    - name: Include SNR/NHC role
      ansible.builtin.include_role:
        name: cifmw_snr_nhc
EOF

# Run cleanup only
ansible-playbook my-cleanup.yml --tags cleanup

# Run cleanup + fresh install
ansible-playbook my-cleanup.yml -e cifmw_snr_nhc_cleanup_before_install=true
```

### Cleanup Process

The cleanup process removes resources in the following order:

1. **NodeHealthCheck CR** - Removes the custom resource
2. **NHC Subscription** - Removes the Node Health Check operator subscription
3. **SNR Subscription** - Removes the Self Node Remediation operator subscription
4. **ClusterServiceVersions** - Waits for CSVs to be removed
5. **OperatorGroup** - Removes the operator group
6. **Namespace** - Optionally removes the namespace (controlled by `cifmw_snr_nhc_cleanup_namespace`)

### Cleanup Tags

Use tags to control which cleanup tasks run:

- `cleanup`: Run all cleanup tasks
- `snr-cleanup`: Clean up only SNR resources
- `nhc-cleanup`: Clean up only NHC resources  
- `namespace-cleanup`: Clean up namespace only

## Testing

This role includes comprehensive testing using Molecule and pytest. Tests validate:
- Role syntax and structure
- Individual task execution
- Idempotency
- Error handling
- Integration with Kubernetes APIs

### Quick Test Run

```bash
# Install test dependencies
pip install --user -r molecule/requirements.txt
ansible-galaxy collection install -r molecule/default/requirements.yml --force

# Run all tests
molecule test

# Run specific test phases
molecule converge  # Execute role
molecule verify    # Run verification tests
```

### Development Testing

```bash
# Quick development cycle
molecule converge  # Apply changes
molecule verify    # Check results
molecule destroy   # Clean up
```

For detailed testing information, see [TESTING.md](TESTING.md).

## Requirements

### System Requirements

- Python 3.9+
- Ansible 2.14+
- Access to OpenShift/Kubernetes cluster

### Ansible Collections

- `kubernetes.core` (>=6.0.0)
- `ansible.posix`
- `community.general`

### Python Dependencies

- `kubernetes` (>=24.0.0)
- `pyyaml` (>=6.0.0)
- `jsonpatch` (>=1.32)

## Development

### Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Run tests: `molecule test`
5. Submit a pull request

### Code Style

- Follow Ansible best practices
- Use descriptive task names
- Include proper error handling
- Test all changes with molecule

### Linting

```bash
# Run linting checks
ansible-lint tasks/main.yml
yamllint .
```

## Troubleshooting

### Common Issues

1. **Permission denied**: Ensure kubeconfig has proper permissions
2. **Namespace already exists**: Role handles existing namespaces gracefully
3. **Operator not ready**: Check cluster resources and connectivity

### Debug Mode

```bash
# Run with debug output
ansible-playbook -vvv your-playbook.yml
```

## License

This role is distributed under the terms of the Apache License 2.0.

## Support

For issues and questions:
- Check the [TESTING.md](TESTING.md) for testing guidance
- Review the troubleshooting section above
- Submit issues to the project repository
