# Ubuntu Docker DevOps Lab

Complete setup for a local DevOps lab using Docker Compose with:

- Jenkins
- SonarQube
- Nexus Repository
- Apache Tomcat
- Custom Docker bridge network
- Persistent data under `/data/devops-lab`
- Host ports starting from `8050`

---

## 1. Architecture

```text
                         Ubuntu Host
                             │
                       Docker Engine
                             │
                     devops-net (bridge)
                             │
       ┌─────────────────────┼─────────────────────┐
       │                     │                     │
    Jenkins              SonarQube              Nexus
    :8080                  :9000                 :8081
       │                     │                     │
       └─────────────────────┼─────────────────────┘
                             │
                           Tomcat
                           :8080


Host Port Mapping
────────────────────────────────

8050 → Jenkins
8051 → SonarQube
8052 → Nexus
8053 → Tomcat
```

---

# 2. Prerequisites

Recommended:

- Ubuntu Desktop/Server
- 16 GB RAM recommended
- At least 50 GB free disk space
- Internet connectivity
- `sudo` access

---

# 3. Update Ubuntu

```bash
sudo apt update
sudo apt upgrade -y
```

Install required packages:

```bash
sudo apt install -y ca-certificates curl gnupg
```

---

# 4. Remove Old Docker Packages

If older Docker packages are installed:

```bash
sudo apt remove -y \
  docker.io \
  docker-doc \
  docker-compose \
  podman-docker \
  containerd \
  runc
```

---

# 5. Add Docker Repository

Create the Docker keyring directory:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

Download Docker's GPG key:

```bash
sudo curl -fsSL \
  https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
```

Set permissions:

```bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add Docker repository:

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Update package information:

```bash
sudo apt update
```

---

# 6. Install Docker Engine and Docker Compose

```bash
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

---

# 7. Verify Docker

Check Docker:

```bash
docker --version
```

Check Docker Compose:

```bash
docker compose version
```

Check Docker service:

```bash
sudo systemctl status docker
```

Enable and start Docker:

```bash
sudo systemctl enable --now docker
```

---

# 8. Configure Docker Without sudo

Add your user to the Docker group:

```bash
sudo usermod -aG docker $USER
```

Verify the group:

```bash
getent group docker
```

Expected:

```text
docker:x:<gid>:$USER
```

For example:

```text
docker:x:<gid>:$USER
```

If the current terminal has not picked up the group membership, either:

### Option 1 — log out and log back in

Recommended.

Then verify:

```bash
groups
```

You should see:

```text
docker
```

### Option 2 — use `newgrp`

If `newgrp` is unavailable:

```bash
sudo apt install -y util-linux-extra
```

Then:

```bash
newgrp docker
```

Verify:

```bash
groups
docker ps
```

Docker should now work without `sudo`.

---

# 9. Create DevOps Lab Directory

All persistent DevOps application data is stored under `/data`.

Create the main directory:

```bash
sudo mkdir -p /data/devops-lab
```

Give your user ownership:

```bash
sudo chown -R $USER:$USER /data/devops-lab
```

Create the application directories:

```bash
mkdir -p \
  /data/devops-lab/jenkins \
  /data/devops-lab/nexus \
  /data/devops-lab/sonarqube/data \
  /data/devops-lab/sonarqube/extensions \
  /data/devops-lab/sonarqube/logs \
  /data/devops-lab/tomcat/logs \
  /data/devops-lab/tomcat/webapps
```

Move into the directory:

```bash
cd /data/devops-lab
```

---

# 10. Directory Structure

Verify:

```bash
tree /data/devops-lab
```

Expected:

```text
/data/devops-lab
├── jenkins
├── nexus
├── sonarqube
│   ├── data
│   ├── extensions
│   └── logs
└── tomcat
    ├── logs
    └── webapps
```

If `tree` is not installed:

```bash
sudo apt install -y tree
```

After creating the Compose file, the final structure will be:

```text
/data/devops-lab
├── docker-compose.yml
├── jenkins
├── nexus
├── sonarqube
│   ├── data
│   ├── extensions
│   └── logs
└── tomcat
    ├── logs
    └── webapps
```

---

# 11. Create Docker Compose File

From:

```bash
cd /data/devops-lab
```

Create:

```bash
nano docker-compose.yml
```

Use the following configuration:

```yaml
services:

  # ============================================================
  # Jenkins
  # Host Port     : 8050
  # Container Port: 8080
  # ============================================================
  jenkins:
    image: jenkins/jenkins:lts-jdk21
    container_name: jenkins
    restart: unless-stopped

    ports:
      - "8050:8080"
      - "50000:50000"

    volumes:
      - /data/devops-lab/jenkins:/var/jenkins_home

    networks:
      - devops-net

    environment:
      TZ: Asia/Kolkata


  # ============================================================
  # SonarQube
  # Host Port     : 8051
  # Container Port: 9000
  # ============================================================
  sonarqube:
    image: sonarqube:lts-community
    container_name: sonarqube
    restart: unless-stopped

    ports:
      - "8051:9000"

    volumes:
      - /data/devops-lab/sonarqube/data:/opt/sonarqube/data
      - /data/devops-lab/sonarqube/extensions:/opt/sonarqube/extensions
      - /data/devops-lab/sonarqube/logs:/opt/sonarqube/logs

    networks:
      - devops-net

    environment:
      TZ: Asia/Kolkata


  # ============================================================
  # Nexus Repository
  # Host Port     : 8052
  # Container Port: 8081
  # ============================================================
  nexus:
    image: sonatype/nexus3:latest
    container_name: nexus
    restart: unless-stopped

    ports:
      - "8052:8081"

    volumes:
      - /data/devops-lab/nexus:/nexus-data

    networks:
      - devops-net

    environment:
      TZ: Asia/Kolkata


  # ============================================================
  # Apache Tomcat
  # Host Port     : 8053
  # Container Port: 8080
  # ============================================================
  tomcat:
    image: tomcat:10.1-jdk21
    container_name: tomcat
    restart: unless-stopped

    ports:
      - "8053:8080"

    volumes:
      - /data/devops-lab/tomcat/webapps:/usr/local/tomcat/webapps
      - /data/devops-lab/tomcat/logs:/usr/local/tomcat/logs

    networks:
      - devops-net

    environment:
      TZ: Asia/Kolkata


# ============================================================
# Custom Docker Bridge Network
# ============================================================
networks:

  devops-net:
    name: devops-net
    driver: bridge
```

Save:

```text
CTRL + O
ENTER
CTRL + X
```

---

# 12. Validate Docker Compose

Before starting the services:

```bash
cd /data/devops-lab
docker compose config
```

The command should complete without errors.

You should see:

```text
name: devops-lab
services:
  jenkins:
  nexus:
  sonarqube:
  tomcat:
networks:
  devops-net:
```

---

# 13. Pull Docker Images

```bash
docker compose pull
```

This downloads:

```text
Jenkins
SonarQube
Nexus
Tomcat
```

Check images:

```bash
docker images
```

---

# 14. Start All Services

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
```

Also:

```bash
docker ps
```

Expected containers:

```text
jenkins
sonarqube
nexus
tomcat
```

---

# 15. Check Docker Network

List networks:

```bash
docker network ls
```

You should see:

```text
devops-net
```

Inspect:

```bash
docker network inspect devops-net
```

All four containers should be connected:

```text
jenkins
sonarqube
nexus
tomcat
```

---

# 16. Application URLs

From the Ubuntu host:

| Application | URL |
|---|---|
| Jenkins | http://localhost:8050 |
| SonarQube | http://localhost:8051 |
| Nexus | http://localhost:8052 |
| Tomcat | http://localhost:8053 |

---

# 17. Jenkins Initial Password

Get the Jenkins initial administrator password:

```bash
docker exec jenkins \
  cat /var/jenkins_home/secrets/initialAdminPassword
```

Open:

```text
http://localhost:8050
```

Use the password returned by the command.

---

# 18. Nexus Initial Password

Get the Nexus administrator password:

```bash
docker exec nexus \
  cat /nexus-data/admin.password
```

Open:

```text
http://localhost:8052
```

Default username:

```text
admin
```

Use the password returned by the command.

---

# 19. SonarQube

Open:

```text
http://localhost:8051
```

Complete the initial SonarQube setup/login.

---

# 21. Tomcat

Open:

```text
http://localhost:8053
```

Tomcat listens internally on:

```text
8080
```

Docker maps the host port:

```text
8053 → 8080
```

---

# 22. Docker Internal Networking

The containers communicate using the custom bridge network:

```text
devops-net
```

Docker provides internal DNS resolution using service/container names.

From Jenkins:

```text
SonarQube:
http://sonarqube:9000

Nexus:
http://nexus:8081

Tomcat:
http://tomcat:8080
```

Do not use these from inside Jenkins:

```text
http://localhost:8051
http://localhost:8052
http://localhost:8053
```

`localhost` inside Jenkins means the Jenkins container itself.

---

# 23. Test Container-to-Container Connectivity

Enter Jenkins:

```bash
docker exec -it jenkins bash
```

Test SonarQube:

```bash
curl http://sonarqube:9000
```

Test Nexus:

```bash
curl http://nexus:8081
```

Test Tomcat:

```bash
curl http://tomcat:8080
```

Exit:

```bash
exit
```

---

# 24. Check Logs

Jenkins:

```bash
docker logs -f jenkins
```

SonarQube:

```bash
docker logs -f sonarqube
```

Nexus:

```bash
docker logs -f nexus
```

Tomcat:

```bash
docker logs -f tomcat
```

All services:

```bash
docker compose logs -f
```

Show only the latest 50 lines:

```bash
docker compose logs --tail=50
```

Specific service:

```bash
docker compose logs --tail=50 sonarqube
```

---

# 25. Check Resource Usage

```bash
docker stats
```

SonarQube and Nexus can consume significant memory, so monitor resources when running other workloads such as VMware or OpenShift.

---

# 26. Start and Stop Services

Stop:

```bash
docker compose stop
```

Start:

```bash
docker compose start
```

Restart all:

```bash
docker compose restart
```

Restart Jenkins:

```bash
docker compose restart jenkins
```

Restart SonarQube:

```bash
docker compose restart sonarqube
```

Restart Nexus:

```bash
docker compose restart nexus
```

Restart Tomcat:

```bash
docker compose restart tomcat
```

---

# 27. Shutdown

Remove containers and keep all data:

```bash
docker compose down
```

Start again:

```bash
docker compose up -d
```

Because the data is stored under `/data/devops-lab`, the application data remains.

---

# 28. Completely Delete the Lab

This Compose file uses host bind mounts, not Docker named volumes.

To remove the containers and network:

```bash
docker compose down
```

To remove all application data as well:

```bash
sudo rm -rf /data/devops-lab
```

**Warning:** the command above permanently deletes Jenkins, SonarQube, Nexus and Tomcat data.

---

# 29. Persistent Storage

All persistent data is stored directly under:

```text
/data/devops-lab
```

Mapping:

```text
/data/devops-lab/jenkins
    ↓
/var/jenkins_home


/data/devops-lab/sonarqube/data
    ↓
/opt/sonarqube/data


/data/devops-lab/sonarqube/extensions
    ↓
/opt/sonarqube/extensions


/data/devops-lab/sonarqube/logs
    ↓
/opt/sonarqube/logs


/data/devops-lab/nexus
    ↓
/nexus-data


/data/devops-lab/tomcat/webapps
    ↓
/usr/local/tomcat/webapps


/data/devops-lab/tomcat/logs
    ↓
/usr/local/tomcat/logs
```

---

# 30. Final Directory Structure

```text
/data/devops-lab/
│
├── docker-compose.yml
│
├── jenkins/
│   └── Jenkins persistent data
│
├── nexus/
│   └── Nexus persistent data
│
├── sonarqube/
│   ├── data/
│   ├── extensions/
│   └── logs/
│
└── tomcat/
    ├── logs/
    └── webapps/
```

---

# 31. Final Port Mapping

```text
Host                     Container
────────────────────────────────────────────
localhost:8050  ───────→ jenkins:8080

localhost:8051  ───────→ sonarqube:9000

localhost:8052  ───────→ nexus:8081

localhost:8053  ───────→ tomcat:8080
```

Jenkins agent port:

```text
50000 → Jenkins 50000
```

---

# 32. Final Network

```text
                    devops-net
                  Custom Bridge
                       │
       ┌───────────────┼────────────────┐
       │               │                │
       ▼               ▼                ▼
    Jenkins         SonarQube         Nexus
    :8080             :9000            :8081
       │               │                │
       └───────────────┼────────────────┘
                       │
                       ▼
                    Tomcat
                    :8080
```

---

# 33. Jenkins CI/CD Flow

The intended CI/CD flow is:

```text
Git Repository
      │
      ▼
   Jenkins
   :8050
      │
      ▼
 Maven Build
      │
      ▼
 Unit Tests
      │
      ▼
 SonarQube
      │
      ▼
 Quality Gate
      │
      ▼
 Package WAR/JAR
      │
      ▼
 Nexus Repository
      │
      ▼
 Deploy Artifact
      │
      ▼
 Tomcat
      │
      ▼
 Application
```

Internal URLs used by Jenkins:

```text
SonarQube → http://sonarqube:9000
Nexus     → http://nexus:8081
Tomcat    → http://tomcat:8080
```

---

# 34. Quick Installation

Once Docker is installed:

```bash
cd /data/devops-lab
```

Validate:

```bash
docker compose config
```

Pull:

```bash
docker compose pull
```

Start:

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
```

Check network:

```bash
docker network inspect devops-net
```

Access:

```text
Jenkins:
http://localhost:8050

SonarQube:
http://localhost:8051

Nexus:
http://localhost:8052

Tomcat:
http://localhost:8053
```

---

# 35. Troubleshooting

## Docker permission denied

Check:

```bash
getent group docker
```

If your username appears:

```text
docker:x:<gid>:sreekanth
```

log out and log back in.

Then:

```bash
groups
docker ps
```

If `newgrp` is missing:

```bash
sudo apt install -y util-linux-extra
newgrp docker
```

---

## Check container status

```bash
docker compose ps
```

---

## Check all logs

```bash
docker compose logs --tail=100
```

---

## Check a specific service

```bash
docker compose logs --tail=100 jenkins
docker compose logs --tail=100 sonarqube
docker compose logs --tail=100 nexus
docker compose logs --tail=100 tomcat
```

---

## Check ports

```bash
sudo ss -lntp | grep -E '8050|8051|8052|8053'
```

Expected host ports:

```text
8050
8051
8052
8053
```

---

## Check network

```bash
docker network inspect devops-net
```

---

# 36. Daily Commands

Go to the lab:

```bash
cd /data/devops-lab
```

Status:

```bash
docker compose ps
```

Start:

```bash
docker compose up -d
```

Stop:

```bash
docker compose stop
```

Restart:

```bash
docker compose restart
```

Logs:

```bash
docker compose logs -f
```

Resource usage:

```bash
docker stats
```

Network:

```bash
docker network inspect devops-net
```

---

# 37. Summary

This setup provides a persistent local DevOps environment:

```text
Jenkins
  ↓
SonarQube
  ↓
Nexus
  ↓
Tomcat
```

with:

```text
Custom bridge network:
devops-net

Persistent storage:
 /data/devops-lab

Host ports:
 8050 Jenkins
 8051 SonarQube
 8052 Nexus
 8053 Tomcat
```

All application data is kept outside Docker's internal volume storage under `/data/devops-lab`, making the environment easier to inspect, back up, migrate, and rebuild.
