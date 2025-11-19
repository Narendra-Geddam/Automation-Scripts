# JFrog Artifactory OSS Installation Guide (Version 7.77.3 + PostgreSQL 14)

This guide explains how to install **JFrog Artifactory OSS (7.77.3)** using **Docker Compose** with **PostgreSQL 14**, using required permissions and version‑specific configuration.

---

## **1. Prerequisites**

Before starting, ensure your system has:

* Ubuntu 20.04 / 22.04 / 24.04
* Docker Engine installed
* Docker Compose (v2+) installed
* Minimum: 4GB RAM, 20GB free storage

Verify Docker installation:

```
docker --version
docker compose version
```

---

## **2. Create Required Directories**

Create workspace:

```
mkdir -p ~/jfrog
cd ~/jfrog
```

Create volumes:

```
mkdir -p artifactory-data
mkdir -p postgres-data
```

Set correct ownership and permissions:

```
sudo chown -R 1030:1030 artifactory-data
sudo chown -R 999:999 postgres-data
sudo chmod -R 700 artifactory-data
sudo chmod -R 700 postgres-data
```

**Why these IDs?**

* Artifactory container user = **1030**
* PostgreSQL container user = **999**

---

## **3. Create docker-compose.yml**

Create the file inside `~/jfrog`:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:14
    container_name: artifactory-postgres
    environment:
      POSTGRES_USER: artifactory
      POSTGRES_PASSWORD: artifactory
      POSTGRES_DB: artifactory
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    restart: always

  artifactory:
    image: releases-docker.jfrog.io/jfrog/artifactory-oss:7.77.3
    container_name: artifactory
    depends_on:
      - postgres
    environment:
      JF_SHARED_DATABASE_TYPE: postgresql
      JF_SHARED_DATABASE_DRIVER: org.postgresql.Driver
      JF_SHARED_DATABASE_URL: jdbc:postgresql://postgres:5432/artifactory
      JF_SHARED_DATABASE_USERNAME: artifactory
      JF_SHARED_DATABASE_PASSWORD: artifactory
    volumes:
      - ./artifactory-data:/var/opt/jfrog/artifactory
    ports:
      - "8081:8081"
      - "8082:8082"
    restart: always
```

---

## **4. Clean Old Volumes Before Starting**

If containers were already created earlier, clean them first:

```
docker stop artifactory artifactory-postgres

docker rm artifactory artifactory-postgres

sudo rm -rf artifactory-data postgres-data
```

Recreate fresh volumes:

```
mkdir artifactory-data postgres-data
sudo chown -R 1030:1030 artifactory-data
sudo chown -R 999:999 postgres-data
sudo chmod -R 700 artifactory-data
sudo chmod -R 700 postgres-data
```

---

## **5. Start Artifactory**

Run:

```
docker compose up -d
```

Check running containers:

```
docker ps
```

Follow Artifactory logs:

```
docker logs -f artifactory
```

---

## **6. Access Artifactory UI**

Open browser:

```
http://<your-server-ip>:8081
```

---

## **7. Important File Locations**

Host machine:

```
./artifactory-data
./postgres-data
```

Inside container:

```
/var/opt/jfrog/artifactory
```

---

## **8. Common Commands**

Restart services:

```
docker compose restart
```

Stop services:

```
docker compose down
```

Fresh reinstall:

```
sudo rm -rf artifactory-data/* postgres-data/*
```

---

## **9. Troubleshooting**

### **Permission denied errors**

Fix by reapplying correct ownership:

```
sudo chown -R 1030:1030 artifactory-data
sudo chown -R 999:999 postgres-data
```

### **Database connection refused**

Check PostgreSQL logs:

```
docker logs artifactory-postgres
```

---

Your Artifactory installation is now ready with version‑specific configuration and correct permissions.
