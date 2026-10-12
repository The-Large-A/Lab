# Lab 11 — CI/CD for a Node.js Application Using Jenkins, Git and Docker

**Tools:** Git · Node.js · Express · Docker · Jenkins (Declarative Pipeline)

**Goal:** Build a small Node.js REST API, containerise it, and automate the build and
deployment with a Jenkins declarative pipeline defined in a `Jenkinsfile`.

Where Lab 10 used a Freestyle job configured through the UI, this lab uses
**Pipeline as Code** — the build definition lives in the repository alongside the
application.

---

## 1. Prerequisites

| Tool | Check with | Notes |
|---|---|---|
| Git | `git --version` | |
| Node.js 20+ | `node -v` and `npm -v` | `winget install OpenJS.NodeJS.LTS` |
| Docker Desktop | `docker ps` | Must be **running** |
| Jenkins | — | `java -jar ~/jenkins/jenkins.war --httpPort=8080` |

Node is only needed locally to generate `package.json` and test before containerising.
The Docker build installs its own copy inside the image.

---

## 2. Project structure

```
C:\Users\<you>\node-status-api\
├── package.json
├── package-lock.json
├── server.js
├── Dockerfile
├── .dockerignore
└── Jenkinsfile
```

```bash
mkdir -p ~/node-status-api
cd ~/node-status-api
```

---

## 3. Initialise the Node project

```bash
npm init -y
npm install express
```

`npm init -y` creates `package.json`; `npm install express` adds Express and creates
`package-lock.json` and `node_modules/`.

Now open `package.json` and add a `start` script — the Dockerfile's `CMD ["npm","start"]`
depends on it:

```json
{
  "name": "node-status-api",
  "version": "1.0.0",
  "description": "Status API for the Jenkins CI/CD lab",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.21.2"
  }
}
```

---

## 4. Write the application

### `server.js`

```javascript
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

app.get('/status', (req, res) => {
    res.json({
        status: "success",
        message: "The Node.js API is running smoothly.",
        timestamp: new Date().toISOString()
    });
});

app.listen(PORT, () => {
    console.log(`Server is running on port ${PORT}`);
});
```

`process.env.PORT || 3000` lets the port be overridden by an environment variable at
runtime, falling back to 3000.

Test it locally before containerising:

```bash
node server.js
```

Open `http://localhost:3000/status`, confirm the JSON response, then stop it with Ctrl+C.

### `Dockerfile`

```dockerfile
# Use an official Node runtime as a parent image
FROM node:20-alpine
WORKDIR /usr/src/app

COPY package*.json ./
RUN npm install
COPY . .

EXPOSE 3000

CMD ["npm", "start"]
```

**Why `package*.json` is copied before the rest of the code:** Docker caches each
instruction as a layer and reuses unchanged ones. Dependencies change rarely, source code
changes every commit. Copying the manifests first means `npm install` stays cached until
`package.json` actually changes. Copying everything first would re-run `npm install` on
every single build.

### `.dockerignore`

```
node_modules
npm-debug.log
.git
Dockerfile
```

Without this, `COPY . .` copies your local `node_modules` into the image, overwriting the
one `npm install` just built inside the container — and sends hundreds of megabytes to
the Docker daemon on every build.

---

## 5. Write the `Jenkinsfile`

```groovy
pipeline {
    agent any

    environment {
        IMAGE_NAME = 'node-status-api'
        CONTAINER_NAME = 'node-status-container'
        PORT = '3000'
    }

    stages {
        stage('Build Docker Image') {
            steps {
                echo 'Building the Docker image...'
                bat "docker build -t ${IMAGE_NAME} ."
            }
        }

        stage('Deploy Docker Container') {
            steps {
                echo 'Deploying the application...'
                // Stop and remove the old container if it exists
                bat """
                    docker rm -f ${CONTAINER_NAME} || true
                    docker run -d -p ${PORT}:${PORT} --name ${CONTAINER_NAME} ${IMAGE_NAME}
                """
            }
        }

        stage('Verify Deployment') {
            steps {
                echo "Application successfully deployed and exposed on port ${PORT}."
            }
        }
    }
}
```

Notes on the syntax:

- **`environment { }`** declares variables once and reuses them across stages. Change the
  port in one place and every stage follows.
- **`bat`**, not `sh` — the agent is Windows. Using `sh` fails immediately.
- **Double quotes** (`"docker build -t ${IMAGE_NAME} ."`) let Groovy substitute
  `${...}` before the command runs. Single quotes would pass the text through literally.
- **Triple quotes** (`"""`) allow a multi-line command block.
- **`|| true`** makes the removal tolerate "no such container" on the first run.
- Each `stage` becomes a separate box in the Jenkins stage view.

---

## 6. Git repository and push

```bash
cd ~/node-status-api
git init
git config user.name "Your Name"
git config user.email "your.email@example.com"

git add .
git commit -m "Initial commit"
git branch -M main
```

Create a **Public** repository on GitHub named `node-status-api`, without a README, then:

```bash
git remote add origin https://<your-username>@github.com/<your-username>/node-status-api.git
git push -u origin main
git log --oneline
```

> If you forgot a `.gitignore`, `node_modules` will be committed — hundreds of files.
> It still works, but add a `.gitignore` containing `node_modules/` and run
> `git rm -r --cached node_modules` to clean it up.

---

## 7. Create the Jenkins pipeline job

Start Jenkins if it is not already running:

```bash
java -jar ~/jenkins/jenkins.war --httpPort=8080
```

At `http://localhost:8080`: **New Item** → name `Node-API-Pipeline` → select
**Pipeline** → **OK**.

Scroll to the **Pipeline** section:

| Field | Value |
|---|---|
| Definition | **Pipeline script from SCM** |
| SCM | **Git** |
| Repository URL | `https://github.com/<your-username>/node-status-api.git` |
| Credentials | `- none -` (public repo) |
| Branch Specifier | `*/main` |
| Script Path | `Jenkinsfile` |

**Save** → **Build Now**.

Using "Pipeline script from SCM" means Jenkins checks out the repository and reads the
`Jenkinsfile` from it. The pipeline definition is version-controlled with the code.

---

## 8. Verify

### In Jenkins

The **Stage View** should show all three stages green: Build Docker Image,
Deploy Docker Container, Verify Deployment.

**Console Output** should end with `Finished: SUCCESS` and show the `docker run` command
with the container ID it produced.

### On the command line

```bash
docker ps
```

Expect `0.0.0.0:3000->3000/tcp` and the container named `node-status-container`.

```bash
docker logs node-status-container
```

```
> node-status-api@1.0.0 start
> node server.js

Server is running on port 3000
```

### In the browser

Open `http://localhost:3000/status`:

```json
{"status":"success","message":"The Node.js API is running smoothly.","timestamp":"2026-09-20T16:25:04.059Z"}
```

Docker Desktop's **Containers** view shows the same container with its logs and port
mapping if you prefer a GUI.

---

## 9. Re-running the pipeline

Click **Build Now** again. The second build will be noticeably faster because Docker
reuses cached layers, and the `docker rm -f` line cleanly replaces the running container
rather than failing on a name clash.

To verify the pipeline really picks up code changes, edit the `message` string in
`server.js`, commit, push, and build again. The new text should appear at
`http://localhost:3000/status`.

---

## 10. Port map

| Service | Port |
|---|---|
| Jenkins | 8080 |
| Application (host) | 3000 |
| Application (inside container) | 3000 |

Unlike Labs 8 and 10, host and container ports are the same here because 3000 does not
clash with Jenkins.

---

## 11. Troubleshooting

| Problem | Cause and fix |
|---|---|
| `sh` step fails | Use `bat` on a Windows agent |
| `docker: command not found` in Jenkins | Jenkins installed as a Windows service — run the WAR from your own terminal |
| `npm start` fails: "Missing script: start" | No `start` entry in `package.json` scripts |
| `Cannot find module 'express'` | `node_modules` copied over the installed one — add `.dockerignore` |
| `port is already allocated` | Something else is on 3000; `docker rm -f node-status-container` |
| `name is already in use` | The `docker rm -f` line is missing or failed |
| Pipeline not found | `Script Path` must be exactly `Jenkinsfile` and the file must be pushed |
| Container exits immediately | `docker logs <name>` — usually a syntax error in `server.js` |
