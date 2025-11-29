# React-CI-CD-Bootcamp-Lab-Devops-project
Hands-on lab: launch Ubuntu EC2, create sudo user, install Node, clone React chat app, build &amp; deploy with pm2 &amp; Nginx, secure with UFW, automate via Jenkins CI/CD to S3, Dockerize with compose—full DevOps pipeline from code to cloud.
https://rlarnfwbhi57o.ok.kimi.link/ website of this project
<img width="1536" height="1024" alt="ChatGPT Image Nov 29, 2025, 08_39_58 PM" src="https://github.com/user-attachments/assets/7143422d-2e69-4352-a063-ef3da3d0f4f0" />

<img width="1002" height="814" alt="image" src="https://github.com/user-attachments/assets/0868b982-991e-4f30-a88a-ddce02ea1270" />
<img width="1002" height="528" alt="image" src="https://github.com/user-attachments/assets/f8fa2917-5024-450c-a67b-cbea11e157f1" />

+----------------+         +----------------+
|   GitHub Repo  | <-----> |  Jenkins EC2   |
+----------------+         +----------------+
                                     |
                                     v
+----------------+         +----------------+
|  pm2 :80       | <-----> |  Nginx         |
+----------------+         +----------------+
                                     |
                                     v
+----------------+         +----------------+
|  Docker :3000  |         |  S3 Bucket     |
+----------------+         +----------------+

 1. Create the Local Folder Structure
On your laptop (or the EC2 instance you used for the challenge) run:
bash
Copy
mkdir devops-react-cicd-lab && cd devops-react-cicd-lab
mkdir -p {scripts,configs,docs,assets}
touch README.md .gitignore
2. Populate the Key Files
2.1 .gitignore
bash
Copy
cat > .gitignore <<'EOF'
node_modules/
build/
dist/
.env
*.pem
*.key
*.log
.DS_Store
EOF
2.2 scripts/01-setup-ec2.sh
bash
Copy
cat > scripts/01-setup-ec2.sh <<'EOF'
#!/usr/bin/env bash
# Run on a fresh Ubuntu 22.04 EC2 instance
set -e
sudo apt update -y
sudo apt install -y openjdk-11-jdk nginx docker.io docker-compose git ufw

# Add ubuntu user to docker group (so you can run docker without sudo)
sudo usermod -aG docker ubuntu

# Allow ports
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 3000/tcp
sudo ufw --force enable

echo "Base packages installed. Re-login to apply docker group."
EOF
chmod +x scripts/01-setup-ec2.sh
2.3 scripts/02-create-sudo-user.sh
bash
Copy
cat > scripts/02-create-sudo-user.sh <<'EOF'
#!/usr/bin/env bash
# Creates the DevOps user with sudo rights
set -e
sudo adduser --disabled-password --gecos "" DevOps
echo 'DevOps:DevOpsPass123!' | sudo chpasswd
sudo usermod -aG sudo DevOps
echo "User DevOps created and added to sudo group."
EOF
chmod +x scripts/02-create-sudo-user.sh
2.4 scripts/03-install-node.sh
bash
Copy
cat > scripts/03-install-node.sh <<'EOF'
#!/usr/bin/env bash
# Installs Node 18 LTS via NodeSource
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs
node -v && npm -v
EOF
chmod +x scripts/03-install-node.sh
2.5 scripts/04-clone-build.sh
bash
Copy
cat > scripts/04-clone-build.sh <<'EOF'
#!/usr/bin/env bash
# Clones the React project and builds it
set -e
sudo mkdir -p /opt/checkout/chat-app
cd /opt/checkout/chat-app
sudo chown -R $USER:$USER .
git clone https://github.com/riyanuddin17/Real-Time-Chat-App.git .
npm ci
npm run build
sudo mkdir -p /opt/deployment/react
sudo rm -rf /opt/deployment/react/*
cp -r build/* /opt/deployment/react/
echo "Build copied to /opt/deployment/react"
EOF
chmod +x scripts/04-clone-build.sh
2.6 scripts/05-pm2-serve.sh
bash
Copy
cat > scripts/05-pm2-serve.sh <<'EOF'
#!/usr/bin/env bash
# Installs pm2 and serves the build folder on port 80
sudo npm i -g pm2
cd /opt/deployment/react
pm2 serve build 80 --spa --name react-chat
pm2 save
pm2 startup
EOF
chmod +x scripts/05-pm2-serve.sh
2.7 configs/nginx-default
bash
Copy
cat > configs/nginx-default <<'EOF'
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    root /opt/deployment/react;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
EOF
2.8 configs/jenkins-pipeline.groovy
bash
Copy
cat > configs/jenkins-pipeline.groovy <<'EOF'
pipeline {
    agent any
    environment {
        AWS_REGION = 'us-east-1'
        S3_BUCKET  = 'your-s3-bucket-name'
    }
    stages {
        stage('Stop old') {
            steps { sh 'pm2 stop react-chat || true' }
        }
        stage('Git Pull') {
            steps { dir('/opt/checkout/chat-app') { sh 'git pull' } }
        }
        stage('Build') {
            steps {
                dir('/opt/checkout/chat-app') {
                    sh 'npm ci'
                    sh 'npm run build'
                }
                sh 'rm -rf /opt/deployment/react/*'
                sh 'cp -r /opt/checkout/chat-app/build/* /opt/deployment/react/'
            }
        }
        stage('Deploy') {
            steps { sh 'pm2 restart react-chat || pm2 serve /opt/deployment/react 80 --spa --name react-chat' }
        }
        stage('Upload to S3') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'aws-creds',
                                  accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                  secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                    sh 'aws s3 sync /opt/deployment/react/ s3://$S3_BUCKET --delete --region $AWS_REGION'
                }
            }
        }
    }
}
EOF
2.9 Dockerfile
bash
Copy
cat > Dockerfile <<'EOF'
# ---- Build stage ----
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# ---- Serve stage ----
FROM nginx:1.25-alpine
COPY --from=builder /app/build /usr/share/nginx/html
COPY configs/nginx-default /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g","daemon off;"]
EOF
2.10 docker-compose.yml
bash
Copy
cat > docker-compose.yml <<'EOF'
version: "3.9"
services:
  chat:
    build: .
    ports:
      - "3000:80"
    restart: unless-stopped
EOF
3. Write a Comprehensive README
bash
Copy
cat > README.md <<'EOF'
# DevOps React CI/CD Lab

A complete, reproducible lab that walks you through:

1. Launching an AWS EC2 Ubuntu instance  
2. Creating a sudo user (`DevOps`)  
3. Installing Node, Nginx, Docker, Jenkins, pm2  
4. Cloning & building a React chat app  
5. Serving it with Nginx **and** pm2  
6. Securing the instance with UFW  
7. Setting up a Jenkins pipeline that:
   - Pulls fresh code  
   - Builds the React app  
   - Deploys with pm2  
   - Uploads the build to an S3 bucket  
8. Dockerizing the React app and running it on port 3000 via `docker-compose`

---

## 📁 Repository Layout
.
├── scripts/               # One-shot setup & deployment scripts
├── configs/               # Nginx, Jenkins, etc.
├── docs/                  # Extra screenshots & PDF guide (you add)
├── assets/                # Diagrams, images
├── Dockerfile
├── docker-compose.yml
└── README.md              # ← you are here
Copy
![IMG-20251125-WA0001](https://github.com/user-attachments/assets/486bd636-4163-44f1-bf63-1eb491425ec9)

![IMG-20251125-WA0002](https://github.com/user-attachments/assets/4d05943d-5513-4f66-a165-b8a8fc82869a)


![IMG-20251125-WA0003](https://github.com/user-attachments/assets/96a1f0f2-5d23-44a6-9c96-916e03a794c8)

![IMG-20251125-WA0004](https://github.com/user-attachments/assets/5726c11c-4430-4005-9431-23f2682cd764)

![IMG-20251125-WA0005](https://github.com/user-attachments/assets/5bfbe0c4-8476-4f76-8460-f19b3d48bc53)

![IMG-20251125-WA0006](https://github.com/user-attachments/assets/50561b4a-a32c-405d-8a2b-9a4770b02a2b)

![IMG-20251125-WA0007](https://github.com/user-attachments/assets/ec70220d-4815-4489-a304-af54b05a4f4f)

![IMG-20251125-WA0008](https://github.com/user-attachments/assets/22e7b87d-8328-403d-9d6c-bbd20819b84b)

![IMG-20251125-WA0009](https://github.com/user-attachments/assets/84acf475-2f53-4b2b-bc8a-f701919b9c0d)

![IMG-20251125-WA0010](https://github.com/user-attachments/assets/fd3ee464-2005-4da6-93eb-86774265abfa)

![IMG-20251125-WA0011](https://github.com/user-attachments/assets/36d2e7f4-9264-4b1e-907b-d95c1687e2bf)

![IMG-20251125-WA0012](https://github.com/user-attachments/assets/f079de18-de6e-444c-ba45-deed396682db)

![IMG-20251125-WA0018](https://github.com/user-attachments/assets/e3fb871d-897c-40bf-871a-4db0b7008bc8)

![IMG-20251125-WA0020](https://github.com/user-attachments/assets/cf4870ab-dcaa-464b-b3d8-5397b4b4a749)

![IMG-20251125-WA0019](https://github.com/user-attachments/assets/0986588f-c5d2-4132-a314-170051214b2a)

![IMG-20251125-WA0021](https://github.com/user-attachments/assets/592387e5-f7a9-4d27-b80a-d5b3b4225747)



---

## 🚀 Quick Start (Instructor / Student)

### 1. Launch EC2
- AMI: **Ubuntu 22.04 LTS**
- Type: `t2.micro` (free tier)
- Security Group: open 22, 80, 443, 3000, 8080 (Jenkins)

### 2. Connect & Run Setup
```bash
git clone https://github.com/<your-org>/devops-react-cicd-lab.git
cd devops-react-cicd-lab
./scripts/01-setup-ec2.sh
# Re-login to apply docker group
3. Create DevOps User
bash
Copy
./scripts/02-create-sudo-user.sh
su - DevOps
4. Install Node & Build
bash
Copy
./scripts/03-install-node.sh
./scripts/04-clone-build.sh
5. Serve with pm2
bash
Copy
./scripts/05-pm2-serve.sh
Visit http://<EC2-PUBLIC-IP> → you should see the chat app.
6. Nginx (optional reverse proxy)
bash
Copy
sudo cp configs/nginx-default /etc/nginx/sites-available/default
sudo systemctl reload nginx
7. Jenkins CI/CD
bash
Copy
# Install Jenkins (already done in 01-setup-ec2.sh)
sudo systemctl enable --now jenkins
# Unlock at http://<IP>:8080
# Install plugins: Pipeline, AWS Credentials, AWS Pipeline Steps
# Create pipeline job, paste configs/jenkins-pipeline.groovy



8. Docker
bash
Copy
docker-compose up --build -d
# App now on http://<IP>:3000
🔐 Security Checklist
[ ] UFW active (only 22, 80, 443, 3000, 8080 open)
[ ] EC2 security group tightened
[ ] Jenkins admin password changed
[ ] S3 bucket policy least-privilege
