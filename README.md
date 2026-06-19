# NexCore Infrastructure Automation

Ansible automation that turns two freshly installed RHEL 9 servers into NexCore d.o.o.'s internal infrastructure — DNS, web (MediaWiki + WordPress), database, shared storage, per-user automounted shares, internal mail, centralized logging, automated maintenance and backups — deployed from a single, idempotent `site.yml`.

Everything is configured through Ansible. SELinux stays in **enforcing** mode and **firewalld** stays active throughout; no managed-node configuration is done by hand.

---

## Architecture

| Host | Address | Roles it plays |
|------|---------|----------------|
| **workstation** | (control node) | Ansible control node only — never managed by playbooks |
| **servera** | 192.168.1.11 | Web server (Apache + PHP), authoritative internal DNS (BIND), mail server (Postfix), central log collector (rsyslog) |
| **serverb** | 192.168.1.12 | Database server (MariaDB), NFS server, log forwarder |

All managed servers run RHEL 9 (or compatible). The control node in this project runs AlmaLinux 9.

---

## What gets deployed

| Requirement | Summary |
|-------------|---------|
| BR-01 | MediaWiki at `wiki.nexcore.local` (DB on serverb) |
| BR-02 | WordPress at `www.nexcore.local` (DB on serverb) |
| BR-03 | Shared storage at `/mnt/shared` on a dedicated LVM volume, read-write for the `dev` group |
| BR-04 | Internal DNS for `nexcore.local`; all servers resolve through it, external names forwarded |
| BR-05 | User/group provisioning with SSH-key-only auth and `/etc/sudoers.d/`-based sudo |
| BR-06 | Per-user `~/Company_Share` automounted from serverb via AutoFS, surviving reboot |
| BR-07 | Internal Postfix mail for `nexcore.local` with local mailboxes |
| BR-08 | Security hardening: SELinux enforcing, firewalld active, per-service ports only |
| BR-09 | Centralized logging: serverb forwards all logs to servera over TCP 514 |
| BR-10 | Automated maintenance: daily health check (+ email), weekly cache cleanup, auto-patching with reboot-if-required |
| BR-11 | Log rotation of the health log (daily, 14 days, compressed) |
| Bonus | Dedicated backup role: nightly date-stamped database dumps to `/var/backups/nexcore/` |

---

## Project structure

```
nexcore-ansible/
├── ansible.cfg
├── inventory.ini
├── requirements.yml
├── site.yml
├── group_vars/
│   └── all.yml              # environment data (users, domain, DNS records, DB defs, ...)
├── host_vars/
│   ├── servera.yml          # servera IP
│   └── serverb.yml          # serverb IP, shared disk (by-id path)
└── roles/
    ├── users/               # accounts, SSH keys, sudoers
    ├── security/            # sshd hardening, SELinux enforcing, firewalld
    ├── dns/                 # BIND authoritative server (servera)
    ├── dns_client/          # NetworkManager resolver config (all hosts)
    ├── storage/             # LVM + XFS + NFS export (serverb)
    ├── automount_server/    # per-user NFS subdirs (serverb)
    ├── automount/           # AutoFS client config (all hosts)
    ├── database/            # MariaDB, app databases + scoped users (serverb)
    ├── web/                 # Apache, PHP, vhosts, SELinux, app deployment (servera)
    ├── email/               # Postfix (servera)
    ├── logging_server/      # rsyslog receiver (servera)
    ├── logging_client/      # rsyslog forwarder (serverb)
    ├── patching/            # health check, cron maintenance, patching, logrotate
    └── backup/              # nightly database dumps (serverb)
```

Roles are split server/client (e.g. `dns`/`dns_client`, `logging_server`/`logging_client`) where a single concern spans two host sets. The deliverable specified a minimum set of roles; these splits were added for clarity.

---

## Variable hierarchy

Variables are placed by asking whether a value is a property of the environment, a host, or a role:

- **`group_vars/all.yml`** — environment data true site-wide: the user list, the domain, DNS records, database definitions, web sites, and (redacted) database passwords.
- **`host_vars/servera.yml`, `host_vars/serverb.yml`** — per-host facts: each server's IP, and serverb's shared disk referenced by a stable `/dev/disk/by-id/` path.
- **`roles/<role>/defaults/main.yml`** — role mechanics: package names, service names, ports, paths, modes.

Cross-host values (e.g. a DNS record pointing at another host's IP) are resolved with `hostvars[]` at render time so the data stays portable.

> **Secrets:** database passwords are stored as plaintext `group_vars` for this lab. In production they would be encrypted with **Ansible Vault**.

---

## Prerequisites

On the **control node**:

- Ansible (ansible-core as shipped with RHEL/AlmaLinux 9)
- The required collections (installed via `requirements.yml`)

On the **managed servers** (one-time manual bootstrap — see below):

- RHEL 9 installed, reachable over SSH
- A subscription/repositories so `dnf` can install packages
- A dedicated spare disk on serverb for the shared LVM volume

---

## Usage

Clone and install collections:

```bash
git clone https://github.com/ffifee/Nexcore-Ansible-Infrastructure.git 
cd Nexcore-Ansible-Infrastructure
ansible-galaxy collection install -r requirements.yml
```

Confirm connectivity:

```bash
ansible all -m ping
```

Run the full deployment:

```bash
ansible-playbook site.yml
```

The playbook is **idempotent** — a second consecutive run reports zero changes.

---