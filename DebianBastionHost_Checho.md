# 2. Bastion Host (Debian 13)

[⬅️ Previous: pfSense Firewall](Firewall_Checho.md) · [Back to README](README.md) · [Next: Active Directory ➡️](ActiveDirectory_Checho.md)

The bastion host (`bastion01`) is the only administrative entry point over SSH. It's installed with the bare minimum and hardened with MFA (TOTP), SSH keys, `ufw`, `fail2ban` and AppArmor.

## Contents

- [Specs](#specs)
- [Installation](#installation)
- [Repository access with minimal pfSense rules](#repository-access-with-minimal-pfsense-rules)
- [Basic tools](#basic-tools)
- [AppArmor](#apparmor)
- [Hardening: ufw, MFA, SSH and fail2ban](#hardening-ufw-mfa-ssh-and-fail2ban)
- [Public key + MFA authentication](#public-key--mfa-authentication)
- [Future improvements](#future-improvements)

## Specs

| Parameter | Value |
| --------- | ----- |
| Hostname / FQDN | `bastion01` / `bastion01.lab.internal` |
| Operating system | Debian 13.4 (no desktop environment) |
| RAM / Disk | 2 GB / 20 GB |
| Network | Management – `10.0.40.0/28` (OPT3 interface in pfSense) |
| IP | `10.0.40.10` |
| Gateway / DNS | `10.0.40.1` (fallback DNS: `8.8.8.8`) |
| Admin VM (SSH client) | `10.0.40.11` |

> A bastion doesn't need many resources: it should run as little as possible to keep the attack surface small.

---

## Installation

### Network

There are no firewall rules for this subnet yet, so the VM won't get connectivity over DHCP. The network is configured manually during installation:

- Gateway and DNS: `10.0.40.1` (this will change to the Active Directory DNS once it's ready).
- Fallback DNS: `8.8.8.8`.
- Domain: `lab.internal` (local test environment), which makes the FQDN `bastion01.lab.internal`.

### Users

Leave the **`root` password empty**. Debian then locks direct root login and adds the initial user to the `sudo` group.

![Empty root password](images/bastion/01_root_sin_contrasena.png)

The installer then asks you to create an unprivileged user (with temporary privilege escalation through `sudo`):

![Creating the non-root user](images/bastion/02_usuario_no_root.png)

### Package manager and proxy

- Mirror: United States → `deb.debian.org`.
- Proxy: none for now. Later on, Squid, Burp Suite or mitmproxy could be deployed to practice web filtering, traffic inspection, malware sandboxing or to simulate a corporate environment.

So that the installer can download packages, a **temporary** pfSense rule allows all IPv4 traffic from the Management subnet out to the Internet. This rule is disabled afterwards (see below) and replaced with rules that only allow what's needed.

### Software selection

Select only **SSH server** and **standard system utilities**. OpenSSH provides remote administration, a secure shell, SSH tunneling and SCP/SFTP.

No desktop environment is installed: it's heavier, harder to harden and widens the attack surface.

![Minimal software selection](images/bastion/03_seleccion_software.png)

### Finish

Install the GRUB boot loader (this is the only operating system on the VM), reboot and log in with the user you created.

Services running after the minimal install:

```bash
systemctl list-units --type=service --state=running
```

![Running services after installation](images/bastion/04_servicios_activos.png)

### Check connectivity and update

```bash
ip a
ip route          # should show the configured default gateway
ping 10.0.40.1
ping 8.8.8.8
ping google.com

sudo apt update && sudo apt upgrade -y
```

After updating, **disable the temporary rule** that allowed full Internet access.

---

## Repository access with minimal pfSense rules

To install tools and get updates without opening up all traffic, an **FQDN alias** is used: pfSense periodically resolves the domains in the alias and updates the rules with their IPs.

### 1. Create the alias

In **Firewall → Aliases → IP → Add**:

| Field | Value |
| ----- | ----- |
| Name | `Bastion_Repo_Domains` |
| Type | Host(s) |
| Hosts | `deb.debian.org`, `security.debian.org` |

![FQDN alias with the Debian repositories](images/bastion/05_alias_fqdn_repos.png)

Save and apply the changes.

### 2. Take snapshots

Before creating the rules, take a snapshot of the firewall and the bastion so you can roll back.

![Firewall snapshot](images/bastion/06_snapshot_firewall.png)

![Bastion snapshot](images/bastion/07_snapshot_bastion.png)

### 3. Create the outbound rules

In **Firewall → Rules → OPT3** (Management interface):

| Action | Protocol | Source | Destination | Port | Description |
| ------ | -------- | ------ | ----------- | ---- | ----------- |
| Pass | TCP/UDP | `10.0.40.10` | any | 53 (DNS) | Allow DNS from Bastion Host |
| Pass | TCP | `10.0.40.10` | `Bastion_Repo_Domains` | 443 (HTTPS) | Allow Package Downloads (HTTPS) |
| Pass | TCP | `10.0.40.10` | `Bastion_Repo_Domains` | 80 (HTTP) | Allow Package Downloads (HTTP) |

> 💡 For HTTP and HTTPS you can create two separate rules or a *Port Alias* containing 80 and 443.

![DNS rule](images/bastion/08_regla_dns.png)

![HTTPS rule to the alias](images/bastion/09_regla_https.png)

![HTTP rule to the alias](images/bastion/10_regla_http.png)

Apply the changes. The result on OPT3 (it also includes an ICMP rule to ping from the bastion, and the temporary Internet rule, now disabled):

![Bastion outbound rules on OPT3](images/bastion/11_reglas_opt3_salida.png)

### 4. Verify from the bastion

```bash
sudo apt-get clean     # clears the package cache
sudo apt-get update    # should work if pfSense allows the traffic
```

---

## Basic tools

```bash
sudo apt install -y curl auditd fail2ban ufw libpam-google-authenticator \
  htop nmap tcpdump traceroute mtr-tiny
```

| Tool | Use |
| ---- | --- |
| `curl` | Test HTTP/HTTPS connectivity and APIs |
| `auditd` | Linux system auditing (`man auditd`) |
| `fail2ban` | Brute-force protection |
| `ufw` | Host firewall |
| `libpam-google-authenticator` | MFA (TOTP) at login (`man google-authenticator`) |
| `htop` | Interactive CPU, memory and process monitoring |
| `nmap` | Internal network discovery and auditing (can't reach outside the network) |
| `tcpdump` | Command-line traffic capture and analysis (similar to Wireshark) |
| `traceroute` / `mtr` | Connectivity troubleshooting |

The Wazuh agent is installed later (see [Wazuh](WazuhServer_Checho.md#install-the-agents)).

---

## AppArmor

Debian 13 ships with **AppArmor** enabled by default. AppArmor restricts what each program can do (which files it reads, which networks it reaches) using per-program profiles. If a tool fails for no obvious reason, check whether AppArmor is blocking it.

```bash
sudo aa-status | less     # loaded profiles
```

- **Complain:** the application runs normally, but violations are logged.
- **Enforce:** AppArmor actively blocks anything the profile doesn't allow.

To create or edit profiles:

```bash
sudo apt install apparmor-utils apparmor-profiles
```

Profiles live in `/etc/apparmor.d/` and are named by replacing the `/` in the executable's path with `.`. For example, the profile for `/usr/bin/ping` is `/etc/apparmor.d/usr.bin.ping`.

**Basic rules**

| Rule | Meaning |
| ---- | ------- |
| `r` | Read |
| `w` | Write / modify |
| `px` | Execute the file under its own profile |
| `/**` | The directory and all its subdirectories |
| `network inet stream,` | TCP connections (IPv4) |
| `network inet6 dgram,` | UDP connections (IPv6) |

```text
/var/log/nginx/*.log w,      # allows writing any .log file in that path
/etc/nginx/** r,             # allows reading everything under /etc/nginx/
```

**Recommended approach: learning mode**

1. Put the program in *complain* mode and use it normally.
2. Run `sudo aa-logprof`: it reads the logs and asks whether to allow or deny each action.
3. Reload the profile into the kernel:

   ```bash
   sudo apparmor_parser -r /etc/apparmor.d/profile.name
   ```

4. Switch back to strict mode:

   ```bash
   sudo aa-enforce /path/to/program
   ```

---

## Hardening: ufw, MFA, SSH and fail2ban

### 1. ufw

Allow SSH from the Management subnet and enable the firewall:

```bash
sudo ufw allow from 10.0.40.0/28
sudo ufw allow 22/tcp
sudo ufw enable
```

> 💡 `sudo ufw allow 22/tcp` allows SSH from any source. Once key-based access has been tested (below), you can restrict it to the Admin VM with `sudo ufw allow from 10.0.40.11 to any port 22 proto tcp` and remove the generic rules (`sudo ufw status numbered` → `sudo ufw delete <n>`).

### 2. Google Authenticator (PAM MFA)

As the user who will log in, run:

```bash
google-authenticator
```

Answer **Y** to time-based tokens, **Y** to disallow reusing the same token, **N** to widening the time window and **Y** to rate limiting. Scan the QR code with your authenticator app.

Edit the PAM configuration for SSH (PAM, *Pluggable Authentication Modules*, is the framework that centralizes authentication on Linux):

```bash
sudo nano /etc/pam.d/sshd
```

Add this line right **below** `@include common-auth`:

```text
auth required pam_google_authenticator.so nullok
```

- It goes after `common-auth` so the password is asked first and the TOTP code second.
- `nullok` prevents a lockout if a user hasn't set up MFA yet. **Remove it once testing is done.**

![PAM configuration for SSH](images/bastion/12_pam_sshd.png)

> ⚠️ In the screenshot the module appears as `pam_google_authentication.so`. The correct module name is **`pam_google_authenticator.so`**.

### 3. SSH configuration (phase 1: password + TOTP)

```bash
sudo nano /etc/ssh/sshd_config
```

```text
PermitRootLogin no

PubkeyAuthentication yes
PasswordAuthentication yes

KbdInteractiveAuthentication yes
ChallengeResponseAuthentication yes   # older alias for KbdInteractiveAuthentication

UsePAM yes

AuthenticationMethods keyboard-interactive

X11Forwarding no
AllowAgentForwarding no
PermitTunnel no

MaxAuthTries 3
LoginGraceTime 30
```

Check the syntax (no output means it's fine) and restart:

```bash
sudo sshd -t
sudo systemctl restart ssh
```

### 4. fail2ban + ufw

> Useful resource (Spanish): <https://www.alcancelibre.org/manuales/configuracion-de-fail2ban>

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

In the `[DEFAULT]` section, set `ufw` as the ban action and whitelist the Management network:

```ini
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1 10.0.40.0/28
# How long an IP stays banned
bantime  = 1h
# Time window for counting failed attempts
findtime = 10m
# Failures allowed before a ban
maxretry = 3
banaction = ufw
banaction_allports = ufw
# action_ = ban the IP only (already the default)
action = %(action_)s

[sshd]
enabled = true
port    = ssh
```

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

Useful commands:

```bash
sudo fail2ban-client status sshd                    # jail status
sudo fail2ban-client set sshd unbanip <IP_ADDRESS>  # unban an IP
sudo tail -f /var/log/fail2ban.log                  # watch the log
```

> ⚠️ With MFA, failed TOTP attempts produce different log messages than a wrong password, so fail2ban's `sshd` filter may not catch them. After a failed MFA attempt, check the authentication log and confirm the filter matches the format. On Debian 13, SSH logs go to journald by default: `sudo journalctl -u ssh` (or `/var/log/auth.log` if `rsyslog` is installed).

### 5. Test from the Admin VM

An administrator VM (`10.0.40.11`) is created to connect to the bastion over SSH. When connecting, it should ask for the password and the Google Authenticator code.

> Because the Admin VM is on the **same subnet** as the bastion, SSH traffic doesn't go through pfSense: the connection works even without a firewall rule.

---

## Public key + MFA authentication

### 1. Create the key pair on the Admin VM

```bash
ssh-keygen -t ed25519 -a 100
```

Ed25519 is the modern SSH algorithm: smaller keys, faster, and excellent security. Accept the default path and **use a passphrase**.

### 2. Check the keys

```bash
ls -l ~/.ssh
```

The permissions should look like this:

```text
-rw------- id_ed25519
-rw-r--r-- id_ed25519.pub
```

### 3. Copy the public key to the bastion

```bash
ssh-copy-id sergio@10.0.40.10
```

It asks for the user's password on the bastion and takes care of creating `~/.ssh/authorized_keys` with the right permissions. On the bastion, verify:

```bash
cat ~/.ssh/authorized_keys     # should show: ssh-ed25519 AAAAC3...
```

### 4. Require key + MFA

On the bastion, edit `/etc/ssh/sshd_config` so it looks like this:

```text
PermitRootLogin no

PubkeyAuthentication yes
PasswordAuthentication no

KbdInteractiveAuthentication yes
UsePAM yes

AuthenticationMethods publickey,keyboard-interactive
```

### 5. Restart SSH

```bash
sudo sshd -t
sudo systemctl restart ssh
```

### 6. Test the authentication

**Without closing the current session**, open another terminal on the Admin VM:

```bash
ssh sergio@10.0.40.10
```

Expected flow:

```text
SSH key accepted
      ↓
Verification code:
      ↓
TOTP accepted
      ↓
Login successful
```

It may first ask for the key passphrase and then the Google code.

### 7. Confirm which key is used

```bash
ssh -v sergio@10.0.40.10
```

If you see `Offering public key:` and `Server accepts key`, the key is being used correctly.

### 8. Restrict SSH to the Admin VM

**In pfSense (OPT3):**

| Action | Protocol | Source | Destination | Port |
| ------ | -------- | ------ | ----------- | ---- |
| Pass | TCP | `10.0.40.11` | `10.0.40.10` | 22 |
| Block | TCP | OPT3 net | `10.0.40.10` | 22 |

![SSH rules to the bastion on OPT3](images/bastion/13_reglas_opt3_ssh_admin.png)

> ⚠️ These rules only filter traffic that **crosses** the firewall. As long as the Admin VM and the bastion are on the same subnet, the effective control is on the bastion itself (`ufw` and the `from=` restriction below).

**On the bastion**, limit where the key is accepted from:

```bash
nano ~/.ssh/authorized_keys
```

```text
ssh-ed25519 AAAA...
      ↓
from="10.0.40.11" ssh-ed25519 AAAA...
```

That way, even if someone steals the private key, they could only use it from `10.0.40.11`.

> 🔐 The **private** key must never be on the bastion. Only the **public** key goes there.

### 9. Tighten fail2ban

To catch brute force even if the Admin VM is compromised, remove the Management subnet from the whitelist in `/etc/fail2ban/jail.local`:

```ini
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1
```

```bash
sudo systemctl restart fail2ban
```

---

## Future improvements

- Move the Admin VM to a different subnet/VLAN so all SSH traffic crosses the firewall and can be filtered, logged and monitored, as in an enterprise architecture.
- Add a VPN for remote administrative access.

**Recommended reading**

- [SSH bastion hardening best practices – Teleport](https://goteleport.com/blog/security-hardening-ssh-bastion-best-practices/)
- [Weak Diffie-Hellman and the Logjam Attack](https://weakdh.org/)
- [CVE-2020-27347 – NVD](https://nvd.nist.gov/vuln/detail/CVE-2020-27347)
- Risks of unrestricted ICMP (to do: add a specific source)

[⬅️ Previous: pfSense Firewall](Firewall_Checho.md) · [Back to README](README.md) · [Next: Active Directory ➡️](ActiveDirectory_Checho.md)
