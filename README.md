# Python Code Library App

A digital library built as Python microservices (Flask + MySQL), packaged with Docker and delivered through a Jenkins CI/CD pipeline with SonarQube code analysis, Docker Hub image publishing, and deployment to a Docker Swarm cluster on AWS EC2.

**Author:** Kiran Subrahamanya Sai ([@Kiransai006](https://github.com/Kiransai006))

---

## Table of Contents

1. [Architecture](#architecture)
2. [Repository structure](#repository-structure)
3. [Run locally with Docker Compose](#run-locally-with-docker-compose)
4. [CI/CD pipeline](#cicd-pipeline)
5. [Infrastructure](#infrastructure)
6. [Setup guide](#setup-guide)
7. [Deploying and verifying](#deploying-and-verifying)
8. [Troubleshooting](#troubleshooting)
9. [Security notes](#security-notes)
10. [Acknowledgements](#acknowledgements)

---

## Architecture

| Service | Folder | Port | Purpose |
|---|---|---|---|
| `frontend` | `./` (`app.py`, `templates/`) | 5000 | Web UI (sign up, sign in, browse and borrow books) |
| `auth_service` | `auth/` | 5001 | User sign-up and sign-in API |
| `book_service` | `book/` | 5002 | Book catalogue API |
| `borrow_service` | `borrow/` | 5003 | Borrow records API |
| `db` | `database/` | 3306 | MySQL 5.7 with the `digital_library` schema and sample data |

The frontend calls the three API services by service name over a shared Docker network. The API services connect to MySQL using the `DB_HOST`, `DB_USER`, `DB_PASSWORD` and `DB_NAME` environment variables (defaults: `db`, `root`, `root`, `digital_library`).

The database service must be named `db`, because the Python services use that name as their default database host.

---

## Repository structure

```
.
├── app.py                    # Frontend (Flask)
├── Dockerfile                # Frontend image
├── requirements.txt
├── templates/                # HTML templates
├── auth/                     # Auth service + Dockerfile
├── book/                     # Book service + Dockerfile
├── borrow/                   # Borrow service + Dockerfile
├── database/                 # MySQL image + database.sql
├── docker-compose.yml        # Single-host build and run (also used by CI)
├── docker-stack.yml          # Docker Swarm stack definition
├── Jenkinsfile               # Jenkins pipeline definition
└── sonar-project.properties  # SonarQube settings (optional)
```

---

## Run locally with Docker Compose

Requirements: Docker with the Compose plugin.

```bash
git clone https://github.com/Kiransai006/python-code-library-app.git
cd python-code-library-app
docker compose up -d --build
docker compose ps
```

MySQL needs about 30 seconds to initialise. Then open `http://localhost:5000`, sign up, sign in and borrow a book.

Quick API check:

```bash
curl http://localhost:5002/books
```

Stop and remove everything (including the database volume):

```bash
docker compose down -v
```

---

## CI/CD pipeline

The pipeline runs on Jenkins and has these stages:

| # | Stage | What it does |
|---|---|---|
| 1 | Checkout | Clones the repository from GitHub (`main` branch) |
| 2 | Build | Creates a Python virtual environment, installs dependencies, byte-compiles the code |
| 3 | Test | Runs `flake8` (syntax errors) and `pytest` with coverage, and publishes JUnit results |
| 4 | SonarQube Analysis | Runs `sonar-scanner` and uploads the report to SonarQube |
| 5 | Quality Gate | Reads the SonarQube task result and fails the build if the gate is not `OK` |
| 6 | Package Artifacts | Creates a `.tar.gz` of the source and archives it in Jenkins together with coverage and test reports |
| 7 | Docker Build | `docker compose build` builds all five images and tags them `<service>-<build number>` |
| 8 | Compose Smoke Test | Starts the full stack with Compose and checks the book API and the web UI |
| 9 | Docker Push | Pushes the five images to Docker Hub |
| 10 | Deploy to Swarm | Runs on the Swarm manager node: `docker stack deploy`, then waits until every service is fully running |
| 11 | Verify App | Calls the application on the manager's private IP to confirm it answers |

Image naming on Docker Hub:

```
kiransai006/python-code-library-app:db-<build>
kiransai006/python-code-library-app:auth-<build>
kiransai006/python-code-library-app:book-<build>
kiransai006/python-code-library-app:borrow-<build>
kiransai006/python-code-library-app:frontend-<build>
```

Each build deploys a new set of tags with a rolling update (`start-first`, automatic rollback on failure).

---

## Infrastructure

Four AWS EC2 instances, all in the same VPC:

| Instance | Role |
|---|---|
| `jenkins` | Jenkins controller, SonarQube (Docker container, port 9000), Python, Docker and Compose for build and smoke tests |
| `manager` | Docker Swarm manager, also a Jenkins agent with the label `manager` |
| `slave` | Docker Swarm worker |
| `app` | Used for the first manual deployment with Docker Compose |

### Network ports (security groups)

| Port | Protocol | Between | Purpose |
|---|---|---|---|
| 22 | TCP | Jenkins to manager | SSH for the Jenkins agent |
| 8080 | TCP | You, and SonarQube to Jenkins | Jenkins UI and the SonarQube webhook |
| 9000 | TCP | You | SonarQube UI |
| 5000 | TCP | You | Application UI |
| 2377 | TCP | manager and slave | Swarm cluster management |
| 7946 | TCP and UDP | manager and slave | Swarm node communication |
| 4789 | UDP | manager and slave | Swarm overlay network |

---

## Setup guide

### 1. Jenkins instance

```bash
sudo dnf install -y python3 python3-pip git docker
sudo systemctl enable --now docker
sudo usermod -aG docker jenkins

# Docker Compose plugin
sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 \
  -o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose

# Docker Buildx (Compose build needs 0.17.0 or newer)
sudo curl -fSL https://github.com/docker/buildx/releases/download/v0.19.3/buildx-v0.19.3.linux-amd64 \
  -o /usr/local/lib/docker/cli-plugins/docker-buildx
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-buildx
sudo mkdir -p /var/lib/jenkins/.docker/cli-plugins
sudo cp /usr/local/lib/docker/cli-plugins/docker-buildx /var/lib/jenkins/.docker/cli-plugins/
sudo chown -R jenkins:jenkins /var/lib/jenkins/.docker

sudo systemctl restart jenkins
sudo -u jenkins docker compose version
sudo -u jenkins docker buildx version
```

If the instance has little memory, add swap before starting SonarQube:

```bash
sudo dd if=/dev/zero of=/swapfile bs=1M count=2048 status=progress
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab
```

### 2. SonarQube (on the Jenkins instance)

```bash
sudo sysctl -w vm.max_map_count=262144
echo 'vm.max_map_count=262144' | sudo tee -a /etc/sysctl.conf

docker run -d --name sonarqube --restart unless-stopped \
  -p 9000:9000 \
  -v sonar_data:/opt/sonarqube/data \
  -v sonar_extensions:/opt/sonarqube/extensions \
  -v sonar_logs:/opt/sonarqube/logs \
  sonarqube:lts-community
```

Open `http://<jenkins-ip>:9000`, log in with `admin` / `admin`, change the password, and generate a **Global Analysis Token** (My Account, Security).

### 3. Jenkins configuration

**Plugins:** SonarQube Scanner, Pipeline, Git, JUnit, Credentials Binding.

**Credentials** (Manage Jenkins, Credentials, Global):

| ID | Kind | Value |
|---|---|---|
| `dockerhub-creds` | Username with password | Docker Hub username and an access token |
| `sonar-token` | Secret text | SonarQube analysis token |
| `manager-ssh` | SSH username with private key | `ec2-user` and the manager's key |

**SonarQube server** (Manage Jenkins, System): name `sonarqube`, URL `http://<jenkins-private-ip>:9000`, token `sonar-token`.

**Scanner tool** (Manage Jenkins, Tools): name `sonar-scanner`, install automatically.

**Manager node** (Manage Jenkins, Nodes, New Node): name `manager`, permanent agent, label `manager`, launch via SSH with the `manager-ssh` credential, remote root `/home/ec2-user/jenkins-agent`. The manager instance needs Java (`sudo dnf install -y java-17-amazon-corretto`) and its user must be in the `docker` group.

### 4. Docker Swarm

On the manager:

```bash
docker swarm init --advertise-addr <manager-private-ip>
docker swarm join-token worker
```

On the slave, run the printed join command, then verify on the manager:

```bash
docker node ls
```

Both nodes must show `Ready`.

### 5. Pipeline job

1. In Jenkins, create a **Pipeline** job named `python-code-library-app`.
2. Use the contents of the `Jenkinsfile` as the pipeline script, or choose "Pipeline script from SCM" with this repository and branch `main`.
3. Run **Build Now**.

---

## Deploying and verifying

On the manager, after a successful build:

```bash
docker stack services codelib
docker stack ps codelib
```

Expected: `auth_service`, `book_service`, `borrow_service` and `frontend` at `2/2`, and `db` at `1/1`.

Open `http://<manager-public-ip>:5000`, sign up, sign in, borrow a book and check the "My books" page.

Useful commands:

```bash
docker service logs codelib_frontend --tail 50     # service logs
docker service scale codelib_frontend=3            # scale a service
docker stack rm codelib                            # remove the stack
```

Deployment details: the database is pinned to the manager node (its data volume is local to that node), the four application services run two replicas each, and only port 5000 is published.

---

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `Couldn't find any revision to build` | Wrong branch name in the pipeline. Use `main` (or your actual default branch) |
| Quality Gate stage waits until timeout | The SonarQube webhook cannot reach Jenkins. The pipeline reads the task result from the SonarQube API instead of waiting for the webhook |
| `compose build requires buildx 0.17.0 or later` | Install a newer buildx for the `jenkins` user (see setup step 1) |
| `permission denied ... docker.sock` | The agent or Jenkins user is not in the `docker` group. Add it and restart the agent |
| Agent cannot copy `remoting.jar` | The remote root folder on the node has the wrong owner. `chown` it to `ec2-user` |
| Deploy stage hangs on a `curl` check | `curl` has no timeout. Use `curl --max-time 5`, and test the manager from the Jenkins instance using the manager's private IP |
| Services stay at `0/2` | Check `docker stack ps codelib --no-trunc` for the error. Common causes: not enough memory, wrong image tag, blocked ports |
| MySQL container is killed | Memory limit too low. The stack gives `db` a 512M limit |

---

## Security notes

This is a learning project. Before using it in production, address these items:

- Run Flask with `debug=False` and read `secret_key` from an environment variable.
- Store database credentials as Docker Swarm secrets instead of the default `root` / `root`.
- Hash passwords in the `users` table (for example with `werkzeug.security`).
- Do not publish MySQL (3306) or the API ports (5001 to 5003) outside the Docker network.
- Restrict security group rules to your own IP or the group itself, and put the app behind HTTPS.
- Add automated tests so the Test stage and the SonarQube coverage metric are meaningful.

---

## Acknowledgements

Originally based on the open-source `devops0014/python-code-library-app` project, extended with a Jenkins CI/CD pipeline, SonarQube analysis, Docker Hub publishing and Docker Swarm deployment.
