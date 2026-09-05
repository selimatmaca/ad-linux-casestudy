# AD Linux Integration with Ansible

This project automates:

- Installation of Active Directory Domain Services
- Creation of the `casestudy.local` forest/domain
- Installation of required Linux AD integration packages
- Joining Rocky Linux servers to Active Directory
- Secure password storage using Ansible Vault

## Environment

- CASE-DC01 - 192.168.1.140
- CASE-LINUX01 - 192.168.1.141
- CASE-LINUX02 - 192.168.1.142

## Playbooks

### Install Active Directory

```bash
ansible-playbook install_ad.yml --ask-pass --ask-vault-pass
