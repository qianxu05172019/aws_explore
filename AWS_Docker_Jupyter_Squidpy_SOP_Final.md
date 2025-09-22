# 🚀 AWS + Docker + JupyterLab (Squidpy) Setup Guide

## 🔍 Background
When trying to connect via **AWS Console → EC2 Instance Connect**, the connection failed because the SSH server was modified to listen on port **443** (corporate network blocks port 22). AWS Instance Connect can only use port 22, so it failed.

---

## 🛠️ Solutions Implemented

1. **Enable SSH on both 22 and 443**
   - Updated `/etc/ssh/sshd_config`:
     ```bash
     Port 22
     Port 443
     ```
   - Restarted ssh:
     ```bash
     sudo systemctl restart ssh
     ```
   - This way AWS Console still works via 22, while local connections from restricted networks can use 443.

2. **SSH via 443 locally**
   ```bash
   ssh -i ~/Downloads/squidpy-server-aws.pem -p 443 ubuntu@<Public_IP>
   ```

3. **Persistent Storage with EBS**
   - Created and attached a new 100 GiB EBS volume.
   - Formatted and mounted to `/mnt/data`:
     ```bash
     sudo mkfs -t ext4 /dev/nvme1n1
     sudo mkdir -p /mnt/data
     sudo mount /dev/nvme1n1 /mnt/data
     sudo chown ubuntu:ubuntu /mnt/data
     ```
   - All project data now lives in `/mnt/data`, safe across Stop/Start cycles.

4. **Bind EBS Volume into Docker**
   - Mounted the EBS volume inside Docker:
     ```
     -v /mnt/data:/workspace
     ```
   - All files saved in Jupyter’s `/workspace` go directly into persistent EBS storage.

5. **Custom Docker Image with JupyterLab**
   - Created a `Dockerfile`:
     ```dockerfile
     FROM hubmap/spatial-transcriptomics-squidpy:latest
     RUN pip3 install --no-cache-dir jupyterlab ipywidgets
     RUN mkdir -p /workspace
     WORKDIR /workspace
     EXPOSE 8888
     CMD ["jupyter", "lab", "--ip=0.0.0.0", "--port=8888",
          "--no-browser", "--allow-root", "--notebook-dir=/workspace"]
     ```
   - Built the image:
     ```bash
     docker build -t my-squidpy-jlab:latest .
     ```
   - Ran the container:
     ```bash
     docker run -d        -p 8888:8888        -v /mnt/data:/workspace        --name squidpy        --restart unless-stopped        my-squidpy-jlab:latest
     ```

---

## 📌 Final SOP (Daily Workflow)

### **Step 1 — SSH into EC2**
```bash
ssh -i ~/Downloads/squidpy-server-aws.pem -p 443 ubuntu@<Public_IP>
```

### **Step 2 — Start the container**
```bash
docker start squidpy
```

### **Step 3 — Open SSH tunnel locally**
On your laptop (new terminal):
```bash
ssh -i ~/Downloads/squidpy-server-aws.pem -p 443 -N -L 8888:localhost:8888 ubuntu@<Public_IP>
```

### **Step 4 — Open JupyterLab in browser**
Visit:
```
http://localhost:8888
```
Get the token from container logs:
```bash
docker logs squidpy | grep -i token
```

### **Step 5 — Work in persistent volume**
- Save all notebooks and data in `/workspace` (inside JupyterLab).  
- This maps to `/mnt/data` on the EC2 host, backed by EBS (safe across Stop/Start).  

---


---


---

## 📂 Mount EBS Data Volume

After each `Stop` / `Start` cycle of your EC2 instance, make sure the data volume is mounted to `/mnt/data` before starting Docker:

```bash
# Check available disks and filesystems
lsblk -f

# Create the mount point (only first time)
sudo mkdir -p /mnt/data

# Mount the data volume (adjust device name if needed)
sudo mount /dev/nvme1n1 /mnt/data

# Ensure ubuntu user has access
sudo chown ubuntu:ubuntu /mnt/data

# Verify files are visible
ls -al /mnt/data | head
```

👉 If you want the mount to persist automatically after Stop/Start, add the UUID of `/dev/nvme1n1` into `/etc/fstab`.


## ♻️ Re-create container (only if the bind mount is wrong or missing)

If after a Stop/Start cycle your Jupyter `/workspace` does **not** show the content of the EBS volume (`/mnt/data` on the host),
recreate the container to ensure the bind mount `-v /mnt/data:/workspace` is present:

```bash
docker stop squidpy
docker rm squidpy

docker run -d   -p 8888:8888   -v /mnt/data:/workspace   --name squidpy   --restart unless-stopped   my-squidpy-jlab:latest
```

After this, open the SSH tunnel on your laptop and visit `http://localhost:8888` as usual.


## ✅ Achievements
- Solved corporate firewall restrictions by running SSH on port 443.  
- Solved data loss problem with persistent EBS volume.  
- Solved repeated Jupyter installation by baking it into a custom Docker image.  
- Daily workflow simplified to: **Login → Start container → Open tunnel → Access Jupyter**.  
