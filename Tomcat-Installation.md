🖥️ 1. Install Tomcat 9.0.112 on Ubuntu 22.04 Host
---

This installation supports:

✔ systemctl start tomcat
✔ systemctl stop tomcat
✔ Java check & auto-install
✔ Tomcat service running as a dedicated tomcat user

📌 Installation Script: install_tomcat_ubuntu22.sh
```bash
#!/usr/bin/env bash
set -euo pipefail

### CONFIGURABLE VARIABLES ###

TOMCAT_MAJOR="9"
TOMCAT_VERSION="9.0.112"
TOMCAT_USER="tomcat"
TOMCAT_GROUP="tomcat"
TOMCAT_INSTALL_DIR="/opt/tomcat"
TOMCAT_SERVICE_NAME="tomcat"
JAVA_PACKAGE="openjdk-17-jdk"

---

echo "[*] Checking Java installation..."
if command -v java >/dev/null 2>&1; then
    JAVA_VERSION=$(java -version 2>&1 | head -n 1)
    echo "[✔] Java already installed: $JAVA_VERSION"
else
    echo "[!] Java not found. Installing..."
    sudo apt-get update -y
    sudo apt-get install -y "$JAVA_PACKAGE"
fi

JAVA_HOME_PATH=$(dirname "$(dirname "$(readlink -f "$(which java)")")")
echo "[*] JAVA_HOME = $JAVA_HOME_PATH"

echo "[*] Creating Tomcat user..."
if ! id -u "$TOMCAT_USER" >/dev/null 2>&1; then
  sudo useradd -m -U -d "$TOMCAT_INSTALL_DIR" -s /bin/false "$TOMCAT_USER"
fi

echo "[*] Creating Tomcat directory..."
sudo mkdir -p "$TOMCAT_INSTALL_DIR"

TOMCAT_TAR="apache-tomcat-${TOMCAT_VERSION}.tar.gz"
TOMCAT_URL="https://dlcdn.apache.org/tomcat/tomcat-${TOMCAT_MAJOR}/v${TOMCAT_VERSION}/bin/${TOMCAT_TAR}"

echo "[*] Downloading Tomcat..."
cd /tmp
curl -fSL "$TOMCAT_URL" -o "$TOMCAT_TAR"

echo "[*] Extracting Tomcat..."
sudo tar -xzf "$TOMCAT_TAR" -C "$TOMCAT_INSTALL_DIR" --strip-components=1

echo "[*] Setting permissions..."
sudo chown -R "$TOMCAT_USER":"$TOMCAT_GROUP" "$TOMCAT_INSTALL_DIR"
sudo chmod -R u+rwX,go-rwx "$TOMCAT_INSTALL_DIR"

echo "[*] Creating systemd service..."

sudo bash -c "cat >/etc/systemd/system/${TOMCAT_SERVICE_NAME}.service" <<EOF
[Unit]
Description=Apache Tomcat Web Application Container
After=network.target

[Service]
Type=forking

User=${TOMCAT_USER}
Group=${TOMCAT_GROUP}

Environment="JAVA_HOME=${JAVA_HOME_PATH}"
Environment="CATALINA_HOME=${TOMCAT_INSTALL_DIR}"
Environment="CATALINA_BASE=${TOMCAT_INSTALL_DIR}"
Environment="CATALINA_PID=${TOMCAT_INSTALL_DIR}/temp/tomcat.pid"

ExecStart=${TOMCAT_INSTALL_DIR}/bin/startup.sh
ExecStop=${TOMCAT_INSTALL_DIR}/bin/shutdown.sh

Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

echo "[*] Enabling and starting Tomcat..."
sudo systemctl daemon-reload
sudo systemctl enable "$TOMCAT_SERVICE_NAME"
sudo systemctl start "$TOMCAT_SERVICE_NAME"

echo "[✔] Tomcat installed and running!"
echo "Check: systemctl status tomcat"
```

# Tomcat java 21 
```
#!/usr/bin/env bash
set -euo pipefail

### CONFIGURATION ###
TOMCAT_MAJOR="9"
TOMCAT_VERSION="9.0.112"
TOMCAT_USER="tomcat"
TOMCAT_GROUP="tomcat"
TOMCAT_INSTALL_DIR="/opt/tomcat"
TOMCAT_SERVICE_NAME="tomcat"

# 🔒 Explicit Java 21 (no auto-detection)
JAVA_HOME="/usr/lib/jvm/java-21-openjdk"

### PRE-CHECKS ###
echo "[*] Verifying Java 21..."
if [ ! -x "$JAVA_HOME/bin/java" ]; then
  echo "[ERROR] Java 21 not found at $JAVA_HOME"
  echo "Install Java 21 before running this script."
  exit 1
fi

"$JAVA_HOME/bin/java" -version
echo "[✔] Java 21 verified"

### USER & GROUP ###
echo "[*] Creating tomcat group/user if not exists..."
getent group "$TOMCAT_GROUP" >/dev/null || sudo groupadd "$TOMCAT_GROUP"
id "$TOMCAT_USER" >/dev/null 2>&1 || sudo useradd -r -g "$TOMCAT_GROUP" \
  -d "$TOMCAT_INSTALL_DIR" -s /sbin/nologin "$TOMCAT_USER"

### DOWNLOAD TOMCAT ###
echo "[*] Downloading Tomcat $TOMCAT_VERSION..."
cd /tmp
TOMCAT_TAR="apache-tomcat-${TOMCAT_VERSION}.tar.gz"
TOMCAT_URL="https://dlcdn.apache.org/tomcat/tomcat-${TOMCAT_MAJOR}/v${TOMCAT_VERSION}/bin/${TOMCAT_TAR}"

curl -fSL "$TOMCAT_URL" -o "$TOMCAT_TAR"

### INSTALL ###
echo "[*] Installing Tomcat..."
sudo mkdir -p "$TOMCAT_INSTALL_DIR"
sudo tar -xzf "$TOMCAT_TAR" -C "$TOMCAT_INSTALL_DIR" --strip-components=1

sudo chown -R "$TOMCAT_USER:$TOMCAT_GROUP" "$TOMCAT_INSTALL_DIR"
sudo chmod -R u+rwX,go-rwx "$TOMCAT_INSTALL_DIR"

### SYSTEMD SERVICE ###
echo "[*] Creating systemd service..."

sudo tee /etc/systemd/system/${TOMCAT_SERVICE_NAME}.service >/dev/null <<EOF
[Unit]
Description=Apache Tomcat ${TOMCAT_MAJOR}
After=network.target

[Service]
Type=forking
User=${TOMCAT_USER}
Group=${TOMCAT_GROUP}

Environment="JAVA_HOME=${JAVA_HOME}"
Environment="CATALINA_HOME=${TOMCAT_INSTALL_DIR}"
Environment="CATALINA_BASE=${TOMCAT_INSTALL_DIR}"
Environment="CATALINA_PID=${TOMCAT_INSTALL_DIR}/temp/tomcat.pid"

ExecStart=${TOMCAT_INSTALL_DIR}/bin/startup.sh
ExecStop=${TOMCAT_INSTALL_DIR}/bin/shutdown.sh

Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

### START TOMCAT ###
echo "[*] Enabling and starting Tomcat..."
sudo systemctl daemon-reload
sudo systemctl enable "$TOMCAT_SERVICE_NAME"
sudo systemctl restart "$TOMCAT_SERVICE_NAME"

### STATUS ###
echo "[✔] Tomcat installation completed"
systemctl status "$TOMCAT_SERVICE_NAME" --no-pager
```
📌 Usage Commands
sudo systemctl start tomcat
sudo systemctl stop tomcat
sudo systemctl restart tomcat
sudo systemctl status tomcat

# Access in browser:
http://<server-ip>:8080/


--------------------------------------------
🐳 2. Install Tomcat 9.0.112 Inside a Docker Container
--------------------------------------------

This Dockerfile builds:

✔ Ubuntu 22.04
✔ Java 17
✔ Tomcat 9.0.112
✔ Jenkins agent startup + Tomcat startup

📌 Dockerfile: Dockerfile
```Dockerfile
# --------------------------------------------------
# Ubuntu + Java 17 + Tomcat 9.0.112 + Jenkins Agent
# --------------------------------------------------
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive
ENV TZ=Asia/Kolkata

# Install Java 17 + utilities
RUN apt-get update && \
    apt-get install -y openjdk-17-jdk wget curl tar ca-certificates && \
    rm -rf /var/lib/apt/lists/*

# -----------------------------
# Install Tomcat (Download from Apache)
# -----------------------------
ENV TOMCAT_MAJOR=9
ENV TOMCAT_VERSION=9.0.112
ENV CATALINA_HOME=/opt/tomcat
ENV PATH="$CATALINA_HOME/bin:$PATH"

RUN mkdir -p "${CATALINA_HOME}" && \
    curl -fSL "https://dlcdn.apache.org/tomcat/tomcat-${TOMCAT_MAJOR}/v${TOMCAT_VERSION}/bin/apache-tomcat-${TOMCAT_VERSION}.tar.gz" \
      -o /tmp/tomcat.tar.gz && \
    tar -xzf /tmp/tomcat.tar.gz -C "${CATALINA_HOME}" --strip-components=1 && \
    rm -f /tmp/tomcat.tar.gz && \
    chmod +x "${CATALINA_HOME}"/bin/*.sh && \
    sed -i 's/Server port="8005"/Server port="-1"/' "${CATALINA_HOME}/conf/server.xml"

# -----------------------------
# Install Jenkins Agent (LOCAL)
# -----------------------------
RUN mkdir -p /opt/jenkins
WORKDIR /opt/jenkins
COPY tomcat-agent/agent.jar /opt/jenkins/agent.jar

# Expose Tomcat HTTP
EXPOSE 8080

# ---------------------------------------------------------
# Start Tomcat + Jenkins Agent
# ---------------------------------------------------------
CMD catalina.sh run & \
    java -jar /opt/jenkins/agent.jar \
      -url "$JENKINS_URL" \
      -secret "$JENKINS_SECRET" \
      -name "$JENKINS_AGENT_NAME" \
      -workDir "/opt/jenkins" \
      -webSocket
```
📌 Build Image
```bash
docker build -t tomcat-agent:9.0.112 .
```
📌 Run Container
``` bash
docker run -d \
  --name tomcat-agent \
  -p 8080:8080 \
  -e JENKINS_URL="http://your-jenkins/controller" \
  -e JENKINS_SECRET="xxxx" \
  -e JENKINS_AGENT_NAME="docker-agent" \
  tomcat-agent:9.0.112
```
🎯 Summary
Platform	Java Managed?	Tomcat Version	Control
Ubuntu Host	via APT	9.0.112	systemctl start/stop
Docker	Inside image	9.0.112	Runs via CMD (no systemd)
