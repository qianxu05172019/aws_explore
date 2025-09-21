# AWS_Squidpy_Journey

## 🚀 My First Attempt Running Squidpy on AWS – Full Exploration Log  

---

### 1. Launching the EC2 Instance & First SSH Attempt  
- Created a new **Ubuntu 22.04 LTS** EC2 instance from the AWS Console.  
- Downloaded the key pair file `squidpy-server-aws.pem`.  
- Tried to connect:  

```bash
ssh -i my-key.pem -N -C -L 8888:localhost:8888 ubuntu@<EC2_Public_IP>
```


### 2. Fixing Private Key Permissions  
After pointing to the correct file, got another error:  

```bash
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Permissions 0644 for 'squidpy-server-aws.pem' are too open.
```

👉 Solution: restrict permissions  
```bash
chmod 400 squidpy-server-aws.pem
```

Reconnected successfully:  
```bash
ssh -i squidpy-server-aws.pem -N -C -L 8888:localhost:8888 ubuntu@<EC2_Public_IP>
```

---

### 3. Setting up Docker on EC2  
After logging in, installed Docker:  
```bash
sudo apt update && sudo apt install -y docker.io
```

Added `ubuntu` user to the `docker` group:  
```bash
sudo usermod -aG docker ubuntu
exit   # re-login required
```

---

### 4. Pulling the Squidpy Image  
```bash
docker pull hubmap/spatial-transcriptomics-squidpy
```

Check image:  
```bash
docker images
```

---

### 5. Running the Container  
First attempt:  
```bash
docker run -it -p 8888:8888 hubmap/spatial-transcriptomics-squidpy
```

Got into the container shell:  
```
root@08ab723f1323:/opt#
```

---

### 6. Checking Python & Squidpy  
Tried:  
```bash
python -c "import squidpy, scanpy; print('squidpy', squidpy.__version__)"
```
Error: `bash: python: command not found`

👉 Fix: use `python3` instead  
```bash
python3 -c "import squidpy, scanpy; print('squidpy', squidpy.__version__)"
```

Output:  
```
squidpy 1.6.2
```

Confirmed Squidpy installed correctly.  

---

### 7. Launching JupyterLab  
# Make sure pip is available
apt update && apt install -y python3-pip

# Install JupyterLab
pip3 install jupyterlab

```bash
jupyter lab --ip=0.0.0.0 --port=8888 --no-browser --allow-root
```

Terminal output showed extensions loaded, with a note about running as root.  

---

### 8. Finding the EC2 Public IP  
From inside EC2:  
```bash
curl ifconfig.me
```

Got the public IP for accessing Jupyter.  

---

### 9. Public Access Too Slow → Using SSH Tunnel  
Tried:  
```
http://<EC2_Public_IP>:8888
```
Result: extremely slow.  

👉 Solution: use **SSH tunnel** from local machine:  
```bash
ssh -i squidpy-server-aws.pem -N -C -L 8888:localhost:8888 ubuntu@<EC2_Public_IP>
```

Then opened in browser:  
```
http://localhost:8888
```

Pasted the token, logged in successfully. Performance much smoother.  

---

## ✅ Final Summary
This first run-through included:  
- **SSH issues** → solved by fixing file path & permissions.  
- **Docker setup** → pulled and launched Squidpy image.  
- **Python mismatch** → switched from `python` to `python3`.  
- **JupyterLab launch syntax errors** → fixed with correct `--ip=...` formatting.  
- **Slow direct browser access** → solved by using SSH tunneling with compression.  

Result: **Squidpy 1.6.2 running on AWS via Docker, accessed smoothly in JupyterLab.** 🎉  
