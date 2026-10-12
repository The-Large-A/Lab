# Lab 8 — End-to-End CI/CD Pipeline for a Java Web Application

**Tools:** Git · Maven · Docker · Jenkins

**Goal:** Build a small Java web application, containerise it, and automate the whole
build-test-package-deploy chain with a Jenkins pipeline.

---

## 1. Prerequisites

| Tool | Check with | Notes |
|---|---|---|
| Git | `git --version` | |
| JDK 21 | `java -version` | `winget install EclipseAdoptium.Temurin.21.JDK` |
| Maven | `mvn -version` | `choco install maven -y` (admin shell) |
| Docker Desktop | `docker ps` | Must be **running**, not just installed |
| Jenkins | — | Download the WAR, do not use the MSI (see §6) |

```bash
mkdir -p ~/jenkins
curl -L -o ~/jenkins/jenkins.war https://get.jenkins.io/war-stable/latest/jenkins.war
```

> **Critical:** after installing the JDK or Maven, close every terminal and open a new
> one. `PATH` and `JAVA_HOME` only apply to new shells. This is the single most common
> reason `mvn` "isn't installed" right after installing it.

---

## 2. Project structure

Create the project **outside** any cloud-synced folder (OneDrive sync fights with
Maven's `target/` directory and with `.git`).

```
C:\Users\<you>\student-app\
├── pom.xml
├── Dockerfile
├── Jenkinsfile
├── .gitignore
├── README.md
└── src/
    ├── main/java/com/lab/StudentApp.java
    ├── main/java/com/lab/StudentStore.java
    └── test/java/com/lab/StudentStoreTest.java
```

```bash
mkdir -p ~/student-app/src/main/java/com/lab
mkdir -p ~/student-app/src/test/java/com/lab
cd ~/student-app
```

---

## 3. Write the source files

### `pom.xml`

The application uses the JDK's built-in HTTP server (`com.sun.net.httpserver`), so the
only dependency is JUnit at test scope. That keeps the build fast and the Docker image
small.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">

  <modelVersion>4.0.0</modelVersion>

  <groupId>com.lab</groupId>
  <artifactId>student-app</artifactId>
  <version>1.0.0</version>
  <packaging>jar</packaging>

  <properties>
    <maven.compiler.release>21</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <version>5.11.3</version>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <finalName>student-app</finalName>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-jar-plugin</artifactId>
        <version>3.4.2</version>
        <configuration>
          <archive>
            <manifest>
              <mainClass>com.lab.StudentApp</mainClass>
            </manifest>
          </archive>
        </configuration>
      </plugin>
    </plugins>
  </build>

</project>
```

`<finalName>student-app</finalName>` produces `target/student-app.jar` instead of
`student-app-1.0.0.jar`, so the Dockerfile never needs updating when the version changes.

### `src/main/java/com/lab/StudentStore.java`

```java
package com.lab;

import java.util.ArrayList;
import java.util.List;

/** In-memory store of student records. */
public class StudentStore {

    private final List<String> students = new ArrayList<>();

    /** Adds a student. Blank names are rejected. */
    public void add(String name) {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("Student name is required");
        }
        students.add(name.trim());
    }

    /** Returns all students. */
    public List<String> list() {
        return List.copyOf(students);
    }

    /** Returns how many students are stored. */
    public int count() {
        return students.size();
    }
}
```

### `src/main/java/com/lab/StudentApp.java`

```java
package com.lab;

import com.sun.net.httpserver.HttpExchange;
import com.sun.net.httpserver.HttpServer;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.URLDecoder;
import java.nio.charset.StandardCharsets;

/** Minimal Student Management web application. */
public class StudentApp {

    static final StudentStore STORE = new StudentStore();

    public static void main(String[] args) throws IOException {
        int port = Integer.parseInt(System.getenv().getOrDefault("PORT", "8080"));
        HttpServer server = HttpServer.create(new InetSocketAddress(port), 0);
        server.createContext("/", StudentApp::home);
        server.createContext("/students", StudentApp::students);
        server.createContext("/add", StudentApp::add);
        server.start();
        System.out.println("Student Management System running on port " + port);
    }

    static void home(HttpExchange ex) throws IOException {
        send(ex, 200, """
                Student Management System
                -------------------------
                GET  /students         view all students
                GET  /add?name=Asha    add a student
                """);
    }

    static void students(HttpExchange ex) throws IOException {
        if ("POST".equals(ex.getRequestMethod())) {
            String name = new String(ex.getRequestBody().readAllBytes(), StandardCharsets.UTF_8);
            STORE.add(name);
            send(ex, 201, "Added: " + name.trim() + "\n");
            return;
        }
        StringBuilder out = new StringBuilder("Students (" + STORE.count() + "):\n");
        for (String s : STORE.list()) {
            out.append(" - ").append(s).append("\n");
        }
        send(ex, 200, out.toString());
    }

    static void add(HttpExchange ex) throws IOException {
        String query = ex.getRequestURI().getQuery();
        if (query == null || !query.startsWith("name=")) {
            send(ex, 400, "Usage: /add?name=Asha\n");
            return;
        }
        String name = URLDecoder.decode(query.substring(5), StandardCharsets.UTF_8);
        STORE.add(name);
        send(ex, 200, "Added: " + name + "\nTotal students: " + STORE.count() + "\n");
    }

    static void send(HttpExchange ex, int status, String body) throws IOException {
        byte[] bytes = body.getBytes(StandardCharsets.UTF_8);
        ex.getResponseHeaders().add("Content-Type", "text/plain; charset=utf-8");
        ex.sendResponseHeaders(status, bytes.length);
        ex.getResponseBody().write(bytes);
        ex.close();
    }
}
```

`HttpServer` uses longest-prefix matching, so `/students` and `/add` take precedence
over the catch-all `/` context.

### `src/test/java/com/lab/StudentStoreTest.java`

```java
package com.lab;

import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTrue;

class StudentStoreTest {

    @Test
    void addsAndListsStudents() {
        StudentStore store = new StudentStore();
        store.add("Asha");
        store.add("Ravi");

        assertEquals(2, store.count());
        assertTrue(store.list().contains("Asha"));
        assertTrue(store.list().contains("Ravi"));
    }

    @Test
    void rejectsBlankName() {
        StudentStore store = new StudentStore();
        assertThrows(IllegalArgumentException.class, () -> store.add("   "));
    }
}
```

### `Dockerfile`

```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY target/student-app.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

`jre` (not `jdk`) because Maven already compiled the code — the container only needs to
run it. `alpine` keeps the image around 180 MB instead of 400 MB.

### `Jenkinsfile`

```groovy
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/<your-username>/student-app.git'
            }
        }

        stage('Build & Test') {
            steps {
                bat 'mvn clean package'
            }
        }

        stage('Docker Build') {
            steps {
                bat 'docker build -t student-app:latest .'
            }
        }

        stage('Deploy') {
            steps {
                bat(script: 'docker rm -f student-app', returnStatus: true)
                bat 'docker run -d --name student-app -p 8081:8080 student-app:latest'
            }
        }
    }
}
```

Use `bat` not `sh` — the agent is Windows. `returnStatus: true` makes the cleanup step
tolerate "no such container" on the first run.

### `.gitignore`

```
target/
.idea/
*.iml
```

---

## 4. Git repository and commits

The brief asks for **meaningful commits during development**, so stage progressively
rather than committing everything at once.

```bash
cd ~/student-app
git init
git config user.name "Your Name"
git config user.email "your.email@example.com"

git add pom.xml .gitignore
git commit -m "Add Maven project structure and pom.xml"

git add src/main/java/com/lab/StudentStore.java
git commit -m "Add student store with add and view operations"

git add src/main/java/com/lab/StudentApp.java
git commit -m "Add HTTP web layer for student management"

git add src/test/java/com/lab/StudentStoreTest.java
git commit -m "Add unit tests for student store"

git add Dockerfile
git commit -m "Add Dockerfile for containerisation"

git add Jenkinsfile
git commit -m "Add Jenkins CI/CD pipeline"

git add README.md
git commit -m "Add project README"

git branch -M main
git log --oneline --graph
```

### Push to GitHub

On GitHub: **New repository** → name it `student-app` → **Public** → do **not** tick
"Add a README" (that would create a divergent commit and your first push would be
rejected).

```bash
git remote add origin https://<your-username>@github.com/<your-username>/student-app.git
git push -u origin main
```

---

## 5. Maven build and test

```bash
mvn clean package
```

Confirm the output contains both:

```
Tests run: 2, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

```bash
ls -lh target/student-app.jar
```

---

## 6. Docker

```bash
docker build -t student-app:latest .
docker images student-app
docker run -d --name student-app -p 8081:8080 student-app:latest
docker ps
```

`docker ps` should show `0.0.0.0:8081->8080/tcp`.

Verify in the browser:

- `http://localhost:8081/` — landing page
- `http://localhost:8081/add?name=Asha` — add operation
- `http://localhost:8081/add?name=Ravi`
- `http://localhost:8081/students` — view operation

Then remove the container so Jenkins can create its own on port 8081:

```bash
docker rm -f student-app
```

---

## 7. Jenkins

### Start it

**Do not install Jenkins from the MSI.** The MSI registers Jenkins as a Windows service
running under a system account that cannot see your `PATH` or reach Docker Desktop.
Running the WAR from your own terminal means Jenkins inherits your environment and can
call `mvn` and `docker` with zero configuration.

```bash
java -jar ~/jenkins/jenkins.war --httpPort=8080
```

Leave that window running. In a second terminal:

```bash
cat ~/.jenkins/secrets/initialAdminPassword
```

Open `http://localhost:8080`:

1. Paste the password → **Continue**
2. **Install suggested plugins** (3–5 min; includes the Git and Pipeline plugins)
3. Create your admin user
4. Accept the default Jenkins URL → **Start using Jenkins**

### Create the pipeline job

**New Item** → name `student-app-pipeline` → **Pipeline** → OK.

Scroll to the **Pipeline** section:

| Field | Value |
|---|---|
| Definition | **Pipeline script from SCM** |
| SCM | **Git** |
| Repository URL | `https://github.com/<your-username>/student-app.git` |
| Branch Specifier | `*/main` |
| Script Path | `Jenkinsfile` |

**Save**, then **Build Now**.

### Verify

The **Stage View** should show all four stages green: Checkout, Build & Test,
Docker Build, Deploy. **Console Output** should end with `Finished: SUCCESS`.

```bash
docker ps
```

Then open `http://localhost:8081/students`.

> The list will read `Students (0)` — this is correct. Jenkins built and deployed a
> brand-new container, and the store is held in memory, so it starts empty.

---

## 8. Port map

| Service | Port |
|---|---|
| Jenkins | 8080 |
| Application (host) | 8081 |
| Application (inside container) | 8080 |

---

## 9. Troubleshooting

| Problem | Cause and fix |
|---|---|
| `mvn: command not found` | Open a **new** terminal after installing |
| `src refspec main does not match any` | Local branch is still `master` — run `git branch -M main` |
| Jenkins can't find `mvn` or `docker` | Jenkins installed as a service. Use `java -jar jenkins.war` instead |
| `sh` step fails | Use `bat` on Windows |
| Push rejected on first push | You ticked "Add a README" when creating the repo |
| Port 8081 already allocated | `docker rm -f student-app` before re-running |
| `docker ps` says daemon not running | Start Docker Desktop and wait for the whale icon to settle |
