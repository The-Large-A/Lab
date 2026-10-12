# Lab 9 — Maven Dependency Management

**Tools:** Maven · JUnit · Apache Commons Lang

**Goal:** Build a Maven project, add a third-party dependency and a unit test, inspect
the dependency tree, configure an exclusion, and explain the difference between direct
dependencies, transitive dependencies, scopes and exclusions.

---

## 1. Prerequisites

| Tool | Check with |
|---|---|
| JDK 21 | `java -version` |
| Maven 3.9 | `mvn -version` |

Maven must report the correct JDK:

```
Apache Maven 3.9.16
Java version: 21.0.12.1, vendor: Eclipse Adoptium
```

No Docker, Jenkins or Git is needed for this lab.

---

## 2. Project structure

```
C:\Users\<you>\student-management\
├── pom.xml
└── src/
    ├── main/java/com/college/StudentService.java
    └── test/java/com/college/StudentServiceTest.java
```

```bash
mkdir -p ~/student-management/src/main/java/com/college
mkdir -p ~/student-management/src/test/java/com/college
cd ~/student-management
```

---

## 3. Create `pom.xml`

Required coordinates: groupId `com.college`, artifactId `student-management`,
version `1.0-SNAPSHOT`.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">

  <modelVersion>4.0.0</modelVersion>

  <groupId>com.college</groupId>
  <artifactId>student-management</artifactId>
  <version>1.0-SNAPSHOT</version>
  <packaging>jar</packaging>

  <properties>
    <maven.compiler.release>21</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <dependencies>

    <dependency>
      <groupId>org.apache.commons</groupId>
      <artifactId>commons-lang3</artifactId>
      <version>3.17.0</version>
    </dependency>

    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.13.2</version>
      <scope>test</scope>
    </dependency>

  </dependencies>

</project>
```

**Why JUnit 4 and not 5:** JUnit 4 has exactly one transitive dependency,
`hamcrest-core`, which makes it the clearest possible example for the exclusion part of
this lab. JUnit 5 pulls in seven transitive artifacts and the tree is harder to read.

---

## 4. Write the application class

`src/main/java/com/college/StudentService.java` — this must use at least one Apache
Commons Lang utility method. It uses two: `StringUtils.isBlank` and
`StringUtils.capitalize`.

```java
package com.college;

import org.apache.commons.lang3.StringUtils;

/** Student operations built on Apache Commons Lang utility methods. */
public class StudentService {

    /**
     * Normalises a student name using Commons Lang.
     * Blank input returns "Unknown".
     */
    public String formatName(String name) {
        if (StringUtils.isBlank(name)) {
            return "Unknown";
        }
        return StringUtils.capitalize(name.trim().toLowerCase());
    }

    public static void main(String[] args) {
        StudentService service = new StudentService();
        System.out.println(service.formatName("  aSHA  "));
        System.out.println(service.formatName("   "));
    }
}
```

---

## 5. Write the unit test

`src/test/java/com/college/StudentServiceTest.java` — JUnit 4 syntax: the class and
methods must be `public`, and the import is `org.junit.Test`.

```java
package com.college;

import org.junit.Test;

import static org.junit.Assert.assertEquals;

public class StudentServiceTest {

    private final StudentService service = new StudentService();

    @Test
    public void capitalisesAndTrimsName() {
        assertEquals("Asha", service.formatName("  aSHA  "));
    }

    @Test
    public void returnsUnknownForBlankName() {
        assertEquals("Unknown", service.formatName("   "));
    }
}
```

The tests deliberately use only `assertEquals`, never `assertThat` — this matters in §9.

---

## 6. Compile

```bash
mvn compile
```

Expect `BUILD SUCCESS`. Maven downloads `commons-lang3` on the first run and caches it
in `~/.m2/repository`.

---

## 7. Run the tests

```bash
mvn test
```

```
Tests run: 2, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

`mvn test` runs `validate → compile → test`, because running any lifecycle phase runs
every phase before it.

---

## 8. Inspect the dependency tree

```bash
mvn dependency:tree
```

```
com.college:student-management:jar:1.0-SNAPSHOT
+- org.apache.commons:commons-lang3:jar:3.17.0:compile
\- junit:junit:jar:4.13.2:test
   \- org.hamcrest:hamcrest-core:jar:1.3:test
```

**Read the indentation — that is the whole point of this step.**

| Artifact | Position | Type |
|---|---|---|
| `commons-lang3` | Root level | **Direct** — declared in `pom.xml` |
| `junit` | Root level | **Direct** — declared in `pom.xml` |
| `hamcrest-core` | Indented under `junit` | **Transitive** — pulled in by JUnit |

Note the scopes shown at the end of each line: `commons-lang3` is `compile` because the
application needs it at runtime; `junit` and `hamcrest-core` are `test` because only the
test code uses them.

---

## 9. Configure the exclusion

Replace the entire `junit` dependency block in `pom.xml` with:

```xml
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.13.2</version>
      <scope>test</scope>
      <exclusions>
        <exclusion>
          <groupId>org.hamcrest</groupId>
          <artifactId>hamcrest-core</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
```

An `<exclusion>` takes only `groupId` and `artifactId` — never a `version`.

### Verify the tree changed

```bash
mvn dependency:tree
```

```
com.college:student-management:jar:1.0-SNAPSHOT
+- org.apache.commons:commons-lang3:jar:3.17.0:compile
\- junit:junit:jar:4.13.2:test
```

`hamcrest-core` is gone.

### Prove the exclusion actually took effect

```bash
mvn test
```

This now **fails**:

```
java.lang.NoClassDefFoundError: org/hamcrest/SelfDescribing
    at org.junit.internal.builders.JUnit4Builder.runnerForClass(JUnit4Builder.java:10)
...
BUILD FAILURE
```

**This is expected and is the most instructive part of the lab.** JUnit 4 uses Hamcrest
internally to construct its test runner, even though the tests only call `assertEquals`
and never touch `assertThat`. The failure proves the JAR was genuinely removed from the
test classpath, not merely hidden from the dependency tree.

---

## 10. Restore a working build (recommended)

Leaving a failing build as your submission is risky if the work is run by someone else.
The correct real-world remedy is to exclude the transitive copy and then declare the
library directly yourself.

Add this **after** the `junit` block, still inside `<dependencies>`:

```xml
    <dependency>
      <groupId>org.hamcrest</groupId>
      <artifactId>hamcrest-core</artifactId>
      <version>1.3</version>
      <scope>test</scope>
    </dependency>
```

```bash
mvn dependency:tree
```

```
com.college:student-management:jar:1.0-SNAPSHOT
+- org.apache.commons:commons-lang3:jar:3.17.0:compile
+- junit:junit:jar:4.13.2:test
\- org.hamcrest:hamcrest-core:jar:1.3:test
```

`hamcrest-core` has moved from **indented** (transitive) to **root level** (direct) —
you now control its version explicitly.

```bash
mvn test
```

```
Tests run: 2, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

---

## 11. The four definitions

These are the written deliverable for this lab.

**Direct dependency** — a library declared explicitly in the `<dependencies>` section of
`pom.xml`, whose version and scope the project controls. In this project, `commons-lang3`
and `junit` are direct dependencies.

**Transitive dependency** — a library Maven adds automatically because a direct
dependency requires it. It is never declared in the project. Here, `hamcrest-core` is
transitive because `junit` depends on it.

**Dependency scope** — controls when a dependency is available on the classpath and
whether it is packaged with the application. `compile`, the default, applies to
compilation, testing and runtime and is packaged into the artifact; `test` applies only
to compiling and running tests and is never packaged. `commons-lang3` uses `compile`
because the application needs it at runtime, while `junit` uses `test` because only the
test code requires it.

**Dependency exclusion** — an explicit instruction that prevents a transitive dependency
from being included, written as an `<exclusions>` block inside the dependency that brings
it in. It is used to resolve version conflicts, remove duplicate or unwanted libraries,
and reduce the size of the final artifact.

### Scope reference

| Scope | Compile | Test | Runtime | Packaged | Typical use |
|---|---|---|---|---|---|
| `compile` | ✅ | ✅ | ✅ | ✅ | Normal libraries (default) |
| `test` | ❌ | ✅ | ❌ | ❌ | JUnit, Mockito |
| `provided` | ✅ | ✅ | ❌ | ❌ | Servlet API |
| `runtime` | ❌ | ✅ | ✅ | ✅ | JDBC drivers |
| `system` | ✅ | ✅ | ❌ | ❌ | Deprecated, avoid |

---

## 12. Command reference

```bash
mvn compile                 # compile main sources
mvn test                    # compile + run tests
mvn package                 # compile + test + build the JAR
mvn clean package           # delete target/ first
mvn dependency:tree         # show direct and transitive dependencies
mvn dependency:tree -Dverbose   # also show omitted/conflicting versions
mvn package -DskipTests     # skip running tests
```

---

## 13. Troubleshooting

| Problem | Cause and fix |
|---|---|
| `mvn: command not found` | Open a **new** terminal after installing Maven |
| `release version 21 not supported` | JDK older than 21 — check `mvn -version` |
| `Tests run: 0` | Class name must match `*Test.java`; class and methods must be `public` in JUnit 4 |
| `package org.junit does not exist` | JUnit dependency missing or misspelled |
| `NoClassDefFoundError: org/hamcrest/SelfDescribing` | Expected after the exclusion — see §9 |
| Dependency not downloading | Check internet access; delete the broken folder under `~/.m2/repository` and retry |
