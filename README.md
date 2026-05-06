# Debian VPS Bootstrap

Ansible bootstrap for a Debian VPS. It updates the system, configures DNS and
time sync, limits logs, creates a `deploy` user with passwordless sudo, hardens
SSH on port `22024`, enables UFW and Fail2ban, installs Docker from Docker's
official Debian repository, configures Docker log rotation, and logs in to
`ghcr.io`.

## 1. Configure inventory

Edit `inventory/hosts.yml` and replace the example IP:

```yaml
all:
  children:
    vps:
      hosts:
        your-vps:
          ansible_host: 127.0.0.1
```

## 2. Add SSH keys

Edit `group_vars/all.yml` and fill `authorized_keys` with real public keys:

```yaml
authorized_keys:
  - "ssh-ed25519 AAAA... vadim@example"
```

The playbook refuses to apply SSH hardening while this list is empty.

## 3. Create encrypted Vault secrets

Create the real Vault file:

```bash
ansible-vault create group_vars/vault.yml
```

Use this content inside the encrypted file:

```yaml
vault_ghcr_username: your-github-username
vault_ghcr_token: ghp_or_github_pat_token_here
```

`group_vars/vault.yml` is ignored by Git. A non-secret example is available at
`group_vars/vault.yml.example`.

## 4. First run on a fresh VPS

Fresh Debian VPS hosts usually start as `root` on SSH port `22`:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/bootstrap.yml -u root -e ansible_port=22 --ask-vault-pass
```

If the first connection uses an SSH password, add the server fingerprint to
`known_hosts` before running Ansible:

```bash
ssh-keyscan -H YOUR_SERVER >> ~/.ssh/known_hosts
```

Then run with password prompting:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/bootstrap.yml -u root -e ansible_port=22 --ask-pass --ask-vault-pass
```

## 5. Subsequent runs

After bootstrap, connect as `deploy` on SSH port `22024`:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/bootstrap.yml -u deploy -e ansible_port=22024 --ask-vault-pass --become
```

## What gets configured

- APT cache update and package upgrade, equivalent to `apt update && apt upgrade -y`.
- Swap file at `/swapfile`, enabled by default at `2048M` with `vm.swappiness=10`.
- Redis-friendly memory tuning: `vm.overcommit_memory=1` and Transparent Huge Pages disabled.
- Unattended security updates with automatic reboot disabled.
- Provider-independent DNS through `systemd-resolved` with Quad9, Google, and Cloudflare resolvers.
- Time sync through `systemd-timesyncd`.
- systemd journal cap at `100M`.
- `deploy` user with passwordless sudo through `/etc/sudoers.d/deploy`.
- SSH public-key login only, no root login, `AllowUsers deploy`, port `22024`.
- UFW defaults: deny incoming, allow outgoing, and allow only `22024/tcp`, `80/tcp`, `443/tcp`.
- Fail2ban `sshd` jail: 3 failed attempts in 10 minutes means a 24 hour ban.
- Docker Engine and Docker Compose plugin from Docker's official Debian APT repository.
- Docker `local` logging driver with `20m` x `5` rotation, about `100M` per container.
- `docker login ghcr.io` for the `deploy` user using Ansible Vault credentials.

## Docker and UFW caveat

UFW protects host ports, but Docker-published container ports can bypass UFW
filtering because Docker manages its own firewall rules. Treat published Docker
ports as intentionally public unless you add a stricter Docker firewall strategy
through the `DOCKER-USER` chain or equivalent provider-level firewall rules.

## Verification checklist

After bootstrap, verify:

```bash
ssh -p 22024 deploy@YOUR_SERVER
sudo -n true
systemctl is-active systemd-resolved systemd-timesyncd fail2ban docker
resolvectl dns
journalctl --disk-usage
swapon --show
sysctl vm.swappiness
sysctl vm.overcommit_memory
cat /sys/kernel/mm/transparent_hugepage/enabled
cat /sys/kernel/mm/transparent_hugepage/defrag
sudo ufw status verbose
sudo fail2ban-client status sshd
docker info --format '{{ .LoggingDriver }}'
docker run --rm hello-world
```

Password SSH login and root SSH login should be rejected.
