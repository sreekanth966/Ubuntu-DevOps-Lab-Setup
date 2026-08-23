# Ubuntu Docker DevOps Lab

Complete local DevOps lab setup using Docker Compose:

- Jenkins
- SonarQube
- Nexus Repository
- Apache Tomcat
- Custom Docker bridge network
- Persistent storage under `/data/devops-lab`
- Host ports starting from `8050`
- Tomcat Manager and Host Manager
- Tomcat users for administration and Jenkins deployment

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

Create the keyring directory:

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

# 7. Verify Docker Installation

Check Docker:

```bash
docker --version
```

Check Compose:

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

Check the Docker group:

```bash
getent group docker
```

You should see your current user in the group.

For example:

```text
docker:x:<gid>:$USER
```

## Apply the new group membership

Recommended: log out of Ubuntu and log back in.

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

## If `newgrp` is not available

Install:

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

Give the current user ownership:

```bash
sudo chown -R $USER:$USER /data/devops-lab
```

Create all application directories:

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

Move into the lab directory:

```bash
cd /data/devops-lab
```

---

# 10. Directory Structure

Verify:

```bash
tree /data/devops-lab
```

If `tree` is not installed:

```bash
sudo apt install -y tree
```

Initial structure:

```text
/data/devops-lab
├── jenkins
├── nexus
├── sonarqube
│   ├── data
│   ├── extensions
│   └── logs
└── tomcat
    ├── conf
    ├── logs
    └── webapps
```

Final structure:

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
    ├── conf
    ├── logs
    └── webapps
        ├── ROOT
        ├── manager
        └── host-manager
```

---

# 11. Create Docker Compose File

Create:

```bash
nano /data/devops-lab/docker-compose.yml
```

Use this complete configuration:

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
      # Persistent Tomcat configuration
      - /data/devops-lab/tomcat/conf:/usr/local/tomcat/conf

      # Persistent web applications
      - /data/devops-lab/tomcat/webapps:/usr/local/tomcat/webapps

      # Persistent Tomcat logs
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

```bash
cd /data/devops-lab
docker compose config
```

The command must complete without errors.

---

# 13. Pull Images

```bash
docker compose pull
```

Check:

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

Expected:

```text
jenkins
sonarqube
nexus
tomcat
```

All should eventually show:

```text
Up
```

---

# 15. Recreate Services After Compose Changes

If `docker-compose.yml` changes:

```bash
docker compose up -d --force-recreate
```

This is especially important when adding or changing bind mounts.

---

# 16. Check Docker Network

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

The following containers should be connected:

```text
jenkins
sonarqube
nexus
tomcat
```

---

# 17. Application URLs

| Application | URL |
|---|---|
| Jenkins | http://localhost:8050 |
| SonarQube | http://localhost:8051 |
| Nexus | http://localhost:8052 |
| Tomcat | http://localhost:8053 |

---

# 18. Jenkins Initial Password

Get the initial administrator password:

```bash
docker exec jenkins \
  cat /var/jenkins_home/secrets/initialAdminPassword
```

Open:

```text
http://localhost:8050
```

Use the password returned by the command.

Example:

```text
Username: admin
Password: <initial password returned above>
```

---

# 19. SonarQube Initial Login

Open:

```text
http://localhost:8051
```

Initial credentials:

```text
Username: admin
Password: admin
```

After the initial login, change the password if the environment will be accessible beyond the local lab.

---

# 20. Nexus Initial Login

Open:

```text
http://localhost:8052
```

Nexus uses the administrator password generated during initialization.

Try:

```bash
docker exec nexus cat /nexus-data/admin.password
```

If the file exists, use the returned password.

If Nexus has already completed its initial setup and the password file no longer exists, use the administrator credentials configured during the Nexus setup.

---

# 21. Tomcat Default Welcome Page

Because the following host directory is bind-mounted:

```text
/data/devops-lab/tomcat/webapps
```

it initially hides the image's original Tomcat `webapps` directory.

If the directory is empty, copy the default ROOT application:

```bash
docker cp tomcat:/usr/local/tomcat/webapps.dist/ROOT \
  /data/devops-lab/tomcat/webapps/
```

Verify:

```bash
ls -la /data/devops-lab/tomcat/webapps
```

You should see:

```text
ROOT/
```

Restart:

```bash
docker compose restart tomcat
```

Open:

```text
http://localhost:8053
```

The Tomcat welcome page should now appear.

---

# 22. Install Tomcat Manager and Host Manager

The default Tomcat image keeps the Manager applications under:

```text
/usr/local/tomcat/webapps.dist/
```

Copy Manager:

```bash
docker cp tomcat:/usr/local/tomcat/webapps.dist/manager \
  /data/devops-lab/tomcat/webapps/
```

Copy Host Manager:

```bash
docker cp tomcat:/usr/local/tomcat/webapps.dist/host-manager \
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

# 23. Create Persistent Tomcat Configuration

The Tomcat configuration is stored under:

```text
/data/devops-lab/tomcat/conf
```

If the directory is empty, copy the original configuration from the running container:

```bash
docker cp tomcat:/usr/local/tomcat/conf \
  /data/devops-lab/tomcat/
```

Verify:

```bash
ls -la /data/devops-lab/tomcat/conf
```

You should see:

```text
server.xml
web.xml
tomcat-users.xml
tomcat-users.xsd
context.xml
...
```

---

# 24. Configure Tomcat Users

Create/update:

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

---

# 25. Tomcat User Details

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

The `admin` user can access:

```text
Manager App
Server Status
Host Manager
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

This user is intended for automated Jenkins WAR deployments.

Do not use the administrator account for Jenkins deployment when the dedicated deployment account is sufficient.

> These passwords are for this local lab setup only. Do not use these credentials in production.

---

# 26. Configure Tomcat Manager Access Through Docker

Tomcat 10.1 uses `RemoteCIDRValve` in the Manager and Host Manager applications.

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

The default configuration may contain:

```xml
<Valve className="org.apache.catalina.valves.RemoteCIDRValve"
       allow="127.0.0.0/8,::1/128" />
```

This can cause `403` when accessing the Manager through Docker.

For this isolated local lab, allow Docker/network access:

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

And:

```bash
grep -nA1 "RemoteCIDRValve" \
/data/devops-lab/tomcat/webapps/host-manager/META-INF/context.xml
```

Expected:

```xml
<Valve className="org.apache.catalina.valves.RemoteCIDRValve"
       allow="0.0.0.0/0,::/0" />
```

> `0.0.0.0/0,::/0` allows access from any reachable address. This is suitable only for an isolated local lab. For production, restrict access to trusted IP/CIDR ranges.

---

# 27. Restart Tomcat After Configuration Changes

```bash
docker compose restart tomcat
```

If the Compose file itself changed, use:

```bash
docker compose up -d --force-recreate tomcat
```

---

# 28. Verify Tomcat Persistent Mounts

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

This is important because the `conf` mount ensures your persistent `tomcat-users.xml` is actually used by Tomcat.

---

# 29. Verify Tomcat Users Inside Container

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

If the container still shows the original default Tomcat users file, check the mounts:

```bash
docker inspect tomcat \
  --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

---

# 30. Test Tomcat Manager Authentication

Do not use only `curl -I` for authentication testing.

Test the Manager text interface:

```bash
curl -u 'admin:Admin#123456' \
  http://localhost:8053/manager/text/list
```

Expected:

```text
OK - Listed applications for virtual host localhost
```

Test the Jenkins deployment user:

```bash
curl -u 'jenkins-deploy:Jenkins#123' \
  http://localhost:8053/manager/text/list
```

Expected:

```text
OK - Listed applications for virtual host localhost
```

---

# 31. Test Tomcat Manager From Jenkins

This verifies the actual Jenkins-to-Tomcat Docker network path.

Enter the Jenkins container:

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

The internal Jenkins deployment URL is:

```text
http://tomcat:8080/manager/text
```

Do not use:

```text
http://localhost:8053/manager/text
```

from inside Jenkins because `localhost` means the Jenkins container.

---

# 32. Tomcat URLs

## Main Tomcat

```text
http://localhost:8053
```

## Server Status

```text
http://localhost:8053/manager/status
```

## Manager App

```text
http://localhost:8053/manager/html
```

## Host Manager

```text
http://localhost:8053/host-manager/html
```

Login:

```text
Username: admin
Password: Admin#123456
```

---

# 33. Nexus Permission Fix

Nexus runs as a non-root user inside the container.

Check its UID/GID:

```bash
docker run --rm sonatype/nexus3:latest id nexus
```

Expected to be similar to:

```text
uid=200(nexus) gid=200(nexus) groups=200(nexus)
```

If Nexus has permission errors on `/nexus-data`, stop it:

```bash
docker compose stop nexus
```

Set ownership using the UID/GID returned above.

For the standard Nexus image:

```bash
sudo chown -R 200:200 /data/devops-lab/nexus
```

Set reasonable permissions:

```bash
sudo chmod -R u+rwX,go-rwx /data/devops-lab/nexus
```

Start Nexus:

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

# 34. Tomcat 404 Troubleshooting

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

If the directory is empty, Tomcat has no ROOT application because the bind mount hides the image's original `webapps` directory.

Copy ROOT:

```bash
docker cp tomcat:/usr/local/tomcat/webapps.dist/ROOT \
  /data/devops-lab/tomcat/webapps/
```

Restart:

```bash
docker compose restart tomcat
```

---

# 35. Tomcat Manager 403 Troubleshooting

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

the Manager is restricted to localhost inside the container.

For the local lab:

```bash
sed -i 's/allow="127.0.0.0\/8,::1\/128"/allow="0.0.0.0\/0,::\/0"/' \
/data/devops-lab/tomcat/webapps/manager/META-INF/context.xml
```

Do the same for Host Manager:

```bash
sed -i 's/allow="127.0.0.0\/8,::1\/128"/allow="0.0.0.0\/0,::\/0"/' \
/data/devops-lab/tomcat/webapps/host-manager/META-INF/context.xml
```

Restart:

```bash
docker compose restart tomcat
```

---

# 36. Tomcat Manager 401 Troubleshooting

If you get:

```text
401 Unauthorized
```

after fixing the `RemoteCIDRValve`, verify the user file inside the container:

```bash
docker exec tomcat cat /usr/local/tomcat/conf/tomcat-users.xml
```

If it shows only the original commented example users, the `conf` bind mount is missing.

Check:

```bash
docker inspect tomcat \
  --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

You must have:

```text
/data/devops-lab/tomcat/conf -> /usr/local/tomcat/conf
```

Then recreate Tomcat:

```bash
docker compose up -d --force-recreate tomcat
```

Verify again:

```bash
docker exec tomcat cat /usr/local/tomcat/conf/tomcat-users.xml
```

---

# 37. Check All Services

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

# 38. Check Logs

All services:

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

Latest 50 lines:

```bash
docker compose logs --tail=50
```

---

# 39. Check Resource Usage

```bash
docker stats
```

This is useful because Jenkins, SonarQube and Nexus can consume significant memory.

---

# 40. Check Ports

```bash
sudo ss -lntp | grep -E '8050|8051|8052|8053|50000'
```

Expected host ports:

```text
8050
8051
8052
8053
50000
```

---

# 41. Start and Stop

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

Restart one service:

```bash
docker compose restart tomcat
```

---

# 42. Shutdown Without Deleting Persistent Data

```bash
docker compose down
```

This removes containers and the Compose network but keeps:

```text
/data/devops-lab
```

Start again:

```bash
docker compose up -d
```

---

# 43. Do Not Delete Persistent Data Accidentally

This setup uses host bind mounts rather than Docker-managed named volumes.

The important persistent data is under:

```text
/data/devops-lab
```

Do not run:

```bash
sudo rm -rf /data/devops-lab
```

unless you intentionally want to delete the entire lab.

---

# 44. Final Persistent Storage

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

# 45. Final Port Mapping

```text
Host                         Container
────────────────────────────────────────────────
localhost:8050       ──────→ Jenkins :8080

localhost:8051       ──────→ SonarQube :9000

localhost:8052       ──────→ Nexus :8081

localhost:8053       ──────→ Tomcat :8080

localhost:50000      ──────→ Jenkins :50000
```

---

# 46. Internal Docker URLs

From Jenkins or another container on `devops-net`:

```text
Jenkins:
http://jenkins:8080

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

# 47. Final DevOps CI/CD Architecture

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

Jenkins communicates internally through:

```text
SonarQube → http://sonarqube:9000
Nexus     → http://nexus:8081
Tomcat    → http://tomcat:8080
```

---

# 48. Jenkins → Tomcat Deployment

Use the dedicated Tomcat account:

```text
Username: jenkins-deploy
Password: Jenkins#123
Role: manager-script
```

Jenkins deployment endpoint:

```text
http://tomcat:8080/manager/text
```

The deployment flow will be:

```text
Jenkins
   │
   ├── Build WAR
   │
   ├── Run tests
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

# 49. Daily Commands

Go to the lab:

```bash
cd /data/devops-lab
```

Check services:

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

Validate Compose:

```bash
docker compose config
```

---

# 50. Quick Health Check

Run:

```bash
docker compose ps
```

Then:

```bash
curl -I http://localhost:8050
```

```bash
curl -I http://localhost:8051
```

```bash
curl -I http://localhost:8052
```

```bash
curl -I http://localhost:8053
```

Test Tomcat Manager authentication:

```bash
curl -u 'admin:Admin#123456' \
  http://localhost:8053/manager/text/list
```

Test Jenkins deployment authentication:

```bash
curl -u 'jenkins-deploy:Jenkins#123' \
  http://localhost:8053/manager/text/list
```

---

# 51. Final Environment

The completed local DevOps environment is:

```text
┌─────────────────────────────────────────────────────┐
│                    Ubuntu Host                      │
│                                                     │
│                  Docker Engine                      │
│                                                     │
│                 devops-net (bridge)                 │
│                                                     │
│   ┌──────────┐   ┌───────────┐   ┌──────────┐      │
│   │ Jenkins  │   │ SonarQube │   │  Nexus   │      │
│   │  :8080   │   │   :9000   │   │  :8081   │      │
│   └────┬─────┘   └───────────┘   └──────────┘      │
│        │                                            │
│        │                                            │
│   ┌────▼─────┐                                     │
│   │  Tomcat  │                                     │
│   │  :8080   │                                     │
│   └──────────┘                                     │
│                                                     │
│ Host Ports:                                        │
│ 8050 → Jenkins                                     │
│ 8051 → SonarQube                                   │
│ 8052 → Nexus                                       │
│ 8053 → Tomcat                                      │
│                                                     │
└─────────────────────────────────────────────────────┘
```

Persistent storage:

```text
/data/devops-lab
```

Custom network:

```text
devops-net
```

The environment is ready for building a complete Jenkins → Maven → SonarQube → Nexus → Tomcat CI/CD pipeline.
