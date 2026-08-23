# Docker DevOps Lab --- Jenkins, SonarQube, Nexus & Tomcat

## 1. Architecture

``` text
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

Host Ports
──────────

8050 → Jenkins
8051 → SonarQube
8052 → Nexus
8053 → Tomcat
```

## 2. Prerequisites

Ubuntu system with:

-   Internet connectivity
-   `sudo` access
-   Minimum 8 GB RAM
-   Recommended 16 GB+ RAM
-   At least 50 GB free disk space

## 3. Update Ubuntu

``` bash
sudo apt update
sudo apt upgrade -y
```

Install required packages:

``` bash
sudo apt install -y ca-certificates curl gnupg
```

## 4. Remove Old Docker Packages

``` bash
sudo apt remove -y \
  docker.io \
  docker-doc \
  docker-compose \
  podman-docker \
  containerd \
  runc
```

## 5. Add Docker Repository

Create the keyring directory:

``` bash
sudo install -m 0755 -d /etc/apt/keyrings
```

Download Docker's GPG key:

``` bash
sudo curl -fsSL \
  https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
```

Set permissions:

``` bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add Docker repository:

``` bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Update package information:

``` bash
sudo apt update
```

## 6. Install Docker Engine and Compose

``` bash
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

## 7. Verify Docker Installation

``` bash
docker --version
docker compose version
sudo systemctl status docker
```

Enable and start Docker:

``` bash
sudo systemctl enable --now docker
```

## 8. Configure Docker Without sudo

``` bash
sudo usermod -aG docker $USER
newgrp docker
```

Verify:

``` bash
docker ps
groups
```

If Docker still requires `sudo`, log out and log back in.

## 9. Create DevOps Lab Directory

``` bash
mkdir -p ~/devops-lab
cd ~/devops-lab
```

## 10. Create Docker Compose File

``` bash
nano docker-compose.yml
```

Paste:

``` yaml
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
      - jenkins_home:/var/jenkins_home

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
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs

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
      - nexus_data:/nexus-data

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
      - tomcat_webapps:/usr/local/tomcat/webapps
      - tomcat_logs:/usr/local/tomcat/logs

    networks:
      - devops-net

    environment:
      TZ: Asia/Kolkata


# ============================================================
# Custom Bridge Network
# ============================================================
networks:

  devops-net:
    name: devops-net
    driver: bridge


# ============================================================
# Persistent Volumes
# ============================================================
volumes:

  jenkins_home:

  sonarqube_data:
  sonarqube_extensions:
  sonarqube_logs:

  nexus_data:

  tomcat_webapps:
  tomcat_logs:
```

## 11. Validate Docker Compose

``` bash
docker compose config
```

## 12. Pull Images

``` bash
docker compose pull
```

## 13. Start All Services

``` bash
docker compose up -d
```

Check:

``` bash
docker compose ps
docker ps
```

Expected services:

``` text
jenkins
sonarqube
nexus
tomcat
```

## 14. Check Docker Network

``` bash
docker network ls
docker network inspect devops-net
```

The following containers should be connected:

``` text
jenkins
sonarqube
nexus
tomcat
```

## 15. Application URLs

  Application   URL
  ------------- -----------------------
  Jenkins       http://localhost:8050
  SonarQube     http://localhost:8051
  Nexus         http://localhost:8052
  Tomcat        http://localhost:8053

## 16. Jenkins Initial Password

``` bash
docker exec jenkins \
  cat /var/jenkins_home/secrets/initialAdminPassword
```

Open:

``` text
http://localhost:8050
```

Use the password returned by the command.

## 17. Nexus Initial Password

``` bash
docker exec nexus \
  cat /nexus-data/admin.password
```

Open:

``` text
http://localhost:8052
```

Default username:

``` text
admin
```

Use the password returned by the command.

## 18. SonarQube

Open:

``` text
http://localhost:8051
```

Complete the initial SonarQube setup/login.

## 19. Tomcat

Open:

``` text
http://localhost:8053
```

Tomcat listens internally on port `8080` and Docker maps it to host port
`8053`.

## 20. Docker Internal Networking

All four containers use:

``` text
devops-net
```

From Jenkins, use:

``` text
SonarQube → http://sonarqube:9000
Nexus     → http://nexus:8081
Tomcat    → http://tomcat:8080
```

Do not use these from inside Jenkins:

``` text
http://localhost:8051
http://localhost:8052
http://localhost:8053
```

## 21. Test Internal Connectivity

Enter Jenkins:

``` bash
docker exec -it jenkins bash
```

Test SonarQube:

``` bash
curl http://sonarqube:9000
```

Test Nexus:

``` bash
curl http://nexus:8081
```

Test Tomcat:

``` bash
curl http://tomcat:8080
```

Exit:

``` bash
exit
```

## 22. Check Logs

Jenkins:

``` bash
docker logs -f jenkins
```

SonarQube:

``` bash
docker logs -f sonarqube
```

Nexus:

``` bash
docker logs -f nexus
```

Tomcat:

``` bash
docker logs -f tomcat
```

All services:

``` bash
docker compose logs -f
```

## 23. Check Resource Usage

``` bash
docker stats
```

## 24. Start and Stop

Stop:

``` bash
docker compose stop
```

Start:

``` bash
docker compose start
```

Restart everything:

``` bash
docker compose restart
```

Restart Jenkins:

``` bash
docker compose restart jenkins
```

Restart SonarQube:

``` bash
docker compose restart sonarqube
```

Restart Nexus:

``` bash
docker compose restart nexus
```

Restart Tomcat:

``` bash
docker compose restart tomcat
```

## 25. Shutdown

Stop and remove containers:

``` bash
docker compose down
```

Persistent volumes are retained.

Start again:

``` bash
docker compose up -d
```

## 26. Completely Delete the Lab

Warning: this deletes persistent application data.

``` bash
docker compose down -v
```

This removes:

``` text
Containers
Network
Jenkins data
SonarQube data
Nexus data
Tomcat data
```

Do not use `-v` unless you intentionally want a clean installation.

## 27. Persistent Volumes

Check volumes:

``` bash
docker volume ls
```

Expected volumes include:

``` text
devops-lab_jenkins_home
devops-lab_sonarqube_data
devops-lab_sonarqube_extensions
devops-lab_sonarqube_logs
devops-lab_nexus_data
devops-lab_tomcat_webapps
devops-lab_tomcat_logs
```

The exact prefix can vary depending on the Compose project name.

## 28. Final Port Mapping

``` text
┌─────────────────────────────────────────────┐
│                 Ubuntu Host                 │
│                                             │
│  8050 ──────────────── Jenkins              │
│  8051 ──────────────── SonarQube            │
│  8052 ──────────────── Nexus                │
│  8053 ──────────────── Tomcat               │
│                                             │
└─────────────────────────────────────────────┘
                       │
                       ▼
              Docker Bridge Network
                  devops-net
                       │
       ┌───────────────┼────────────────┐
       │               │                │
       ▼               ▼                ▼
    Jenkins         SonarQube         Nexus
    :8080             :9000            :8081
       │
       ▼
    Tomcat
    :8080
```

## 29. Jenkins CI/CD Flow

``` text
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
                    :9000 internal
                          │
                          ▼
                    Quality Gate
                          │
                          ▼
                  Package WAR/JAR
                          │
                          ▼
                       Nexus
                    :8081 internal
                          │
                          ▼
                    Deploy Artifact
                          │
                          ▼
                      Tomcat
                    :8080 internal
                          │
                          ▼
                   Application
```

## 30. Quick Installation

After creating `docker-compose.yml`:

``` bash
cd ~/devops-lab

docker compose config

docker compose pull

docker compose up -d

docker compose ps

docker network inspect devops-net
```

Access:

``` text
Jenkins:
http://localhost:8050

SonarQube:
http://localhost:8051

Nexus:
http://localhost:8052

Tomcat:
http://localhost:8053
```

The environment is ready for a Jenkins → Maven → SonarQube → Nexus →
Tomcat CI/CD pipeline.
