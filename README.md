# Ansible Collection for STIG Benchmarks

Documentation for the collection.

Contains the following roles

- RHEL7-STIG
- RHEL8-STIG
- RHEL9-STIG
- UBUNTU18-STIG
- UBUNTU20-STIG

## Installing this collection

You can install the galaxy galaxy collection with the Ansible Galaxy CLI:

```bash
ansible-galaxy collection install ansible_lockdown.stig_benchmarks
```

You can also include it in a `requirements.yml` file and install it with `ansible-galaxy collection install -r requirements.yml`, using the format:

```yaml
---

collections:
  - name: ansible_lockdown.stig_benchmarks
```

### More information

- [Ansible Using collections](https://docs.ansible.com/ansible/latest/user_guide/collections_using.html) for more details.

## Contributing to this collection

We welcome community contributions to this collection. If you find problems, please open an issue or create a PR against the [Ansible-Lockdown](https://github.com/ansible-lockdown/stig_collection).
More information about contributing can be found in our [Contribution Guidelines.](https://github.com/ansible-lockdown/stig_collection/blob/main/CONTRIBUTING.rst)
