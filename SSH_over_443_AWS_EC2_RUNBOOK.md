# SSH over 443 to AWS EC2 (Production Runbook)

## When to Use
- Corporate network blocks outbound SSH (TCP/22)
- VS Code Remote-SSH or CLI SSH times out

## Goal
Enable SSH access via TCP/443 while keeping default TCP/22.

---

## Checklist (TL;DR)

- [ ] Access instance via **EC2 Instance Connect** (browser)
- [ ] Configure `sshd` to listen on **22 and 443**
- [ ] Restart `sshd`
- [ ] Open **443/tcp** in Security Group
- [ ] Verify listeners on the instance
- [ ] Connect locally using `-p 443`
- [ ] Configure VS Code Remote-SSH

---

## Step-by-Step

### 1) Access Instance (Browser)
AWS Console → EC2 → Instance → **Connect** → **EC2 Instance Connect**

---

### 2) Configure SSH Daemon (on EC2)
```bash
sudo vim /etc/ssh/sshd_config
```
Ensure:
```text
Port 22
Port 443
```

Restart:
```bash
sudo systemctl restart sshd
```

---

### 3) Verify Listening Ports
```bash
sudo ss -lntp | grep ssh
```
Expected:
```text
0.0.0.0:22
0.0.0.0:443
```

---

### 4) Security Group (Inbound Rules)
Add:
- **SSH** | TCP | **22** | `0.0.0.0/0`
- **Custom TCP** | TCP | **443** | `0.0.0.0/0`

---

### 5) Local Connection Test
```bash
ssh -p 443 -i ~/Downloads/your-key.pem ec2-user@<PUBLIC_IP>
```

---

### 6) VS Code Remote-SSH Config
`~/.ssh/config`
```text
Host aws443
    HostName <PUBLIC_IP>
    User ec2-user
    Port 443
    IdentityFile ~/Downloads/your-key.pem
```

Connect in VS Code:
```
Remote-SSH: Connect to Host → aws443
```

---

## Validation
- VS Code status bar shows: `SSH: <PUBLIC_IP>`
- `hostname` returns EC2 internal hostname

---

## Notes
- If still timing out, the corporate firewall may block non-HTTPS traffic even on 443.
- Consider **AWS SSM Session Manager** for environments that block all SSH.

---

## File
`SSH_over_443_AWS_EC2_RUNBOOK.md`
