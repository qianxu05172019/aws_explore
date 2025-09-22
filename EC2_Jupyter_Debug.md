# Debug Notes: EC2 + Jupyter Connection Issues on Corporate Network

## Background
- Corporate network blocks **outbound TCP/22**, making it impossible to connect to AWS EC2 over the default SSH port.
- Symptoms:
  - `ssh -i key.pem ubuntu@<IP>` → `Connection refused`
  - `nc -vz <IP> 22` → `Connection refused`
- Goal: Still be able to use JupyterLab on EC2 via SSH tunnel (`-L 8888:localhost:8888`) under corporate network restrictions.

---

## Problems Identified
1. **Initial symptom**
   - Public IP was correct, instance status was *running*, but local SSH always failed with "connection refused".
   - This indicated it was not a key issue, but a blocked port (22).

2. **Checking instance port listening**
   On EC2 (via Instance Connect):
   ```bash
   sudo ss -tlnp | grep ssh
   ```
   Output:
   ```
   0.0.0.0:22
   [::]:22
   ```
   → Only listening on port 22, nothing on 443.

3. **Root cause**
   - Ubuntu uses **systemd socket activation (`ssh.socket`)** by default, forcing sshd to bind only to port 22.
   - Even if `sshd_config` was modified, the change would not take effect.

---

## Solution (Final Correct Steps)

### 1. Modify sshd to listen on 443
```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak.$(date +%F-%H%M)
sudo sed -i 's/^#\?Port .*/Port 443/' /etc/ssh/sshd_config
grep -i '^Port' /etc/ssh/sshd_config
```
Expected output:
```
Port 443
```

### 2. Disable socket activation and enable ssh.service
```bash
sudo systemctl disable --now ssh.socket
sudo systemctl enable ssh
sudo systemctl restart ssh
sudo systemctl status ssh --no-pager
```

### 3. Verify listening port
```bash
sudo ss -tlnp | grep ssh
```
Correct result should include:
```
0.0.0.0:443
[::]:443
```

### 4. Configure Security Group
AWS Console → EC2 → Instance → **Security Groups → Inbound rules**  
- Add/verify rule:
  - **Type**: Custom TCP  
  - **Port range**: 443  
  - **Source**: My IP (recommended)

### 5. Test local connection
```bash
# Port connectivity test
nc -vz <Public_IP> 443

# Establish SSH tunnel
ssh -i ~/Downloads/squidpy-server-aws.pem -p 443 -N -L 8888:localhost:8888 ubuntu@<Public_IP>
```

### 6. Access JupyterLab
Open in browser:
```
http://localhost:8888
```
Use the token printed by JupyterLab on EC2.

---

✅ Final result:  
- With corporate network restrictions, `ssh -p 443` + `-L 8888:localhost:8888` works successfully.  
- JupyterLab is accessible at `localhost:8888` via SSH tunnel.
