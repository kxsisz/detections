# ⟡ ssh brute force — defense

> detection is only half the job — response is what protects the system

---

## ⟡ overview

This folder documents the **defensive countermeasures** applied against SSH brute force attacks in a controlled lab environment. It is the second part of the SSH brute force detection project and shifts focus from identifying the attack to **stopping it**.

**Environment:** Ubuntu (target) + Kali Linux (attacker) on VirtualBox
**Attack type:** SSH brute force — repeated failed login attempts from a single IP
**Detection source:** `/var/log/auth.log`
**Defense tooling:** Fail2Ban, UFW, SSH hardening

All measures demonstrated here are practical, command-driven, and validated in the lab.

---

## ⟡ threat summary

SSH brute force attacks involve an automated process systematically attempting username/password combinations against an SSH service until a valid pair is found.

**Characteristics observed in this lab:**

Repeated `Failed password` entries over a short time window, all originating from a single source IP (`192.168.1.105`). Multiple usernames were targeted — `root`, `admin`, and invalid users — with sequential source ports confirming automated tooling.

**MITRE ATT&CK mapping:**

| Technique | ID | Tactic | Notes |

| Brute Force | T1110 | Credential Access | Primary attack observed in this lab |
| Valid Accounts | T1078 | Defense Evasion / Persistence | Potential follow-on **only if** brute force succeeds |

> T1078 is not part of the brute force itself. It represents what an attacker may do after obtaining valid credentials — operating inside the system while appearing as a legitimate user. Preventing T1110 eliminates the path to T1078.

---

## ⟡ defense strategy

This lab applies a **layered defensive model**. No single control is sufficient on its own — each layer compensates for weaknesses in the others.

| Layer | Approach | Tooling |

| Detection | Monitor authentication logs for failure patterns | `/var/log/auth.log`, grep |
| Prevention | Harden SSH configuration to eliminate weak auth vectors | `sshd_config` |
| Automated Response | Ban IPs that exceed failure thresholds automatically | Fail2Ban |
| Access Control | Restrict SSH exposure at the network level | UFW, iptables |

The sequence matters: **detect the behavior → prevent the vector → automate the response → restrict the surface.**

---

## ⟡ defensive measures

### 1. fail2ban

Fail2Ban monitors log files and automatically bans IPs that exceed a defined failure threshold within a configured time window.

**Installation:**

```bash
sudo apt update && sudo apt install fail2ban -y
```

**Configure the SSH jail:**

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

Set the following under `[sshd]`:

```ini
[sshd]
enabled  = true
port     = ssh
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 5
bantime  = 3600
findtime = 600
```

| Parameter | Value | Meaning |
| `maxretry` | 5 | Trigger ban after 5 failures |
| `bantime` | 3600 | IP is blocked for 1 hour |
| `findtime` | 600 | Failure window spans 10 minutes |

**Enable and start the service:**

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

**How it works:**

Fail2Ban reads `/var/log/auth.log`, matches repeated `Failed password` entries against the `sshd` filter, and inserts a `REJECT` rule into `iptables` to block the offending IP automatically — no manual intervention required.

---

### 2. disable password authentication

Disabling password-based login forces all authentication through SSH keys, eliminating brute force as an attack vector entirely.

```bash
sudo nano /etc/ssh/sshd_config
```

Apply the following settings:

```
PasswordAuthentication no
PermitRootLogin no
ChallengeResponseAuthentication no
MaxAuthTries 3
LoginGraceTime 20
```

**Why these matter:**

`MaxAuthTries 3` limits the number of authentication attempts per connection. Combined with Fail2Ban, this means an attacker is banned before exhausting even the first connection cycle.

`LoginGraceTime 20` closes unauthenticated connections after 20 seconds. This limits the window available for slow or evasive brute force attempts and reduces resource exhaustion risk on the target host.

Restart SSH to apply changes (Ubuntu uses `ssh`, not `sshd`):

```bash
sudo systemctl restart ssh
```

> **Critical:** ensure SSH key access is working **before** disabling password authentication, or you will lock yourself out of the system.

---

### 3. ssh key-based authentication

**Generate a key pair on the admin machine:**

```bash
ssh-keygen -t ed25519 -C "lab-access"
```

**Copy the public key to the target:**

```bash
ssh-copy-id user@192.168.1.100
```

Or append manually on the target system:

```bash
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

With key-based authentication enabled and password authentication disabled, brute force attacks against the SSH service have no viable path forward.

---

### 4. firewall restrictions (ufw)

> **Order matters:** always add allow rules **before** enabling UFW to avoid locking yourself out of the system.

**Step 1 — Allow SSH before enabling the firewall:**

```bash
sudo ufw allow OpenSSH
```

**Step 2 — Set default policies:**

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

**Step 3 — Enable UFW:**

```bash
sudo ufw enable
```

**Verify the ruleset:**

```bash
sudo ufw status verbose
```

Expected output:

```
Status: active
To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW IN    Anywhere
```

---

### 5. restrict ssh access by ip

If SSH access is only required from a known admin IP, whitelist it explicitly and deny everything else.

**UFW approach:**

```bash
sudo ufw delete allow OpenSSH
sudo ufw allow from 192.168.1.50 to any port 22
```

**iptables approach:**

```bash
sudo iptables -A INPUT -p tcp --dport 22 -s 192.168.1.50 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j DROP
```

This ensures SSH is unreachable from unauthorized sources — even if credentials are somehow obtained through another vector.

---

### 6. change the default ssh port (optional)

Changing port 22 reduces exposure to automated scanners and opportunistic bots that target the default port indiscriminately.

```bash
sudo nano /etc/ssh/sshd_config
```

Change:

```
Port 22
```

To:

```
Port 2222
```

Update UFW to reflect the new port:

```bash
sudo ufw allow 2222/tcp
sudo ufw delete allow OpenSSH
sudo systemctl restart ssh
```

> This is a noise-reduction measure, not a security control. It is effective against mass scanners but will not stop a targeted attacker. Always combine with stronger controls.

---

## ⟡ validation

After applying defenses, confirm they are functioning as expected.

**Check Fail2Ban jail status:**

```bash
sudo fail2ban-client status sshd
```

Expected output after an attack attempt:

```
Status for the jail: sshd
|- Filter
|  |- Currently failed: 3
|  |- Total failed:     47
|  `- File list:        /var/log/auth.log
`- Actions
   |- Currently banned: 1
   |- Total banned:     1
   `- Banned IP list:   192.168.1.105
```

**Verify the iptables ban rule was inserted:**

```bash
sudo iptables -L f2b-sshd -n --line-numbers
```

Expected output:

```
Chain f2b-sshd (1 references)
num  target     prot opt source               destination
1    REJECT     all  --  192.168.1.105        0.0.0.0/0    reject-with icmp-port-unreachable
2    RETURN     all  --  0.0.0.0/0            0.0.0.0/0
```

**Simulate an attack from Kali and observe the result:**

```bash
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://192.168.1.100
```

After exceeding `maxretry`, the attacker's connection will be blocked. Depending on the Fail2Ban action and firewall configuration, this may manifest as a connection timeout, a TCP reset, or an explicit refusal — all confirming the ban is active.

```
[ERROR] could not connect to ssh://192.168.1.100:22 - Connection timed out
```

**Manually unban an IP during testing:**

```bash
sudo fail2ban-client set sshd unbanip 192.168.1.105
```

---

## ⟡ attack vs defense comparison

| No defenses active | Continuous login attempts proceed uninterrupted | Credential compromise possible |
| Fail2Ban only | Blocked after 5 failures within 10 minutes | IP banned for 1 hour |
| Password auth disabled | All attempts return permission denied | Brute force has no effect |
| IP whitelisting active | Connection times out or resets immediately | Attack surface eliminated |
| All measures combined | No viable authentication path available | Full mitigation achieved |

---

## ⟡ response recommendations

When brute force activity is confirmed in `/var/log/auth.log`, execute the following response sequence:

**1. Confirm the activity and identify the source:**

```bash
grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn
```

**2. Immediately ban the source IP:**

```bash
sudo fail2ban-client set sshd banip 192.168.1.105
```

**3. Check for any successful logins from the same IP:**

```bash
grep "Accepted" /var/log/auth.log | grep "192.168.1.105"
```

**4. Review recently authenticated sessions:**

```bash
last | head -20
```

**5. Audit for new or unexpected user accounts:**

```bash
awk -F: '$3 >= 1000 {print $1}' /etc/passwd
```

**6. If a successful login is confirmed — escalate immediately:**

Isolate the host from the network. Preserve log files for forensic analysis. Rotate all SSH keys and credentials on the affected system. Investigate for persistence mechanisms and lateral movement before returning the system to production.

---

## ⟡ best practices

| Use SSH keys, disable passwords | Eliminates brute force as a viable attack vector |
| Set `PermitRootLogin no` | Removes the highest-value credential target |
| Set `MaxAuthTries 3` | Limits attempts per connection, accelerates Fail2Ban banning |
| Set `LoginGraceTime 20` | Closes idle unauthenticated connections quickly |
| Enable Fail2Ban with tight `findtime` | Automates response without manual monitoring |
| Whitelist known admin IPs via UFW | Reduces SSH exposure to near zero |
| Keep OpenSSH updated | Prevents exploitation of known service vulnerabilities |
| Rotate SSH keys periodically | Limits blast radius of a compromised key |
| Monitor `/var/log/auth.log` continuously | Enables rapid detection before brute force succeeds |

**Consolidated `sshd_config` hardening block:**

```
PermitRootLogin no
PasswordAuthentication no
ChallengeResponseAuthentication no
MaxAuthTries 3
LoginGraceTime 20
AllowUsers youruser
```

---

## ⟡ conclusion

Detection without response is incomplete. This lab demonstrates that SSH brute force attacks — while noisy and detectable — can be fully mitigated through layered defensive controls applied at multiple levels of the stack.

The combination of **Fail2Ban** (automated banning), **key-based authentication** (elimination of the password attack surface), and **firewall restrictions** (network-level access control) creates a defense-in-depth posture that renders brute force attacks ineffective regardless of the wordlist or tooling used by the attacker.

> **Detection tells you the attack is happening. Defense stops it from succeeding. Both are required.**

This project reflects the core operational loop of a Blue Team analyst: observe, analyze, respond, harden — and document every step.

---
