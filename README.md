# ⟡ detections

```bash
$ whoami
> blue team | detection engineering | log analysis

$ motto
> you can't detect what you don't understand
```

---

```bash
$ cat overview

this directory contains detection-focused labs built from real attack behavior.

each detection answers:

how does malicious activity actually appear in logs?
```

---

```bash
$ cat approach

1. simulate attacker behavior
2. collect logs
3. identify patterns & anomalies
4. validate findings

no siem. no shortcuts.
focus: understanding raw logs first.
```

---

```bash
$ tree

detections/
 └── <technique>/
      ├── detection/
      └── defense/
```

---

```bash
$ ls detections

ssh-brute-force
```

---

```bash
$ cat ssh-brute-force/info

log_source: /var/log/auth.log
focus: failed authentication patterns
behavior: repeated login attempts from single source
```

---

```bash
$ current_focus

- authentication logs
- abnormal vs normal behavior
- brute force patterns
- manual log analysis (grep, awk)
```

---

```bash
$ goal

build strong detection engineering fundamentals
through hands-on analysis of real attack behavior
```

---

```bash
$ echo "status"
> "You are not asked if you will be attacked — only if you will notice."
```
