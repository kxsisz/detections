# ⟡ ssh brute force detection

> detection starts with understanding behavior

---

## ⟡ overview

This lab documents the simulation and analysis of an SSH brute force attack from a defensive (Blue Team) perspective.
The goal is to understand how brute force activity appears in authentication logs and how to identify key indicators of this behavior.

- **environment:** linux
- **log source:** `/var/log/auth.log`
- **focus:** log analysis & pattern recognition

---

## ⟡ scenario

A controlled lab environment was used to generate repeated failed SSH login attempts against a Linux host.

Observed behavior:

- multiple failed authentication attempts
- repeated attempts from a single IP address
- multiple targeted usernames (`root`, `admin`, invalid users)

All activity originated from the same source, simulating automated brute force behavior.

> no SIEM or external tools were used — analysis was performed directly on log data

---

## ⟡ log evidence

Example entries from `/var/log/auth.log`:

```
Failed password for root from 192.168.1.105 port 54321 ssh2
Failed password for admin from 192.168.1.105 port 54322 ssh2
Failed password for invalid user testuser from 192.168.1.105 port 54323 ssh2
```

Indicators observed:

- single source IP across all attempts
- multiple usernames targeted
- presence of `invalid user` entries
- high frequency of failed attempts
- sequential source ports (automation indicator)

---

## ⟡ detection

SSH brute force activity can be identified through the following patterns:

- high volume of `Failed password` entries in a short period
- repeated attempts from the same source IP
- multiple usernames being targeted
- `invalid user` entries indicating unknown account attempts
- rapid, consistent attempt frequency

These patterns distinguish brute force behavior from normal login activity.

---

## ⟡ investigation steps

**1. review recent authentication failures**
```bash
grep "Failed password" /var/log/auth.log
```

**2. count failed attempts per source IP**
```bash
grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn
```

**3. identify all targeted usernames**
```bash
grep "Failed password" /var/log/auth.log | awk '{print $9}' | sort | uniq -c | sort -rn
```

**4. isolate attempts against invalid users**
```bash
grep "Invalid user" /var/log/auth.log
```

**5. establish the time range of activity**
```bash
grep "Failed password" /var/log/auth.log | awk '{print $1, $2, $3}' | head -5
grep "Failed password" /var/log/auth.log | awk '{print $1, $2, $3}' | tail -5
```

---

## ⟡ conclusion

This lab demonstrates how SSH brute force activity leaves clear traces in `/var/log/auth.log`.

Through log analysis, it was possible to:

- identify the attack source (single IP)
- detect targeted accounts (including invalid users)
- understand frequency and behavior patterns

From a defensive perspective, this provides enough evidence to classify the activity and respond accordingly.

Possible responses:

- block source IP (firewall rules)
- implement automated protection (e.g., `fail2ban`)
