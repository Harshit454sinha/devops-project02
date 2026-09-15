# Tasklane — Todo List DevOps Project

A static **Todo List website** (HTML + CSS + Bootstrap 5, no JavaScript, no
backend) used to demonstrate a full **CI/CD pipeline**: Git & GitHub →
Jenkins → Docker → Docker Hub → AWS EC2, with an optional **Kubernetes**
deployment.

This README is written for beginners. Follow it top to bottom and you'll go
from "empty folder" to "live website updated automatically on every git
push."

---

## 1. Project Overview

- **Frontend:** static HTML5 + CSS3, styled with Bootstrap 5. No JavaScript,
  no framework, no database.
- **Containerization:** the site is served by **Nginx** inside a **Docker**
  container.
- **CI/CD:** **Jenkins** watches GitHub via a webhook, builds the Docker
  image, pushes it to **Docker Hub**, and deploys it to an **AWS EC2**
  instance.
- **Orchestration (optional):** the same image can be run on **Kubernetes**
  (Minikube or Docker Desktop) using the manifests in `k8s/`.

## 2. Technologies Used

| Layer            | Technology                     |
|-------------------|--------------------------------|
| UI                | HTML5, CSS3, Bootstrap 5        |
| Version control   | Git, GitHub                     |
| CI/CD             | Jenkins                         |
| Containerization  | Docker, Nginx (alpine)          |
| Image registry    | Docker Hub                      |
| Cloud hosting     | AWS EC2 (Ubuntu)                |
| Orchestration     | Kubernetes (Minikube / Docker Desktop) |

## 3. Architecture

```text
Developer changes index.html
            |
         git push
            |
      GitHub Webhook
            |
          Jenkins
    (Checkout -> Validate -> Docker Build)
            |
       Docker Hub push
            |
         AWS EC2
   (pull image, restart container)
            |
       Live Website  ->  http://EC2-PUBLIC-IP
```

```text
HTML + CSS + Bootstrap
          |
       Docker (Nginx image)
          |
        Nginx
          |
      Port 80
```

## 4. Folder Structure

```text
todo-devops-project/
│
├── index.html          # Todo list UI (static)
├── style.css            # Custom styling on top of Bootstrap
│
├── Dockerfile            # Builds the Nginx image that serves the site
├── .dockerignore          # Files excluded from the Docker build context
├── Jenkinsfile             # Declarative Jenkins pipeline
│
├── k8s/
│   ├── deployment.yaml      # Kubernetes Deployment (2 replicas)
│   └── service.yaml          # Kubernetes Service (NodePort)
│
└── README.md                  # This file
```

---

## 5. Run the Website Locally (no Docker, no server)

Since this is a static site, you can just open it in a browser:

1. Download or clone the project folder.
2. Double-click `index.html`, or right-click → "Open with" your browser.

That's it — no build step, no server required.

---

## 6. Git & GitHub Setup

### 6.1 Install Git

```bash
sudo apt update
sudo apt install git -y
git --version
```

### 6.2 Configure Git (first time only)

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

### 6.3 Create the GitHub repository

1. Go to [github.com](https://github.com) and log in.
2. Click **New repository**.
3. Name it `todo-devops-project`, keep it **Public** (or Private if you
   prefer), do **not** initialize with a README (you already have one).
4. Click **Create repository**.

### 6.4 Push your local project to GitHub

From inside the `todo-devops-project` folder:

```bash
git init
git add .
git commit -m "Initial commit: static todo UI, Docker, Jenkins, k8s config"
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/todo-devops-project.git
git push -u origin main
```

After this, every future change is pushed with:

```bash
git add .
git commit -m "Describe what you changed"
git push
```

---

## 7. Install Docker (on your machine and on the EC2 server)

```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER
# log out and log back in for the group change to take effect
docker --version
```

## 8. Build and Run the Docker Image Locally

From inside `todo-devops-project`:

```bash
# Build the image
docker build -t todo-devops-app:latest .

# Run the container
docker run -d --name todo-app -p 80:80 todo-devops-app:latest

# Visit the site
# http://localhost
```

Stop and remove the container when you're done testing:

```bash
docker stop todo-app
docker rm todo-app
```

---

## 9. Create a Docker Hub Repository

1. Go to [hub.docker.com](https://hub.docker.com) and log in / sign up.
2. Click **Create Repository**.
3. Name it `todo-devops-app` (or any name — just match it in the
   `Jenkinsfile` and `k8s/deployment.yaml`).
4. Set visibility to **Public** so EC2 can pull it without extra auth (or
   keep it private and add a `docker login` step on EC2 too).
5. Click **Create**.

### 9.1 Test pushing manually (optional, sanity check)

```bash
docker login
docker tag todo-devops-app:latest YOUR-DOCKERHUB-USERNAME/todo-devops-app:latest
docker push YOUR-DOCKERHUB-USERNAME/todo-devops-app:latest
```

---

## 10. Launch and Configure the AWS EC2 Instance

### 10.1 Launch the instance

1. In the AWS Console, go to **EC2 → Launch Instance**.
2. Name: `todo-app-server`.
3. AMI: **Ubuntu Server 22.04 LTS**.
4. Instance type: `t2.micro` (free tier eligible).
5. Key pair: create a new key pair (e.g. `todo-app-key.pem`) and download it
   — you'll need it for SSH.
6. **Network settings / Security Group:** allow:
   - **SSH (port 22)** from your IP
   - **HTTP (port 80)** from anywhere (`0.0.0.0/0`)
7. Launch the instance and note its **Public IPv4 address**.

### 10.2 Secure your key and connect

```bash
chmod 400 todo-app-key.pem
ssh -i todo-app-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

### 10.3 Install Docker on EC2

Once connected via SSH:

```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ubuntu
exit
```

Reconnect so the `docker` group applies:

```bash
ssh -i todo-app-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
docker --version
```

### 10.4 Test a manual deployment (optional, sanity check)

```bash
docker pull YOUR-DOCKERHUB-USERNAME/todo-devops-app:latest
docker run -d --name todo-app -p 80:80 YOUR-DOCKERHUB-USERNAME/todo-devops-app:latest
```

Visit `http://YOUR_EC2_PUBLIC_IP` in a browser — you should see the Tasklane
site.

---

## 11. Install and Configure Jenkins

You can run Jenkins on your own machine, on a separate EC2 instance, or as a
Docker container. Below is a simple native install on Ubuntu.

### 11.1 Install Jenkins

```bash
sudo apt update
sudo apt install openjdk-17-jre -y

curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install jenkins -y
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

Jenkins runs on port **8080** by default. Get the initial admin password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Open `http://YOUR_JENKINS_SERVER_IP:8080`, paste the password, install
**suggested plugins**, and create your admin user.

### 11.2 Install required Jenkins plugins

From **Manage Jenkins → Plugins → Available plugins**, install:

- **Docker Pipeline**
- **SSH Agent**
- **GitHub Integration** (or GitHub plugin, usually pre-installed)
- **Pipeline: Stage View** (optional, for nicer visualization)

### 11.3 Give Jenkins access to Docker

If Jenkins runs natively (not in a container):

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

### 11.4 Add Jenkins Credentials

Go to **Manage Jenkins → Credentials → System → Global credentials → Add
Credentials**, and create:

1. **Docker Hub credentials**
   - Kind: *Username with password*
   - Username: your Docker Hub username
   - Password: your Docker Hub password or access token
   - ID: `dockerhub-creds` (must match the Jenkinsfile)

2. **EC2 SSH key**
   - Kind: *SSH Username with private key*
   - Username: `ubuntu`
   - Private key: paste the contents of `todo-app-key.pem`
   - ID: `ec2-ssh-key` (must match the Jenkinsfile)

### 11.5 Create the Pipeline Job

1. **New Item → Pipeline**, name it `todo-devops-pipeline`.
2. Under **Pipeline**, choose **Pipeline script from SCM**.
3. SCM: **Git**.
4. Repository URL: `https://github.com/YOUR-USERNAME/todo-devops-project.git`
5. Branch: `*/main`.
6. Script Path: `Jenkinsfile`.
7. Save.

### 11.6 Update the placeholders in the Jenkinsfile

Before your first real run, edit these lines in `Jenkinsfile`:

```groovy
DOCKERHUB_USERNAME = 'your-dockerhub-username'
EC2_HOST            = 'ubuntu@YOUR_EC2_PUBLIC_IP'
```

Commit and push this change.

---

## 12. Configure the GitHub Webhook (auto-trigger Jenkins)

1. In your GitHub repo, go to **Settings → Webhooks → Add webhook**.
2. **Payload URL:** `http://YOUR_JENKINS_SERVER_IP:8080/github-webhook/`
3. **Content type:** `application/json`
4. **Trigger:** "Just the push event"
5. Save.

In Jenkins, open your pipeline job → **Configure → Build Triggers**, and
check **GitHub hook trigger for GITScm polling**.

> Jenkins must be reachable from the internet (public IP, or a tunnel like
> ngrok) for GitHub to deliver the webhook.

---

## 13. How the Jenkins Pipeline Works

The `Jenkinsfile` runs these stages in order:

1. **Checkout** — clones the latest code from GitHub.
2. **Validate** — confirms `index.html` and `style.css` exist and does a
   basic sanity check on the HTML.
3. **Docker Build** — builds the image from `Dockerfile` and tags it with
   both the Jenkins build number and `latest`.
4. **Docker Hub Push** — logs in with the `dockerhub-creds` credential and
   pushes both tags.
5. **AWS EC2 Deployment** — SSHes into EC2 (using `ec2-ssh-key`), pulls the
   new image, stops/removes the old container, and starts the new one.
6. **Verify** — confirms the container is running and port 80 is bound.

---

## 14. Test the Full Pipeline End to End

1. Edit `index.html` locally (e.g. change a task's text).
2. Commit and push:
   ```bash
   git add .
   git commit -m "Update a task"
   git push
   ```
3. GitHub sends the webhook → Jenkins starts a build automatically.
4. Watch the build in **Jenkins → todo-devops-pipeline → Build History**.
5. Once it finishes successfully, refresh `http://YOUR_EC2_PUBLIC_IP` — your
   change should be live.

---

## 15. Kubernetes Deployment (Optional)

The `k8s/` folder contains a basic Deployment and Service so you can also
demonstrate container orchestration.

### 15.1 Update the image name

Edit `k8s/deployment.yaml` and set the image to your own Docker Hub image:

```yaml
image: YOUR-DOCKERHUB-USERNAME/todo-devops-app:latest
```

### 15.2 Run with Minikube

```bash
minikube start
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml

kubectl get pods
kubectl get svc

minikube service todo-app-service --url
```

Open the URL it prints to view the site.

### 15.3 Run with Docker Desktop Kubernetes

1. Enable Kubernetes in Docker Desktop settings.
2. Apply the manifests:
   ```bash
   kubectl apply -f k8s/deployment.yaml
   kubectl apply -f k8s/service.yaml
   ```
3. Visit `http://localhost:30080` (the `nodePort` set in `service.yaml`).

---

## 16. Troubleshooting

| Problem | Likely Cause | Fix |
|---|---|---|
| `docker: permission denied` | User not in `docker` group | `sudo usermod -aG docker $USER`, then log out/in |
| Jenkins build can't run `docker` | Jenkins user lacks Docker access | `sudo usermod -aG docker jenkins`, restart Jenkins |
| Website not reachable on EC2 | Security group blocks port 80 | Add an inbound rule for HTTP (port 80) |
| Webhook not triggering builds | Jenkins not publicly reachable, or wrong payload URL | Confirm the URL ends in `/github-webhook/` and Jenkins is internet-accessible |
| `docker push` fails with auth error | Wrong/expired Docker Hub credentials | Recreate the `dockerhub-creds` credential in Jenkins, consider using an access token instead of a password |
| SSH stage fails ("Host key verification failed") | New EC2 host key not trusted | Pipeline already uses `-o StrictHostKeyChecking=no`; confirm the EC2 IP/hostname is correct |
| Old container still running after deploy | Container name conflict | The pipeline runs `docker stop` / `docker rm` before starting the new one — check logs for errors in that step |
| Kubernetes pods stuck in `ImagePullBackOff` | Image name/tag wrong, or repo is private | Verify the image name in `deployment.yaml` and that the Docker Hub repo is public (or add an image pull secret) |

---

## 17. College Project Demonstration Checklist

- [ ] GitHub repository created with the project pushed
- [ ] README explains the project (this file)
- [ ] `index.html` / `style.css` render correctly in a browser
- [ ] Docker image builds successfully (`docker build`)
- [ ] Container runs locally and serves the site on port 80
- [ ] Docker Hub repository created and image pushed
- [ ] Jenkins installed, plugins configured, credentials added
- [ ] Jenkins pipeline job created and pointed at the `Jenkinsfile`
- [ ] GitHub webhook configured and confirmed to trigger builds
- [ ] AWS EC2 instance running Docker, reachable on port 80
- [ ] End-to-end test: push to GitHub → Jenkins build → live site updates
- [ ] (Optional) Kubernetes Deployment + Service running via Minikube or
      Docker Desktop
- [ ] Screenshots captured of: GitHub repo, Jenkins pipeline stages (green),
      Docker Hub repository, live website on EC2, `kubectl get pods` output

---

## 18. Notes

- This project intentionally has **no JavaScript**. Checkboxes and delete
  buttons are static UI elements for demonstration only — the focus is the
  DevOps pipeline, not application logic.
- The only script referenced by `index.html` is Bootstrap's own bundled
  JavaScript, which is required for the navbar's mobile toggle button to
  open and close. It contains no custom code. If you want a 100%
  script-free page, you can remove that `<script>` tag — the navbar links
  will still work, only the mobile collapse animation will be lost.
