# Lab 12 — Blue-Green Deployment

**Tools:** Git · Node.js · Docker · Docker Hub · Nginx · Jenkins (Declarative Pipeline)

**Goal:** Deploy a Node.js API using the blue-green strategy, so that a new version is
released with **zero downtime** and can be rolled back instantly.

---

## 1. The concept

With a naive deployment you stop the old version and start the new one, leaving a gap
where the service is down. Blue-green removes that gap by keeping **two complete
environments**:

- **Blue** — the version currently live, serving all traffic
- **Green** — idle, where the new version is deployed and health-checked

An **Nginx reverse proxy** sits in front and points at whichever environment is live.

```
                 ┌──────────────┐
  Browser ──────▶│ nginx-router │──────▶ node-blue:3000   (live)
   :8000         │   (port 80)  │   ✗    node-green:3000  (idle)
                 └──────────────┘
```

**The release process:**

1. Blue is live. Deploy the new version to Green.
2. **Health check** Green — is it actually responding?
3. Only if healthy, **rewrite the Nginx config** and reload it so traffic goes to Green.
4. Green is now live. Tear down Blue.
5. The next release reverses the roles — it alternates automatically.

**Why it is zero-downtime:** `nginx -s reload` is a *graceful* reload. The master process
validates the new config, starts new worker processes with it, and lets the old workers
finish their in-flight requests before exiting. No connection is dropped and the router
container never stops.

**Why rollback is instant:** the previous environment is only removed *after* the switch
succeeds, so reverting is just pointing the router back.

---

## 2. Prerequisites

| Requirement | Notes |
|---|---|
| Git, Node.js 20+, Docker Desktop (running) | |
| Jenkins | `java -jar ~/jenkins/jenkins.war --httpPort=8080` |
| **Docker Hub account** | Free account at hub.docker.com |

---

## 3. Project structure

```
C:\Users\<you>\blue-green-api\
├── package.json
├── server.js
├── Dockerfile
├── .dockerignore
└── Jenkinsfile
```

```bash
mkdir -p ~/blue-green-api
cd ~/blue-green-api
npm init -y
npm install express
```

Add the `start` script to `package.json`:

```json
{
  "name": "blue-green-api",
  "version": "1.0.0",
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
const PORT = 3000;

app.get('/status', (req, res) => {
    res.json({
        status: "success",
        message: "The Node.js API is running smoothly.",
        active_environment: process.env.NODE_ENV || "standalone",
        timestamp: new Date().toISOString()
    });
});

app.listen(PORT, () => {
    console.log(`Server is listening on port ${PORT}`);
});
```

**`active_environment` is the key field.** The pipeline injects `NODE_ENV=blue` or
`NODE_ENV=green` with `docker run -e`, so the JSON response tells you which environment
is currently serving you. That is how the traffic switch is proved.

### `Dockerfile`

```dockerfile
FROM node:20-alpine
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```

### `.dockerignore`

```
node_modules
.git
```

---

## 5. Write the `Jenkinsfile`

Replace `<dockerhub-username>` with your own Docker Hub username.

```groovy
pipeline {
    agent any

    environment {
        IMAGE_NAME = '<dockerhub-username>/blue-green-api'
        DOCKERHUB_CREDS = 'dockerhub-credentials'
        TAG = "${env.BUILD_ID}"
    }

    stages {

        stage('Initialize Network & Router') {
            steps {
                // Ensure a shared Docker network and the Nginx proxy exist
                bat '''
                    docker network inspect app-network >nul 2>&1 || docker network create app-network
                    docker ps --format "{{.Names}}" | findstr "nginx-router" >nul 2>&1 || docker run -d --name nginx-router -p 8000:80 --network app-network nginx:alpine
                '''
            }
        }

        stage('Build & Push to Docker Hub') {
            steps {
                bat "docker build -t %IMAGE_NAME%:%TAG% ."
                withCredentials([usernamePassword(credentialsId: env.DOCKERHUB_CREDS,
                                                  passwordVariable: 'DOCKER_PWD',
                                                  usernameVariable: 'DOCKER_USR')]) {
                    bat """
                        docker login -u %DOCKER_USR% -p %DOCKER_PWD%
                        docker push %IMAGE_NAME%:%TAG%
                    """
                }
            }
        }

        stage('Determine Target Environment') {
            steps {
                script {
                    // If node-blue is running, deploy to green next. Otherwise, deploy to blue.
                    def isBlueRunning = bat(script: 'docker ps --format "{{.Names}}" | findstr "node-blue"',
                                            returnStatus: true) == 0
                    env.ACTIVE_ENV = isBlueRunning ? 'blue' : 'green'
                    env.TARGET_ENV = isBlueRunning ? 'green' : 'blue'
                    echo "Current active environment: ${env.ACTIVE_ENV}"
                    echo "Deploying new version to: ${env.TARGET_ENV}"
                }
            }
        }

        stage('Deploy Target Environment') {
            steps {
                bat '''
                    docker rm -f node-%TARGET_ENV% >nul 2>&1 || true
                    docker run -d --name node-%TARGET_ENV% --network app-network -e NODE_ENV=%TARGET_ENV% %IMAGE_NAME%:%TAG%
                '''
            }
        }

        stage('Health Check') {
            steps {
                // Allow Node.js to boot, then verify /status over the internal network
                bat '''
                    timeout /t 5 /nobreak >nul
                    docker exec nginx-router wget -qO- http://node-%TARGET_ENV%:3000/status
                '''
            }
        }

        stage('Switch Traffic (Zero Downtime)') {
            steps {
                script {
                    // Generate a new Nginx configuration pointing at the validated container
                    def nginxConfig = """
                    events {}
                    http {
                        upstream backend {
                            server node-${env.TARGET_ENV}:3000;
                        }
                        server {
                            listen 80;
                            location / {
                                proxy_pass http://backend;
                            }
                        }
                    }
                    """
                    writeFile file: 'nginx.conf', text: nginxConfig
                }
                // Inject the new config into the router and gracefully reload
                bat '''
                    docker cp nginx.conf nginx-router:/etc/nginx/nginx.conf
                    docker exec nginx-router nginx -s reload
                '''
            }
        }

        stage('Teardown Old Environment') {
            steps {
                // Remove the old container to free up resources
                bat 'docker rm -f node-%ACTIVE_ENV% >nul 2>&1 || true'
            }
        }
    }
}
```

### How each stage works

| Stage | Purpose |
|---|---|
| **Initialize Network & Router** | Creates `app-network` and the `nginx-router` container if they do not already exist. The `\|\|` pattern makes this safe to re-run. |
| **Build & Push to Docker Hub** | Builds the image tagged with the Jenkins `BUILD_ID`, so every deployment traces to a specific build, then pushes it. |
| **Determine Target Environment** | Checks whether `node-blue` is running and picks the *other* colour as the target. This is what makes the pipeline alternate. |
| **Deploy Target Environment** | Starts the new container on the shared network with `NODE_ENV` set to its colour. **No port is published** — only the router is reachable from outside. |
| **Health Check** | Runs `wget` *from inside* `nginx-router` against the new container. If it fails, the pipeline stops and traffic never switches. |
| **Switch Traffic** | Writes a new `nginx.conf`, copies it into the router, and reloads gracefully. |
| **Teardown Old Environment** | Removes the previous container, now that nothing is routed to it. |

### Why containers reach each other by name

All containers join the user-defined network `app-network`. Docker provides automatic DNS
on user-defined networks, so `node-blue` and `node-green` resolve as hostnames. That is
why the Nginx config can simply say `server node-green:3000;`.

---

## 6. Git repository and push

```bash
cd ~/blue-green-api
git init
git config user.name "Your Name"
git config user.email "your.email@example.com"

git add .
git commit -m "Initial Commit"
git branch -M main
git remote add origin https://<your-username>@github.com/<your-username>/blue-green-deployment.git
git push -u origin main
```

---

## 7. Add Docker Hub credentials to Jenkins

The pipeline pushes to Docker Hub, which needs authentication. **Never put a password in
a Jenkinsfile.**

In Jenkins: **Manage Jenkins** → **Credentials** → **System** →
**Global credentials (unrestricted)** → **Add Credentials**.

| Field | Value |
|---|---|
| Kind | **Username with password** |
| Username | your Docker Hub username |
| Password | your Docker Hub password or **access token** |
| ID | `dockerhub-credentials` |
| Description | Docker Hub |

The **ID must exactly match** `DOCKERHUB_CREDS` in the Jenkinsfile.

> Prefer an access token over your password: Docker Hub → Account Settings → Security →
> New Access Token. It can be revoked without changing your password.

Jenkins masks these values in the console output — they appear as `****`.

---

## 8. Create the pipeline job

**New Item** → name `Blue-Green-Node-API` → **Pipeline** → **OK**.

| Field | Value |
|---|---|
| Definition | **Pipeline script from SCM** |
| SCM | **Git** |
| Repository URL | `https://github.com/<your-username>/blue-green-deployment.git` |
| Branch Specifier | `*/main` |
| Script Path | `Jenkinsfile` |

**Save.**

---

## 9. Build #1 — deploy to BLUE

Click **Build Now**.

Since no `node-blue` exists yet, "Determine Target Environment" picks **blue**.

Watch the Stage View — all seven stages should go green.

Verify:

```bash
docker ps
```

You should see `nginx-router` (with `0.0.0.0:8000->80/tcp`) and `node-blue`.

Open `http://localhost:8000/status`:

```json
{"status":"success","message":"The Node.js API is running smoothly.","active_environment":"blue","timestamp":"..."}
```

`active_environment` reads **blue**.

Check Docker Hub — your repository should now contain a tag matching the build number.

---

## 10. Build #2 — switch to GREEN

Make a visible change so you can confirm the new version is live.

Edit `server.js` and change the `message`:

```javascript
message: "The Node.js API is running smoothly. Changes are made now and committed to GitHub",
```

```bash
git add server.js
git commit -m "Intentionally changing server.js message to switch to green deployment"
git push origin main
```

Click **Build Now** again.

This time `node-blue` is running, so the pipeline targets **green**: it deploys
`node-green`, health-checks it, rewrites the Nginx config to point at it, reloads, and
removes `node-blue`.

Verify:

```bash
docker ps
```

Only `nginx-router` and `node-green` remain — `node-blue` is gone.

Open `http://localhost:8000/status` again:

```json
{"status":"success","message":"The Node.js API is running smoothly. Changes are made now and committed to GitHub","active_environment":"green","timestamp":"..."}
```

`active_environment` now reads **green** and the message is the updated one. The URL
never changed and the service was never down.

Run **Build Now** a third time and it will switch back to blue.

---

## 11. Port map

| Service | Port |
|---|---|
| Jenkins | 8080 |
| Nginx router (host) | 8000 |
| Nginx (inside router container) | 80 |
| node-blue / node-green | 3000 (internal only, not published) |

The application containers are deliberately **not** published to the host. The only way
in is through the router, which is what makes the traffic switch total.

---

## 12. Deployment strategies compared

| Strategy | How | Downtime | Rollback | Resource cost |
|---|---|---|---|---|
| **Recreate** | Stop old, start new | Yes | Slow — redeploy | 1× |
| **Rolling update** | Replace instances gradually | No | Gradual | ~1.2× |
| **Blue-Green** | Two environments, switch at once | No | Instant | 2× |
| **Canary** | Route a small % to the new version first | No | Instant | ~1.2× |

**Blue-green trade-offs:** you need double the infrastructure during a release, and
shared databases are the hard part — both environments usually talk to the same database,
so schema changes must be backward-compatible across both versions.

---

## 13. Useful Nginx commands

```bash
nginx -t            # validate the config WITHOUT applying it
nginx -s reload     # graceful reload (zero downtime)
nginx -s quit       # graceful stop
nginx -s stop       # immediate stop
```

Inside the router container:

```bash
docker exec nginx-router nginx -t
docker exec nginx-router cat /etc/nginx/nginx.conf
docker exec nginx-router nginx -s reload
```

Always `nginx -t` before a reload in production — an invalid config is rejected and the
old workers keep running, so a bad config cannot take the site down.

---

## 14. Manual cleanup

```bash
docker rm -f node-blue node-green nginx-router
docker network rm app-network
```

---

## 15. Troubleshooting

| Problem | Cause and fix |
|---|---|
| `denied: requested access to the resource is denied` | `IMAGE_NAME` must be `<your-dockerhub-username>/blue-green-api` |
| `Could not find credentials entry with ID` | The Jenkins credential ID does not match `DOCKERHUB_CREDS` |
| Health check fails | Container still booting — increase the `timeout /t 5` value |
| `wget: not found` | Use an Nginx image that bundles BusyBox (`nginx:alpine` does) |
| `host not found in upstream` | The container is not on `app-network`, so DNS cannot resolve the name |
| `nginx: invalid option` on reload | The generated `nginx.conf` is malformed — `docker exec nginx-router nginx -t` to see why |
| Both colours running after a build | Teardown stage failed; remove the stale one manually |
| `active_environment` always `standalone` | `-e NODE_ENV=...` missing from `docker run` |
| Port 8000 already allocated | Another process holds it; stop it or change the router's published port |
