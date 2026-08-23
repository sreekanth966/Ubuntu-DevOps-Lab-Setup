# Ubuntu Docker DevOps Lab

A complete local DevOps lab on Ubuntu using Docker Compose:

- Jenkins
- SonarQube
- Nexus Repository
- Apache Tomcat
- Custom Docker bridge network
- Persistent storage under `/data/devops-lab`
- Host ports starting from `8050`
- Tomcat Manager and Host Manager
- Tomcat administrator and Jenkins deployment users
- Jenkins → SonarQube → Nexus → Tomcat connectivity

> This README is written for a **fresh installation** and follows the correct order so the Tomcat bind mounts do not hide the original Tomcat configuration before it is copied.

---

# 1. Architecture

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
```

## Host Port Mapping

```text
8050 → Jenkins
8051 → SonarQube
8052 → Nexus
8053 → Tomcat
50000 → Jenkins inbound agent
```

---

# 2. Prerequisites

Recommended:

- Ubuntu Desktop/Server
- 16 GB RAM or more
- At least 50 GB free disk space
- Internet connectivity
- `sudo` access

Check Ubuntu:

```bash
cat /etc/os-release
```

Check architecture:

```bash
uname -m
```

---

# 3. Update Ubuntu

```bash
sudo apt update
sudo apt upgrade -y
```

Install required packages:

```bash
sudo apt install -y \
  ca-certificates \
  curl \
  gnupg \
  lsb-release \
  tree \
  util-linux-extra
```

---

# 4. Remove Conflicting/Old Docker Packages

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

If some packages are not installed, that is fine.

---

# 5. Add Official Docker Repository

Create keyring directory:

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

# 7. Enable Docker

```bash
sudo systemctl enable --now docker
```

Check:

```bash
sudo systemctl status docker
```

Check Docker:

```bash
docker --version
```

Check Compose:

```bash
docker compose version
```

Test:

```bash
sudo docker ps
```

---

# 8. Configure Docker Without sudo

Add the current user to the Docker group:

```bash
sudo usermod -aG docker $USER
```

Verify:

```bash
getent group docker
```

The current user should appear in the output.

Apply the group change.

### Recommended

Log out and log back into Ubuntu.

Then:

```bash
groups
```

You should see:

```text
docker
```

Test:

```bash
docker ps
```

### If `newgrp` is required

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

> Do not continue with the rest of the installation until `docker ps` works without `sudo`.

---

# 9. Configure SonarQube Host Requirements

SonarQube uses Elasticsearch internally and requires suitable Linux kernel settings.

Set:

```bash
sudo sysctl -w vm.max_map_count=524288
```

Set:

```bash
sudo sysctl -w fs.file-max=131072
```

Make the settings persistent:

```bash
sudo tee /etc/sysctl.d/99-sonarqube.conf > /dev/null <<'EOF'
vm.max_map_count=524288
fs.file-max=131072
EOF
```

Apply:

```bash
sudo sysctl --system
```

Verify:

```bash
sysctl vm.max_map_count
```

Expected:

```text
vm.max_map_count = 524288
```

Verify:

```bash
sysctl fs.file-max
```

Expected value should be at least:

```text
131072
```

---

# 10. Create DevOps Lab Directory

All persistent application data will be stored under:

```text
/data/devops-lab
```

Create it:

```bash
sudo mkdir -p /data/devops-lab
```

Give ownership to the current user:

```bash
sudo chown -R $USER:$USER /data/devops-lab
```

Create application directories:

```bash
mkdir -p \
  /data/devops-lab/jenkins \
  /data/devops-lab/nexus \
  /data/devops-lab/sonarqube/data \
  /data/devops-lab/sonarqube/extensions \
  /data/devops-lab/sonarqube/logs \
  /data/devops-lab/tomcat/conf \
  /data/devops-lab/tomcat/logs \
  /data/devops-lab/tomcat/webapps
```

Move into the lab:

```bash
cd /data/devops-lab
```

---

# 11. Important Tomcat Initialization Concept

The final Compose file contains:

```yaml
volumes:
  - /data/devops-lab/tomcat/conf:/usr/local/tomcat/conf
  - /data/devops-lab/tomcat/webapps:/usr/local/tomcat/webapps
  - /data/devops-lab/tomcat/logs:/usr/local/tomcat/logs
```

A bind mount hides the corresponding directory inside the image.

Therefore:

```text
Host:
/data/devops-lab/tomcat/conf

mounts over:

Container:
/usr/local/tomcat/conf
```

Likewise:

```text
/data/devops-lab/tomcat/webapps

mounts over:

/usr/local/tomcat/webapps
```

Because the host directories are initially empty, the original Tomcat configuration and web applications must be copied **before starting the final Tomcat container with these bind mounts**.

This avoids the Tomcat `404`, `401`, and configuration problems encountered when the mount is introduced after the container is already running.

---

# 12. Create a Temporary Tomcat Container for Initialization

Pull the Tomcat image:

```bash
docker pull tomcat:10.1-jdk21
```

Create a temporary container:

```bash
docker create \
  --name tomcat-init \
  tomcat:10.1-jdk21
```

The container does not need to run.

Verify:

```bash
docker ps -a --filter name=tomcat-init
```

---

# 13. Copy Tomcat Configuration to Persistent Storage

Copy the original Tomcat configuration:

```bash
docker cp tomcat-init:/usr/local/tomcat/conf \
  /data/devops-lab/tomcat/
```

Verify:

```bash
ls -la /data/devops-lab/tomcat/conf
```

You should see files such as:

```text
server.xml
web.xml
tomcat-users.xml
tomcat-users.xsd
context.xml
logging.properties
```

---

# 14. Copy Tomcat Web Applications

Copy ROOT:

```bash
docker cp tomcat-init:/usr/local/tomcat/webapps.dist/ROOT \
  /data/devops-lab/tomcat/webapps/
```

Copy Manager:

```bash
docker cp tomcat-init:/usr/local/tomcat/webapps.dist/manager \
  /data/devops-lab/tomcat/webapps/
```

Copy Host Manager:

```bash
docker cp tomcat-init:/usr/local/tomcat/webapps.dist/host-manager \
  /data/devops-lab/tomcat/webapps/
```

Verify:

```bash
ls -la /data/devops-lab/tomcat/webapps
```

Expected:

```text
ROOT/
manager/
host-manager/
```

---

# 15. Remove Temporary Tomcat Container

The temporary container is no longer required:

```bash
docker rm tomcat-init
```

Verify:

```bash
docker ps -a --filter name=tomcat-init
```

There should be no result.

---

# 16. Configure Tomcat Users

Create:

```bash
cat > /data/devops-lab/tomcat/conf/tomcat-users.xml <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>

<tomcat-users
    xmlns="http://tomcat.apache.org/xml"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://tomcat.apache.org/xml tomcat-users.xsd"
    version="1.0">

    <!-- Manager roles -->
    <role rolename="manager-gui"/>
    <role rolename="manager-script"/>
    <role rolename="manager-jmx"/>
    <role rolename="manager-status"/>

    <!-- Host Manager roles -->
    <role rolename="admin-gui"/>
    <role rolename="admin-script"/>

    <!-- Tomcat Administrator -->
    <user
        username="admin"
        password="Admin#123456"
        roles="manager-gui,manager-script,manager-jmx,manager-status,admin-gui,admin-script"/>

    <!-- Jenkins deployment user -->
    <user
        username="jenkins-deploy"
        password="Jenkins#123"
        roles="manager-script"/>

</tomcat-users>
EOF
```

Verify:

```bash
cat /data/devops-lab/tomcat/conf/tomcat-users.xml
```

## Tomcat Administrator

```text
Username: admin
Password: Admin#123456
```

Roles:

```text
manager-gui
manager-script
manager-jmx
manager-status
admin-gui
admin-script
```

## Jenkins Deployment User

```text
Username: jenkins-deploy
Password: Jenkins#123
```

Role:

```text
manager-script
```

The Jenkins account is intended for automated WAR deployment.

> These credentials are for this local lab only. Do not use them in production.

---

# 17. Configure Tomcat Manager Access

Tomcat's Manager and Host Manager applications may contain a `RemoteCIDRValve` restricting access to localhost.

Check Manager:

```bash
grep -nE "Valve|Remote" \
  /data/devops-lab/tomcat/webapps/manager/META-INF/context.xml
```

Check Host Manager:

```bash
grep -nE "Valve|Remote" \
  /data/devops-lab/tomcat/webapps/host-manager/META-INF/context.xml
```

The default may contain:

```xml
<Valve className="org.apache.catalina.valves.RemoteCIDRValve"
       allow="127.0.0.0/8,::1/128" />
```

For this isolated local lab, change it to:

```xml
<Valve className="org.apache.catalina.valves.RemoteCIDRValve"
       allow="0.0.0.0/0,::/0" />
```

Run:

```bash
sed -i 's/allow="127.0.0.0\/8,::1\/128"/allow="0.0.0.0\/0,::\/0"/' \
/data/devops-lab/tomcat/webapps/manager/META-INF/context.xml
```

And:

```bash
sed -i 's/allow="127.0.0.0\/8,::1\/128"/allow="0.0.0.0\/0,::\/0"/' \
/data/devops-lab/tomcat/webapps/host-manager/META-INF/context.xml
```

Verify:

```bash
grep -nA1 "RemoteCIDRValve" \
/data/devops-lab/tomcat/webapps/manager/META-INF/context.xml
```

```bash
grep -nA1 "RemoteCIDRValve" \
/data/devops-lab/tomcat/webapps/host-manager/META-INF/context.xml
```

> `0.0.0.0/0,::/0` permits access from any reachable address. This is acceptable only for an isolated local lab. In production, restrict access to trusted CIDRs.

---


# 17A. Tomcat Web Applications

The Tomcat Docker image keeps the default applications such as `ROOT`, `manager`, and `host-manager` under `/usr/local/tomcat/webapps.dist/`, while the active `webapps` directory is empty. Because our Compose file mounts `/data/devops-lab/tomcat/webapps` to `/usr/local/tomcat/webapps`, we copy these required applications into the persistent host directory so the Tomcat welcome page, Manager, and Host Manager are available.

# 18. Create Final Docker Compose File

Create:

```bash
nano /data/devops-lab/docker-compose.yml
```

Use:

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
      - /data/devops-lab/tomcat/conf:/usr/local/tomcat/conf
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

---

# 19. Validate Compose Before Starting

```bash
cd /data/devops-lab
```

Run:

```bash
docker compose config
```

It must complete without errors.

---

# 20. Verify Tomcat Files Before Starting

This is an important fresh-install checkpoint.

Check configuration:

```bash
test -f /data/devops-lab/tomcat/conf/tomcat-users.xml \
  && echo "Tomcat configuration OK"
```

Check ROOT:

```bash
test -d /data/devops-lab/tomcat/webapps/ROOT \
  && echo "ROOT application OK"
```

Check Manager:

```bash
test -d /data/devops-lab/tomcat/webapps/manager \
  && echo "Manager application OK"
```

Check Host Manager:

```bash
test -d /data/devops-lab/tomcat/webapps/host-manager \
  && echo "Host Manager application OK"
```

All four should report `OK`.

---

# 21. Pull All Images

```bash
docker compose pull
```

Check:

```bash
docker images
```

---

# 22. Start the Complete Lab

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
```

Expected:

```text
jenkins
sonarqube
nexus
tomcat
```

All should eventually be:

```text
Up
```

---

# 23. Important: Recreate After Compose Changes

If `docker-compose.yml` is changed, use:

```bash
docker compose up -d --force-recreate
```

For Tomcat only:

```bash
docker compose up -d --force-recreate tomcat
```

After the initial Tomcat files have been copied into `/data/devops-lab`, **do not copy `conf` from the container back to the host again**, because that can overwrite your customized configuration.

The persistent source of truth is:

```text
/data/devops-lab/tomcat/conf
/data/devops-lab/tomcat/webapps
/data/devops-lab/tomcat/logs
```

---

# 24. Verify Docker Network

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

Expected containers:

```text
jenkins
sonarqube
nexus
tomcat
```

---

# 25. Verify Tomcat Persistent Mounts

Run:

```bash
docker inspect tomcat \
  --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

Expected:

```text
/data/devops-lab/tomcat/conf -> /usr/local/tomcat/conf
/data/devops-lab/tomcat/logs -> /usr/local/tomcat/logs
/data/devops-lab/tomcat/webapps -> /usr/local/tomcat/webapps
```

This confirms that the customized host configuration is actually being used.

---

# 26. Verify Tomcat Users Inside Container

```bash
docker exec tomcat cat /usr/local/tomcat/conf/tomcat-users.xml
```

You must see:

```text
admin
Admin#123456
```

and:

```text
jenkins-deploy
Jenkins#123
```

---

# 27. Verify Tomcat Applications

```bash
docker exec tomcat ls -la /usr/local/tomcat/webapps
```

Expected:

```text
ROOT
manager
host-manager
```

---

# 28. Application URLs

| Application | URL |
|---|---|
| Jenkins | http://localhost:8050 |
| SonarQube | http://localhost:8051 |
| Nexus | http://localhost:8052 |
| Tomcat | http://localhost:8053 |

---

# 29. Jenkins Initial Password

Get:

```bash
docker exec jenkins \
  cat /var/jenkins_home/secrets/initialAdminPassword
```

Open:

```text
http://localhost:8050
```

Use the generated password.

> Jenkins does not use a predefined `admin/admin` password. The initial administrator password is generated by Jenkins.

---

# 30. SonarQube Initial Login

Open:

```text
http://localhost:8051
```

Initial credentials:

```text
Username: admin
Password: admin
```

Change the password after the first login if the service will be exposed beyond the local lab.

---

# 31. Nexus Initial Login

Open:

```text
http://localhost:8052
```

For the initial administrator password:

```bash
docker exec nexus cat /nexus-data/admin.password
```

If the file exists, use that password.

After Nexus initialization, the password file may no longer be available. Use the administrator password configured during Nexus setup.

---

# 32. Tomcat URLs

Main Tomcat:

```text
http://localhost:8053
```

Server Status:

```text
http://localhost:8053/manager/status
```

Manager App:

```text
http://localhost:8053/manager/html
```

Host Manager:

```text
http://localhost:8053/host-manager/html
```

Login:

```text
Username: admin
Password: Admin#123456
```

---

# 33. Test Tomcat Manager Authentication

Do not use only:

```bash
curl -I
```

because that only performs a HEAD request and does not properly verify the authenticated Manager API.

Test:

```bash
curl -u 'admin:Admin#123456' \
  http://localhost:8053/manager/text/list
```

Expected:

```text
OK - Listed applications for virtual host localhost
```

Test Jenkins deployment user:

```bash
curl -u 'jenkins-deploy:Jenkins#123' \
  http://localhost:8053/manager/text/list
```

Expected:

```text
OK - Listed applications for virtual host localhost
```

---

# 34. Test Jenkins → Tomcat Connectivity

This tests the Docker bridge network rather than the host port.

Enter Jenkins:

```bash
docker exec -it jenkins bash
```

Then:

```bash
curl -u 'jenkins-deploy:Jenkins#123' \
  http://tomcat:8080/manager/text/list
```

Expected:

```text
OK - Listed applications for virtual host localhost
```

Exit:

```bash
exit
```

The Jenkins-to-Tomcat endpoint is:

```text
http://tomcat:8080/manager/text
```

Do not use:

```text
http://localhost:8053/manager/text
```

from inside Jenkins because `localhost` refers to the Jenkins container itself.

---

# 35. Nexus Permission Troubleshooting

Nexus runs as a non-root user.

Check:

```bash
docker run --rm sonatype/nexus3:latest id nexus
```

Typical output:

```text
uid=200(nexus) gid=200(nexus) groups=200(nexus)
```

If Nexus reports permission errors on `/nexus-data`:

```bash
docker compose stop nexus
```

For the standard image:

```bash
sudo chown -R 200:200 /data/devops-lab/nexus
```

Set permissions:

```bash
sudo chmod -R u+rwX,go-rwx /data/devops-lab/nexus
```

Start:

```bash
docker compose up -d nexus
```

Check:

```bash
docker compose ps
```

Logs:

```bash
docker compose logs --tail=50 nexus
```

---

# 36. Tomcat 404 Troubleshooting

If:

```text
http://localhost:8053
```

returns:

```text
HTTP Status 404
```

check:

```bash
ls -la /data/devops-lab/tomcat/webapps
```

You should have:

```text
ROOT/
manager/
host-manager/
```

If `ROOT` is missing, the initial copy was not completed.

Use a temporary container again:

```bash
docker create --name tomcat-init tomcat:10.1-jdk21
```

Copy ROOT:

```bash
docker cp tomcat-init:/usr/local/tomcat/webapps.dist/ROOT \
  /data/devops-lab/tomcat/webapps/
```

Remove the temporary container:

```bash
docker rm tomcat-init
```

Recreate Tomcat:

```bash
docker compose up -d --force-recreate tomcat
```

---

# 37. Tomcat Manager 403 Troubleshooting

If:

```text
http://localhost:8053/manager/html
```

returns:

```text
403 Access Denied
```

check:

```bash
grep -nE "Valve|Remote" \
/data/devops-lab/tomcat/webapps/manager/META-INF/context.xml
```

If you see:

```xml
allow="127.0.0.0/8,::1/128"
```

the Manager is restricted.

For this local lab:

```bash
sed -i 's/allow="127.0.0.0\/8,::1\/128"/allow="0.0.0.0\/0,::\/0"/' \
/data/devops-lab/tomcat/webapps/manager/META-INF/context.xml
```

Host Manager:

```bash
sed -i 's/allow="127.0.0.0\/8,::1\/128"/allow="0.0.0.0\/0,::\/0"/' \
/data/devops-lab/tomcat/webapps/host-manager/META-INF/context.xml
```

Recreate:

```bash
docker compose up -d --force-recreate tomcat
```

---

# 38. Tomcat Manager 401 Troubleshooting

If:

```text
401 Unauthorized
```

occurs, verify the actual file being used:

```bash
docker exec tomcat cat /usr/local/tomcat/conf/tomcat-users.xml
```

If it contains only the default commented examples, your customized file is not mounted.

Check:

```bash
docker inspect tomcat \
  --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

You must see:

```text
/data/devops-lab/tomcat/conf -> /usr/local/tomcat/conf
```

Then:

```bash
docker compose up -d --force-recreate tomcat
```

Verify again:

```bash
docker exec tomcat cat /usr/local/tomcat/conf/tomcat-users.xml
```

---

# 39. Check All Services

```bash
docker compose ps
```

Expected:

```text
jenkins     Up
sonarqube   Up
nexus       Up
tomcat      Up
```

---

# 40. Check Logs

All:

```bash
docker compose logs -f
```

Jenkins:

```bash
docker compose logs -f jenkins
```

SonarQube:

```bash
docker compose logs -f sonarqube
```

Nexus:

```bash
docker compose logs -f nexus
```

Tomcat:

```bash
docker compose logs -f tomcat
```

Last 50 lines:

```bash
docker compose logs --tail=50
```

---

# 41. Check Resource Usage

```bash
docker stats
```

Jenkins, SonarQube and Nexus can consume significant memory.

---

# 42. Check Ports

```bash
sudo ss -lntp | grep -E '8050|8051|8052|8053|50000'
```

Expected:

```text
8050
8051
8052
8053
50000
```

---

# 43. Start / Stop / Restart

Start:

```bash
docker compose up -d
```

Stop:

```bash
docker compose stop
```

Start stopped containers:

```bash
docker compose start
```

Restart:

```bash
docker compose restart
```

Restart Tomcat:

```bash
docker compose restart tomcat
```

---

# 44. Shutdown Without Deleting Data

```bash
docker compose down
```

This removes containers and the Compose network.

It does not remove the host data:

```text
/data/devops-lab
```

Start again:

```bash
docker compose up -d
```

---

# 45. Reboot Recovery

Because every service uses:

```yaml
restart: unless-stopped
```

Docker will automatically restart the containers after Docker starts.

Check after reboot:

```bash
docker compose ps
```

If necessary:

```bash
cd /data/devops-lab
docker compose up -d
```

No Tomcat files need to be copied again.

---

# 46. Persistent Storage

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
    ├── conf/
    │   ├── server.xml
    │   ├── tomcat-users.xml
    │   └── ...
    │
    ├── logs/
    │
    └── webapps/
        ├── ROOT/
        ├── manager/
        └── host-manager/
```

---

# 47. Important Data Safety

This setup uses host bind mounts.

Important data is stored under:

```text
/data/devops-lab
```

Do not run:

```bash
sudo rm -rf /data/devops-lab
```

unless you intentionally want to delete the entire lab.

Do not use:

```bash
docker compose down -v
```

as this setup is based on host bind mounts and should not need volume deletion.

---

# 48. Final Port Mapping

```text
Host                         Container
──────────────────────────────────────────────
localhost:8050       ──────→ Jenkins :8080

localhost:8051       ──────→ SonarQube :9000

localhost:8052       ──────→ Nexus :8081

localhost:8053       ──────→ Tomcat :8080

localhost:50000      ──────→ Jenkins :50000
```

---

# 49. Internal Docker URLs

Containers communicate using Docker service/container names.

From Jenkins:

```text
SonarQube:
http://sonarqube:9000

Nexus:
http://nexus:8081

Tomcat:
http://tomcat:8080

Tomcat Manager:
http://tomcat:8080/manager/text
```

Do not use host `localhost` URLs from inside containers.

---

# 50. Final DevOps CI/CD Architecture

```text
                    Git Repository
                           │
                           ▼
                       Jenkins
                      Host :8050
                           │
                           ▼
                    Maven Build
                           │
                           ▼
                      Unit Tests
                           │
                           ▼
                       SonarQube
                      Host :8051
                           │
                           ▼
                      Quality Gate
                           │
                           ▼
                    Package WAR/JAR
                           │
                           ▼
                        Nexus
                      Host :8052
                           │
                           ▼
                   Deploy Application
                           │
                           ▼
                       Tomcat
                      Host :8053
                           │
                           ▼
                     Application
```

Internal communication:

```text
Jenkins
   │
   ├── SonarQube → http://sonarqube:9000
   │
   ├── Nexus     → http://nexus:8081
   │
   └── Tomcat    → http://tomcat:8080
```

---

# 51. Jenkins → Tomcat Deployment

Dedicated Tomcat deployment account:

```text
Username: jenkins-deploy
Password: Jenkins#123
Role: manager-script
```

Deployment endpoint:

```text
http://tomcat:8080/manager/text
```

Typical flow:

```text
Jenkins
   │
   ├── Checkout source
   │
   ├── Maven build
   │
   ├── Unit tests
   │
   ├── SonarQube scan
   │
   ├── Quality Gate
   │
   ├── Upload artifact to Nexus
   │
   └── Deploy WAR
          │
          ▼
       Tomcat
```

---

# 52. Quick Health Check

Check containers:

```bash
docker compose ps
```

Check Jenkins:

```bash
curl -I http://localhost:8050
```

Check SonarQube:

```bash
curl -I http://localhost:8051
```

Check Nexus:

```bash
curl -I http://localhost:8052
```

Check Tomcat:

```bash
curl -I http://localhost:8053
```

Test Tomcat Manager:

```bash
curl -u 'admin:Admin#123456' \
  http://localhost:8053/manager/text/list
```

Test Jenkins deployment account:

```bash
curl -u 'jenkins-deploy:Jenkins#123' \
  http://localhost:8053/manager/text/list
```

---

# 53. Final Verification Checklist

Run each item:

```bash
docker --version
```

```bash
docker compose version
```

```bash
docker ps
```

```bash
docker compose config
```

```bash
docker compose ps
```

```bash
docker network inspect devops-net
```

```bash
docker inspect tomcat \
  --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

```bash
docker exec tomcat \
  cat /usr/local/tomcat/conf/tomcat-users.xml
```

```bash
docker exec tomcat \
  ls -la /usr/local/tomcat/webapps
```

```bash
curl -u 'admin:Admin#123456' \
  http://localhost:8053/manager/text/list
```

```bash
curl -u 'jenkins-deploy:Jenkins#123' \
  http://localhost:8053/manager/text/list
```

From Jenkins:

```bash
docker exec jenkins \
  curl -u 'jenkins-deploy:Jenkins#123' \
  http://tomcat:8080/manager/text/list
```

Expected final service state:

```text
jenkins     Up
sonarqube   Up
nexus       Up
tomcat      Up
```

---

# 54. Final Environment

```text
┌──────────────────────────────────────────────────────────┐
│                     Ubuntu Host                          │
│                                                          │
│                    Docker Engine                         │
│                                                          │
│                   devops-net (bridge)                    │
│                                                          │
│   ┌──────────┐    ┌───────────┐    ┌──────────┐         │
│   │ Jenkins  │    │ SonarQube │    │  Nexus   │         │
│   │  :8080   │    │   :9000   │    │  :8081   │         │
│   └────┬─────┘    └───────────┘    └──────────┘         │
│        │                                                 │
│        │                                                 │
│   ┌────▼─────┐                                          │
│   │  Tomcat  │                                          │
│   │  :8080   │                                          │
│   └──────────┘                                          │
│                                                          │
│ Host Ports:                                             │
│                                                          │
│ 8050 → Jenkins                                           │
│ 8051 → SonarQube                                         │
│ 8052 → Nexus                                             │
│ 8053 → Tomcat                                            │
│ 50000 → Jenkins Agent                                    │
│                                                          │
│ Persistent Data:                                        │
│ /data/devops-lab                                        │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

The lab is now ready for a complete:

```text
Jenkins
   ↓
Maven
   ↓
Unit Tests
   ↓
SonarQube
   ↓
Quality Gate
   ↓
Nexus
   ↓
Tomcat
```

CI/CD pipeline.
