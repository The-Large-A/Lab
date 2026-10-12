# Lab 10 — CI/CD for a Web Application Using Jenkins, Git and Docker

**Tools:** Git · GitHub Webhooks · Jenkins (Freestyle) · Docker · Nginx · ngrok

**Goal:** Host a static web page in an Nginx container and have Jenkins rebuild and
redeploy it **automatically** whenever a commit is pushed to GitHub.

This lab uses a **Freestyle** job (configured through the Jenkins UI) rather than a
Pipeline, and introduces GitHub webhooks as the build trigger.

---

## 1. Prerequisites

| Tool | Check with | Notes |
|---|---|---|
| Git | `git --version` | |
| Docker Desktop | `docker ps` | Must be **running** |
| Jenkins | — | `java -jar ~/jenkins/jenkins.war --httpPort=8080` |
| ngrok | `ngrok version` | Needed to expose local Jenkins to GitHub |

Install ngrok:

```bash
winget install ngrok.ngrok
```

Then sign up at ngrok.com, copy your authtoken and run:

```bash
ngrok config add-authtoken <your-token>
```

---

## 2. Project structure

```
C:\Users\<you>\webhooks\
├── index.html
└── Dockerfile
```

```bash
mkdir -p ~/webhooks
cd ~/webhooks
```

---

## 3. Write the application

### `index.html`

```html
<!DOCTYPE html>
<html>
<head><title>Jenkins Freestyle CI/CD</title></head>
<body>
    <h1>Hello from the automated Jenkins pipeline!</h1>
</body>
</html>
```

### `Dockerfile`

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/
EXPOSE 80
```

Three lines is all that is needed. `/usr/share/nginx/html/` is Nginx's default web root,
and the `nginx:alpine` base image already defines its own `CMD`, so none is required
here.

---

## 4. Git repository

```bash
cd ~/webhooks
git init
git config user.name "Your Name"
git config user.email "your.email@example.com"

git add .
git commit -m "Initial Commit"
git branch -M main
```

Create a **Public** repository on GitHub named `freestyle-project`, without a README,
then:

```bash
git remote add origin https://<your-username>@github.com/<your-username>/freestyle-project.git
git push -u origin main
git log --oneline
```

---

## 5. Start Jenkins

```bash
java -jar ~/jenkins/jenkins.war --httpPort=8080
```

Leave running. If this is a first install, unlock it:

```bash
cat ~/.jenkins/secrets/initialAdminPassword
```

Then at `http://localhost:8080`: paste the password → **Install suggested plugins** →
create your admin user. The suggested set includes the **Git** and
**GitHub Integration** plugins, both of which this lab needs.

> **Do not install Jenkins as a Windows service.** The service account cannot see your
> `PATH` or reach Docker Desktop. Running the WAR from your own terminal means Jenkins
> inherits your environment.

---

## 6. Create the Freestyle job

**New Item** → name `Freestyle-Web-App` → select **Freestyle project** → **OK**.

### General

Tick **This project is parameterised** → **Add Parameter** → **String Parameter**:

| Field | Value |
|---|---|
| Name | `GIT_REPO_URL` |
| Default Value | `https://github.com/<your-username>/freestyle-project` |

This keeps the repository URL configurable instead of hard-coded in several places.

### Source Code Management

Select **Git**:

| Field | Value |
|---|---|
| Repository URL | `$GIT_REPO_URL` |
| Credentials | `- none -` (public repo) |
| Branch Specifier | `*/main` |

### Build Triggers

Tick **GitHub hook trigger for GITScm polling**.

This is what makes the job respond to webhooks rather than only to manual builds.

### Build Steps

**Add build step** → **Execute Windows batch command**:

```bat
docker rm --force container1 || true
docker build -t nginx-image1 .
docker run -d -p 8081:80 --name=container1 nginx-image1
```

Line by line:

1. Remove any previous container. `|| true` prevents a failure on the first run when
   nothing exists yet.
2. Rebuild the image from the Dockerfile in the workspace.
3. Run it, mapping **host port 8081** to **container port 80**.

Port 8081 is used because Jenkins already occupies 8080.

**Save.**

---

## 7. Expose Jenkins with ngrok

GitHub's servers cannot reach `http://localhost:8080` — localhost means "this machine",
and your computer has no public address. ngrok opens an outbound tunnel from your machine
to its cloud servers and gives you a public HTTPS URL that forwards back down that tunnel.

In a **new terminal**:

```bash
ngrok http 8080
```

Output:

```
Forwarding   https://decent-enduring-passive.ngrok-free.app -> http://localhost:8080
```

Copy that HTTPS URL. **Leave ngrok running** — closing it kills the tunnel, and on the
free tier a restart gives you a different URL that you would have to re-enter in GitHub.

Visit `http://localhost:4040` to see ngrok's request inspector, which shows every
webhook delivery. This is the fastest way to debug a webhook that is not firing.

---

## 8. Configure the GitHub webhook

On GitHub: repository → **Settings** → **Webhooks** → **Add webhook**.

| Field | Value |
|---|---|
| Payload URL | `https://<your-ngrok-url>/github-webhook/` |
| Content type | `application/json` |
| Secret | leave blank |
| Which events | **Just the push event** |
| Active | ticked |

**Add webhook.**

The trailing slash on `/github-webhook/` matters — that is the exact endpoint the
Jenkins GitHub plugin listens on.

GitHub immediately sends a test ping. Refresh the webhook page and check **Recent
Deliveries** shows a green tick and `200 OK`.

---

## 9. Trigger the pipeline automatically

Make a change locally and push it:

```bash
cd ~/webhooks
```

Edit `index.html` to add a second line:

```html
<!DOCTYPE html>
<html>
<head><title>Jenkins Freestyle CI/CD</title></head>
<body>
    <h1>Hello from the automated Jenkins pipeline!</h1>
    <h2>Changes are made here, webhook</h2>
</body>
</html>
```

```bash
git add index.html
git commit -m "webhook test"
git push origin main
```

**Do not click Build Now.** Within a few seconds Jenkins should start a build on its
own. Watch the job page — a new build appears in the Build History.

Confirm the webhook reached Jenkins: on the job page, open **GitHub Hook Log**. It lists
each received POST with a `200 OK`.

---

## 10. Verify the deployment

```bash
docker ps
```

The container should show `0.0.0.0:8081->80/tcp`.

Open `http://localhost:8081` in a browser. The page must show the **new** content,
proving the full chain worked: push → webhook → Jenkins build → Docker rebuild →
container redeployed.

Check the build's **Console Output** ends with:

```
Finished: SUCCESS
```

---

## 11. How the pieces connect

```
You push to GitHub
        │
        ▼
GitHub sends POST to https://<ngrok>.ngrok-free.app/github-webhook/
        │
        ▼  (tunnel)
ngrok forwards to localhost:8080
        │
        ▼
Jenkins "GitHub hook trigger" fires → job starts
        │
        ▼
Git checkout → docker rm → docker build → docker run
        │
        ▼
New container live on http://localhost:8081
```

---

## 12. Port map

| Service | Port |
|---|---|
| Jenkins | 8080 |
| ngrok inspector | 4040 |
| Application (host) | 8081 |
| Nginx (inside container) | 80 |

---

## 13. Freestyle vs Pipeline

| | Freestyle (this lab) | Pipeline (Labs 8, 11, 12) |
|---|---|---|
| Configured by | Clicking in the Jenkins UI | Writing a `Jenkinsfile` |
| Stored in | Jenkins itself | Your Git repository |
| Version controlled | No | Yes |
| Complex logic | Difficult | Easy (conditions, loops, scripts) |
| Stage visualisation | Limited | Full stage view |
| Best for | Simple single-step jobs | Real CI/CD |

Freestyle is simpler to set up but the configuration lives only inside Jenkins, so it is
not reviewable, not branchable and lost if Jenkins is rebuilt.

---

## 14. Troubleshooting

| Problem | Cause and fix |
|---|---|
| Webhook shows a red X in GitHub | ngrok not running, or the URL changed after restart |
| Build does not trigger | "GitHub hook trigger for GITScm polling" not ticked |
| `404` on the webhook delivery | URL missing the trailing `/github-webhook/` |
| `docker: command not found` in the build | Jenkins running as a Windows service — use the WAR |
| `port is already allocated` | A previous `container1` still exists; the `docker rm --force` line handles this once it is in place |
| Page shows old content | Browser cache — hard-refresh with Ctrl+F5 |
| ngrok URL changed | Free tier assigns a new URL on every restart; update the webhook |
| ngrok warning page in browser | Free-tier interstitial; it does not affect GitHub's API calls |
