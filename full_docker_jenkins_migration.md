# 🧭 Migration Guide: Move Docker & Jenkins Data from SSD to HDD (/data)

### 🧾 Overview
This document details every step performed to migrate **Docker** and **Jenkins** data
from the SSD (`/` and `/home`) to the HDD (`/dev/sda1` mounted at `/data`) on a RHEL-based system.

| Component | Old Path | New Path | Verified |
|------------|-----------|----------|-----------|
| Docker Data | `/var/lib/docker` | `/data/docker` | ✅ |
| Jenkins Home | `/home/narendra/Jenkins/jenkins_home` | `/data/Jenkins/jenkins_home` | ✅ |
| Mount Point | `/dev/sda1` | `/data` | ✅ |

---

## 🧩 Step 1 — Identify Disks
```bash
lsblk
df -h
```

Expected:
```
sda    931.5G  (unused, target HDD)
sdb    238.5G  (SSD, OS + /home + /boot)
```

---

## 💽 Step 2 — Partition and Format the HDD
```bash
sudo fdisk /dev/sda
```
Inside `fdisk`:
```
g        # create a new GPT partition table
n        # new partition
<Enter>  # accept defaults (full disk)
<Enter>
<Enter>
w        # write changes and exit
```
Format partition:
```bash
sudo mkfs.ext4 /dev/sda1
```

---

## 📂 Step 3 — Mount the HDD at /data
```bash
sudo mkdir -p /data
sudo mount /dev/sda1 /data
df -h /data
```

---

## 🔁 Step 4 — Add to /etc/fstab
Find UUID:
```bash
sudo blkid /dev/sda1
```
Edit `/etc/fstab`:
```
UUID=f08d72de-9d7d-422e-9c24-6a93145c6c5a /data ext4 defaults 0 0
```
Mount test:
```bash
sudo mount -a
```

---

## 🐳 Step 5 — Move Docker Data to HDD
```bash
sudo systemctl stop docker
sudo mkdir -p /data/docker
sudo rsync -aP /var/lib/docker/ /data/docker/
sudo mv /var/lib/docker /var/lib/docker.bak

sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<EOF
{
  "data-root": "/data/docker"
}
EOF

sudo systemctl daemon-reexec
sudo systemctl restart docker
docker info | grep "Docker Root Dir"
```

Expected:
```
Docker Root Dir: /data/docker
```

---

## ⚙️ Step 6 — Move Jenkins to HDD
```bash
sudo docker compose down
sudo mkdir -p /data/Jenkins/jenkins_home
sudo rsync -aHAX --info=progress2 /home/narendra/Jenkins/jenkins_home/ /data/Jenkins/jenkins_home/
sudo chown -R 1000:1000 /data/Jenkins/jenkins_home
```

---

## 🧾 Step 7 — Updated docker-compose.yml

Path: `/home/narendra/Jenkins/docker-compose.yml`
```yaml
services:
  jenkins:
    container_name: jenkins
    image: jenkins/jenkins:lts
    restart: unless-stopped
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - /data/Jenkins/jenkins_home:/var/jenkins_home:Z
    networks:
      - net

  agent1:
    image: jenkins/inbound-agent
    container_name: jenkins-agent-1
    restart: unless-stopped
    environment:
      - JENKINS_URL=http://jenkins:8080
      - JENKINS_AGENT_NAME=agent1
      - JENKINS_SECRET=9a58c5edf68345fbb1492c01673a85ecff07e138e3df9721898f5ffa9ee8d750
      - JENKINS_WEB_SOCKET=true
    depends_on:
      - jenkins
    networks:
      - net

  agent2:
    image: jenkins/inbound-agent
    container_name: jenkins-agent-2
    restart: unless-stopped
    environment:
      - JENKINS_URL=http://jenkins:8080
      - JENKINS_AGENT_NAME=agent2
      - JENKINS_SECRET=74188a2f466b3a354d47e5d5d95f790fad6cfd85bce68706aaf1b3eb43f9b1d0
      - JENKINS_WEB_SOCKET=true
    depends_on:
      - jenkins
    networks:
      - net

networks:
  net:
    driver: bridge
```

---

## 🧱 Step 8 — Relaunch Jenkins
```bash
sudo docker rm -f jenkins jenkins-agent-1 jenkins-agent-2 || true
sudo docker compose up -d
sudo docker compose logs -f jenkins
```

Check:
```
Jenkins is fully up and running
```

---

## 🔍 Step 9 — Verify Mounts
```bash
sudo docker inspect jenkins --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{println}}{{end}}'
sudo docker exec -it jenkins ls -la /var/jenkins_home | head
ls -la /data/Jenkins/jenkins_home | head
```

---

## 🧹 Step 10 — Cleanup Old Data
```bash
sudo rm -rf /var/lib/docker.bak
sudo mv /home/narendra/Jenkins/jenkins_home /home/narendra/Jenkins/jenkins_home.old
# After verifying:
sudo rm -rf /home/narendra/Jenkins/jenkins_home.old
```

---

## 🧠 Step 11 — SELinux Context (RHEL)
If not using `:Z`:
```bash
sudo semanage fcontext -a -t container_file_t "/data/Jenkins/jenkins_home(/.*)?"
sudo restorecon -Rv /data/Jenkins/jenkins_home
```

---

## ✅ Verification
```bash
df -h /data
docker info --format '{{.DockerRootDir}}'
sudo docker inspect jenkins --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{println}}{{end}}'
ls -ld /data/Jenkins/jenkins_home
```

✅ Migration completed successfully (November 2025).  
Maintainer: **Narendra Geddam**
