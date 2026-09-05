# Active Directory - Linux Entegrasyonu ve Ansible Otomasyonu

Bu çalışma, Active Directory ortamının Ansible ile kurulmasını ve Rocky Linux sunucuların `realmd`, SSSD ve Kerberos kullanılarak Active Directory domain'ine entegre edilmesini otomatikleştirir.

## Ortam

| Sunucu | Rol | IP |
|---|---|---|
| CASE-ANSIBLE01 | Ansible Control Node | - |
| CASE-DC01 | Windows Server / AD DS / DNS | 192.168.1.140 |
| CASE-LINUX01 | Rocky Linux | 192.168.1.141 |
| CASE-LINUX02 | Rocky Linux | 192.168.1.142 |

Domain: `casestudy.local`

## Mimari

```text
                         CASE-DC01
                     AD DS / DNS / KDC
                      192.168.1.140
                             |
                    LDAP / Kerberos / DNS
                             |
                 +-----------+-----------+
                 |                       |
          CASE-LINUX01              CASE-LINUX02
         192.168.1.141             192.168.1.142
          realmd/SSSD               realmd/SSSD

                    CASE-ANSIBLE01
                   /              \
                WinRM             SSH
                  |                |
               Windows           Linux
```

## Otomasyon Kapsamı

Ansible ile aşağıdaki işlemler otomatikleştirilmiştir:

- AD DS kurulumu ve `casestudy.local` forest/domain oluşturulması
- Gerekli OU'ların oluşturulması
- Domain user ve `linux-admins` / `linux-sudoers` security group'larının oluşturulması
- Linux sunuculara `realmd`, SSSD, Kerberos ve gerekli AD integration paketlerinin kurulması
- Linux sunucuların Active Directory domain'ine join edilmesi
- AD security group üzerinden Linux sudo yetkilendirmesi
- Secret bilgilerinin Ansible Vault ile korunması
- Ansible tarafından AD group membership değiştirildiğinde SSSD'nin kontrollü olarak restart edilmesi

## Repository Yapısı
```text
.
├── ansible.cfg
├── inventory.ini
├── site.yml
├── install_ad.yml
├── join_linux.yml
├── group_vars/
│   ├── all/vault.yml
│   ├── linux.yml
│   └── windows.yml
└── roles/
    ├── windows_ad/
    │   └── tasks/main.yml
    └── linux_join/
        └── tasks/main.yml


```


Windows ve Linux sistemler aynı inventory içerisinde yönetilmektedir. Platform-specific değişkenler `group_vars` altında ayrılmıştır.

## Çalıştırma

Tüm ortam:

```bash
ansible-playbook site.yml --ask-pass --ask-become-pass --ask-vault-pass
```

Sadece Active Directory:

```bash
ansible-playbook install_ad.yml --ask-pass --ask-vault-pass
```

Sadece Linux integration:

```bash
ansible-playbook join_linux.yml --ask-pass --ask-become-pass --ask-vault-pass
```

Playbook'lar idempotent olacak şekilde tasarlanmıştır. Ortam desired state durumundaysa `site.yml` tekrar çalıştırıldığında değişiklik yapılmaması beklenir.

## Doğrulama

AD user ve group çözümleme:

```bash
id 'selim.atmaca@casestudy.local'
```

Kerberos user TGT:

```bash
kinit selim.atmaca@CASESTUDY.LOCAL
klist
```

Machine keytab:

```bash
klist -k /etc/krb5.keytab
kinit -k 'CASE-LINUX01$@CASESTUDY.LOCAL'
klist
```

AD domain user ile her iki Linux sunucuya SSH login ve `linux-sudoers` group membership üzerinden sudo erişimi doğrulanmıştır.

## Tasarım Notları

- Ansible, agentless olması ve aynı Control Node üzerinden hem Windows hem Linux sistemleri yönetebilmesi nedeniyle tercih edilmiştir.
- Linux identity ve authentication akışı NSS/PAM -> SSSD -> Active Directory şeklindedir.
- `realmd/adcli`, domain join işlemi ile birlikte machine account, Kerberos keytab ve temel SSSD/Kerberos configuration işlemlerini yönetmektedir.
- Active Directory merkezi user/group membership kaynağıdır. Linux sudoers policy ise Ansible tarafından dağıtılmaktadır. Standard Windows GPO, Linux `/etc/sudoers` dosyalarını doğrudan yönetmediği için bu işlem GPO yerine Ansible ile gerçekleştirilmiştir.
- Lab ortamında tek Domain Controller ve basitlik amacıyla WinRM/HTTP kullanılmaktadır. Production ortamında multiple Domain Controller/DNS, secure WinRM, kontrollü patching, monitoring ve test edilmiş AD backup/DR prosedürleri kullanılmalıdır.
