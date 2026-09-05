# AD Linux Integration with Ansible

This project automates the deployment of an Active Directory environment and the integration of Rocky Linux servers with Active Directory using Ansible.

The project includes:

- Installation of Active Directory Domain Services and DNS
- Creation of the `casestudy.local` forest/domain
- Creation of Active Directory OUs, users and security groups
- Installation of required Linux AD integration packages
- Joining Rocky Linux servers to Active Directory
- AD-based Linux identity and authentication using SSSD
- Sudo access based on an Active Directory security group
- Secure password storage using Ansible Vault

## Environment

- CASE-ANSIBLE01 - Ansible control node
- CASE-DC01 - 192.168.1.140 - Active Directory / DNS
- CASE-LINUX01 - 192.168.1.141 - Rocky Linux
- CASE-LINUX02 - 192.168.1.142 - Rocky Linux

## Architecture

```text
                         casestudy.local
                               |
                        +-------------+
                        |  CASE-DC01  |
                        | AD DS + DNS |
                        |192.168.1.140|
                        +------+------+
                               |
                  DNS / LDAP / Kerberos
                               |
                    +----------+----------+
                    |                     |
             +------+-------+      +------+-------+
             | CASE-LINUX01 |      | CASE-LINUX02 |
             | Rocky Linux  |      | Rocky Linux  |
             |192.168.1.141 |      |192.168.1.142 |
             +--------------+      +--------------+

                    Ansible Control Node
                       CASE-ANSIBLE01
                              |
                    SSH ------+------ WinRM
```

Active Directory provides centralized identity, DNS and Kerberos authentication.

The Linux servers are joined to the `casestudy.local` domain using `realmd` and `adcli`. SSSD provides Active Directory identity and authentication services to Linux through NSS and PAM.

Ansible is executed from `CASE-ANSIBLE01`. Linux servers are managed over SSH, while the Windows server is managed over WinRM.

## Why Ansible

Ansible was selected because it is agentless and can manage both Windows and Linux systems from the same control node.

The project primarily uses declarative Ansible modules to describe the desired state of the environment. Examples include Windows features, Active Directory objects, Linux packages and sudoers configuration.

A small imperative step is used for the Linux domain join through the `realm join` command. Idempotency is maintained by checking for `/etc/sssd/sssd.conf` before executing the join.

Platform-specific variables are separated using `group_vars`, and sensitive credentials are stored with Ansible Vault.

## Playbooks

### Complete Deployment

The complete environment can be configured using:

```bash
ansible-playbook site.yml --ask-pass --ask-become-pass --ask-vault-pass
```

The `site.yml` playbook first configures Active Directory and then configures the Linux servers.

### Active Directory Only

```bash
ansible-playbook install_ad.yml --ask-pass --ask-vault-pass
```

### Linux Integration Only

```bash
ansible-playbook join_linux.yml --ask-pass --ask-become-pass --ask-vault-pass
```




## Linux and Active Directory Authentication Flow

Linux identity and authentication are provided through SSSD.

NSS is used for identity lookups such as users, groups, UID and GID information:

```text
id / getent
     |
     v
    NSS
     |
     v
    SSSD
     |
     v
Active Directory
```

PAM is used for authentication and session handling:

```text
SSH login
    |
    v
   PAM
    |
    v
 pam_sss
    |
    v
   SSSD
    |
    v
Active Directory
```

The system uses the SSSD authselect profile with automatic home directory creation enabled.

Active Directory users are configured to use fully qualified names such as:

```text
selim.atmaca@casestudy.local
```

SSSD uses the Active Directory provider for identity and access control. AD SID information is mapped to Linux UID and GID values using SSSD ID mapping.

## Kerberos and Keytab

Active Directory also acts as the Kerberos Key Distribution Center (KDC).

A user can obtain a Kerberos Ticket Granting Ticket (TGT) using:

```bash
kinit selim.atmaca@CASESTUDY.LOCAL
klist
```

The obtained TGT can later be used to request service tickets without repeatedly sending the user's password.

The Linux domain join also creates a machine account in Active Directory and stores its Kerberos keys in:

```text
/etc/krb5.keytab
```

The keytab principals can be inspected with:

```bash
klist -k /etc/krb5.keytab
```

Machine authentication was validated using the computer account without entering a password:

```bash
kinit -k 'CASE-LINUX01$@CASESTUDY.LOCAL'
klist
```

This confirms that the Linux host can authenticate to the Active Directory KDC using its keytab.

Kerberos ticket lifetime and renewal settings are defined in `/etc/krb5.conf`. The lab configuration allows renewable Kerberos tickets.

The base Kerberos, SSSD and keytab configuration is created as part of the `realmd` / `adcli` domain join process rather than being manually templated by Ansible. This avoids having two separate mechanisms managing the same domain integration configuration.

## AD Group Based Sudo Access

The Active Directory security group `linux-sudoers` is used to grant administrative privileges on the Linux servers.

Ansible manages the following sudoers policy:

```text
%linux-sudoers@casestudy.local ALL=(ALL) ALL
```

SSSD resolves the AD group membership and sudo uses that membership to authorize privileged commands.

The configuration was validated by logging in to the Linux servers with an Active Directory user and successfully using sudo.





## Idempotency

The Ansible roles are designed to be idempotent. Re-running the playbooks does not recreate existing Active Directory objects, reinstall existing packages, or rejoin Linux servers that are already domain members.

The Linux domain join uses the existence of `/etc/sssd/sssd.conf` to prevent repeated `realm join` operations.

Active Directory group membership tasks report whether a membership change was actually made. If Ansible changes the managed AD group memberships, `site.yml` restarts SSSD on the Linux servers so that the updated membership is immediately available.

If no membership change occurs, the SSSD restart is skipped.

Repeated test runs of `site.yml` completed with `changed=0` when the environment was already in the desired state.

## Troubleshooting and Operational Notes

### DNS

Active Directory DNS is essential for domain discovery and Kerberos operation. The Linux servers use CASE-DC01 as their DNS server.

AD service discovery was validated using the LDAP and Kerberos DNS SRV records:

```text
_ldap._tcp.casestudy.local
_kerberos._tcp.casestudy.local
```

Both records resolved to the domain controller during testing.

### SSSD Group Membership Cache

During testing, a newly changed Active Directory group membership was not immediately visible on a Linux server.

The AD membership was confirmed on the domain controller, but the Linux `id` command initially returned the previous group membership.

Running:

```bash
sss_cache -E
```

did not make the new membership immediately visible in this test.

Restarting SSSD:

```bash
systemctl restart sssd
```

caused the updated AD group membership to be resolved correctly.

For Ansible-managed membership changes, `site.yml` therefore restarts SSSD only when one of the managed AD group membership tasks reports a change. This avoids restarting SSSD unnecessarily on every playbook run.

### Time Synchronization

Kerberos authentication is time-sensitive. Significant clock differences between the Linux servers and the Active Directory domain controller can cause Kerberos authentication failures.

In production, all domain members and domain controllers should use a reliable and consistent time synchronization source.

### SELinux

SELinux should not normally be disabled as a solution to authentication problems. If SELinux blocks an operation, the relevant audit logs should be reviewed and the required policy or context should be corrected.

The lab keeps the configuration as close as possible to the operating system defaults rather than disabling security controls unnecessarily.














## Production Considerations

This project is intentionally designed as a small lab environment. A production deployment would require additional availability, security and operational controls.

### Active Directory High Availability

The lab contains a single domain controller. In production, at least two domain controllers with AD-integrated DNS should be deployed to avoid a single point of failure.

Linux clients can discover available domain controllers through Active Directory DNS SRV records rather than depending on a single statically configured Kerberos server.

Domain controllers should be placed according to the organization's site and network topology, and Active Directory replication should be monitored.

### Secure Ansible Connectivity

The lab uses WinRM over HTTP with Basic authentication for simplicity.

In production, Windows management should use encrypted WinRM communication, preferably HTTPS on port 5986 with trusted certificates.

SSH should use key-based authentication where possible instead of password-based authentication.

Ansible Vault is used in this project to protect sensitive variables. In a larger production environment, secrets could instead be integrated with a centralized secrets management platform.

### Patching

Operating system and security updates should be applied through a controlled patching process.

With multiple domain controllers, patching should be performed sequentially so that authentication and DNS services remain available while individual domain controllers are being maintained.

Linux servers should similarly be patched in controlled batches where service availability requirements exist.

### Monitoring

A production environment should monitor at least:

- Domain controller availability
- Active Directory replication health
- DNS service and DNS resolution
- Kerberos authentication failures
- System time synchronization
- SSSD service health on Linux servers
- Disk, CPU and memory utilization
- Authentication and security logs

Alerts should be generated for failures that can affect authentication or domain availability.

### Disaster Recovery

Active Directory should have tested system-state-aware backup and recovery procedures.

Multiple domain controllers provide service availability but do not replace backups. Recovery procedures should account for accidental deletion, directory corruption and complete site loss.

Ansible configuration should be stored in version control and backed up independently. Because the infrastructure configuration is maintained as code, the configuration of replacement Linux servers can be reproduced consistently.

Sensitive secrets and encryption keys must be backed up separately and securely.

### Linux Access Control

The lab grants sudo privileges through the `linux-sudoers` Active Directory security group.

For production environments, administrative access should follow least-privilege principles. Separate AD groups can be used for different server roles or privilege levels instead of granting the same administrative permissions across all Linux servers.

