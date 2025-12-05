# 🚀 Nexus OSS Automated Installation (AWS Optimized)
![My Image](image.png)

This repository contains an **automated, production-grade installation script** for  
**Nexus Repository OSS 3.86.2-01**, optimized for AWS t2.medium / t3.medium instances.

---

## 📌 Installation Script

```bash
#!/bin/bash
set -e

# Colors
GREEN="\e[32m"
YELLOW="\e[33m"
CYAN="\e[36m"
RED="\e[31m"
RESET="\e[0m"

function print_banner() {
echo -e "${CYAN}"
echo "#############################################################"
echo "#                                                           #"
echo "#        NEXUS OSS INSTALLER  –  PROFESSIONAL MODE          #"
echo "#       Optimized for AWS t2.medium / t3.medium             #"
echo "#                                                           #"
echo "#############################################################"
echo -e "${RESET}"
}

function step() {
    echo -e "${YELLOW}==> $1 ${RESET}"
}

function ok() {
    echo -e "${GREEN}[✔] $1 ${RESET}"
}

print_banner

step "Updating system..."
sudo apt update -y && sudo apt upgrade -y
ok "System updated"

step "Creating 2GB SWAP (AWS Optimized)..."
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile > /dev/null
sudo swapon /swapfile
echo "/swapfile swap swap defaults 0 0" | sudo tee -a /etc/fstab > /dev/null
ok "Swap enabled (2GB)"

step "Installing OpenJDK 11..."
sudo apt install openjdk-11-jdk -y
ok "Java 11 installed"

step "Creating nexus user..."
sudo useradd -M -r -s /bin/false nexus || true
ok "User 'nexus' created"

step "Downloading Nexus Repository OSS 3.86.2-01..."
cd /opt
sudo wget https://download.sonatype.com/nexus/3/nexus-3.86.2-01-linux-x86_64.tar.gz -O nexus.tar.gz
ok "Nexus downloaded"

step "Extracting Nexus..."
sudo tar -xvf nexus.tar.gz > /dev/null
NEXUS_DIR=$(ls -d nexus-3*)
sudo ln -s $NEXUS_DIR nexus
ok "Nexus extracted"

step "Creating Nexus data directories..."
sudo mkdir -p /opt/sonatype-work/nexus3/{log,tmp}
ok "Data dirs created"

step "Setting permissions..."
sudo chown -R nexus:nexus /opt/$NEXUS_DIR
sudo chown -R nexus:nexus /opt/nexus
sudo chown -R nexus:nexus /opt/sonatype-work
ok "Permissions fixed"

step "Writing nexus.rc..."
echo 'run_as_user="nexus"' | sudo tee /opt/nexus/bin/nexus.rc > /dev/null
ok "nexus.rc configured"

step "Writing nexus.vmoptions..."
sudo bash -c 'cat > /opt/nexus/bin/nexus.vmoptions <<EOF
-Xms2g
-Xmx2g
-XX:MaxDirectMemorySize=3g

-Dkaraf.home=/opt/nexus
-Dkaraf.base=/opt/nexus
-Dkaraf.etc=/opt/nexus/etc/karaf

-Djava.util.logging.config.file=/opt/nexus/etc/karaf/java.util.logging.properties

-Dkaraf.data=/opt/sonatype-work/nexus3
-Dkaraf.log=/opt/sonatype-work/nexus3/log
-Djava.io.tmpdir=/opt/sonatype-work/nexus3/tmp
EOF'
ok "VM options configured"

step "Creating systemd service..."
sudo bash -c 'cat > /etc/systemd/system/nexus.service <<EOF
[Unit]
Description=Nexus Repository Manager
After=network.target

[Service]
Type=forking
LimitNOFILE=65536
User=nexus
Group=nexus
ExecStart=/opt/nexus/bin/nexus start
ExecStop=/opt/nexus/bin/nexus stop
Restart=on-abort

[Install]
WantedBy=multi-user.target
EOF'
ok "Systemd service created"

step "Starting Nexus..."
sudo systemctl daemon-reload
sudo systemctl enable nexus
sudo systemctl start nexus

sleep 8

if systemctl is-active --quiet nexus; then
    ok "Nexus is RUNNING at: http://<your-server-ip>:8081"
else
    echo -e "${RED}[✘] Nexus failed to start. Check logs: /opt/sonatype-work/nexus3/log/nexus.log${RESET}"
fi

echo -e "${GREEN}"
echo "#############################################################"
echo "#       Installation Completed Successfully (AWS-Optimized) #"
echo "#       Admin password file:                                 #"
echo "#   /opt/sonatype-work/nexus3/admin.password                #"
echo "#############################################################"
echo -e "${RESET}"
```

---

## 📜 Usage

```bash
chmod +x install_nexus.sh
sudo ./install_nexus.sh
```

---

## 📍 Nexus Access

URL:
```
http://<your-server-ip>:8081
```

Admin password:
```
/opt/sonatype-work/nexus3/admin.password
```

---

## 🛑 Service Commands

```bash
sudo systemctl start nexus
sudo systemctl stop nexus
sudo systemctl restart nexus
sudo systemctl status nexus
```

---

## 🧩 Troubleshooting

Check logs:
```bash
sudo tail -n 50 /opt/sonatype-work/nexus3/log/nexus.log
```

Ensure swap is active:
```bash
swapon --show
```
