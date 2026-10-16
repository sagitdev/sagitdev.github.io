---
title: "Getting the Dhorobha Backend Running Locally: Setup Notes and Pitfalls"
date: 2026-10-15
draft: false
tags: ["Java", "Maven", "Tomcat", "PostgreSQL", "setup", "debugging"]
summary: "How I set up the Dhorobha Java backend on my Mac: the stack, the steps, and six problems I ran into along the way."
---

My first milestone on this work project was getting the **Dhorobha** backend running on my own machine. Dhorobha is a platform for real estate listings (buying and selling) and land-use permit applications. It has three parts: a Java backend, a React website, and a mobile app. Since both the website and the app depend on the backend, the server had to come first.

This post records how I got it running, the problems I hit, and what I learned from each one.

> Note: the project depends on two libraries developed in-house, which I'll call **Framework B** and **Library D** here. Internal code and configuration details are left out.

## The stack

Here's what happens when I open `http://localhost:8080/` in the browser:

{{< mermaid >}}
flowchart LR
    A[Browser] -->|HTTP request| B[Tomcat<br/>port 8080]
    B --> C[Dhorobha Server<br/>ROOT.war]
    C --> D[Framework B<br/>+ Library D]
    D -->|reads| E[Config file]
    D -->|SQL| F[(PostgreSQL 17<br/>port 5432)]
    C -->|HTML / JS / CSS| A
{{< /mermaid >}}

1. **Tomcat** listens on port 8080 and hands the request to the Dhorobha Server app.
2. The server is built on **Framework B**, which depends on **Library D**. On startup, B reads a config file to find the database address and credentials.
3. B connects to **PostgreSQL** and loads the data.
4. The server returns HTML, JS, and CSS, and the browser renders the React frontend.

Every link in this chain has to be in place: Java, Maven, the two internal libraries, PostgreSQL with data, the config file, and Tomcat.

| Tool | Version | Role |
|---|---|---|
| JDK | 21 | Compile and run Java |
| Maven | 3.9.x | Build tool and dependency manager |
| PostgreSQL | 17 | Database |
| Tomcat | 10.1.x | Servlet container that runs the Java web app |

My environment: Intel Mac, Homebrew, bash.

## Setup steps

### 1. Get the code

I forked the repository, cloned my fork with GitHub Desktop (choosing "To contribute to the parent project" so the original repo is set up as `upstream`), and created a working branch.

### 2. Switch to Java 21

The project's `pom.xml` requires Java 21, but my default was 17. I switched only for the current terminal, so other projects keep using 17:

```bash
/usr/libexec/java_home -V                        # list installed JDKs
export JAVA_HOME=$(/usr/libexec/java_home -v 21)
mvn -v                                           # confirm Maven uses Java 21
```

### 3. Install the internal libraries into the local Maven repository

Framework B and Library D aren't published online, so Maven can't download them. I installed the jars provided by the team into `~/.m2/repository` manually, D first because B depends on it:

```bash
mvn install:install-file -Dfile=<library-d>.jar -DpomFile=<library-d>/pom.xml
mvn install:install-file -Dfile=<framework-b>.jar -DpomFile=<framework-b>/pom.xml
```

Passing `-DpomFile` lets Maven know each library's coordinates and its own dependencies.

### 4. Build

```bash
mvn clean package
```

Success looks like `BUILD SUCCESS` and a `.war` file in `target/`. This step failed for me at first; see Problem 3.

### 5. Prepare the database

```bash
brew services start postgresql@17
psql -d postgres -c "CREATE ROLE \"<DbUser>\" WITH LOGIN PASSWORD '<password>';"
psql -d postgres -c "CREATE DATABASE \"<DbName>\" OWNER \"<DbUser>\";"
pg_restore -U <DbUser> -d <DbName> --no-owner --no-privileges <backup-file>
```

- The names must match the config file exactly. PostgreSQL lowercases unquoted names, so names with capital letters need double quotes (escaped as `\"` inside a bash string).
- `--no-owner --no-privileges` skips the original owners and permissions recorded in the backup, which don't exist on my machine.

### 6. Place the config file

Framework B looks for its config file in a `Data` folder under Tomcat's working directory (`CATALINA_BASE`). I found the exact location by reading the framework's source rather than guessing. To find `CATALINA_BASE`:

```bash
/usr/local/opt/tomcat@10/bin/catalina version
```

Then I created the expected folder and copied the config file into it.

### 7. Deploy as ROOT and start Tomcat

```bash
BASE=/usr/local/Cellar/tomcat@10/10.1.47/libexec
mv "$BASE/webapps/ROOT" "$BASE/webapps/ROOT.tomcat-default"   # first time only
cp target/<app>.war "$BASE/webapps/ROOT.war"
JAVA_HOME=$(/usr/libexec/java_home -v 21) "$BASE/bin/catalina.sh" run
```

When the log shows the app started and `Server startup in [...] milliseconds`, the Dhorobha home page loads at `http://localhost:8080/`. 🎉

## Problems I ran into

### Problem 1: PostgreSQL 17 wouldn't start, and `psql` wasn't found

- **Symptom**: `brew services list` showed `postgresql@17 error`, and the terminal said `psql: command not found`.
- **Cause**: two separate issues. PostgreSQL 16 was also installed and already holding port 5432, so 17 couldn't bind to it. And Homebrew's versioned packages like `postgresql@17` are **keg-only**: their commands aren't added to `PATH`, to avoid conflicts between versions.
- **How I found it**: the log (`tail -n 20 /usr/local/var/log/postgresql@17.log`) said `Address already in use`.
- **Fix**:

  ```bash
  brew services stop postgresql@16
  brew services start postgresql@17
  echo 'export PATH="/usr/local/opt/postgresql@17/bin:$PATH"' >> ~/.bash_profile
  source ~/.bash_profile
  ```

### Problem 2: The build needed Java 21, but I had 17

- **Cause**: multiple JDKs installed; `java -version` shows only the current default.
- **Fix**: `export JAVA_HOME=$(/usr/libexec/java_home -v 21)` (Step 2).

### Problem 3: 100 identical compile errors ⭐

This was the most interesting problem, and the debugging process taught me the most.

- **Symptom**: `mvn clean package` failed with 100 errors, all in the same file, all saying:

  ```
  non-static method getEntityManager(java.lang.String) cannot be referenced from a static context
  ```

- **How I narrowed it down**:
  1. **Look for the pattern.** 100 identical errors in one file usually mean one root cause, not 100 problems.
  2. **Read the failing line.** The code called `getEntityManager` statically, through a class name.
  3. **Find where the method is defined.** It wasn't in the project. The class extended a base class from Framework B, and its header said it was generated by B and shouldn't be modified. So editing the code wasn't the answer.
  4. **Inspect the jar directly.** `javap` shows method signatures inside a jar without source code:

     ```bash
     javap -cp <framework-b>.jar <package>.<BaseClass> | grep getEntityManager
     ```

     In my local jar, the method was **not** `static`.
  5. **Check the timeline.** My local jar was installed the last time I set up this environment, almost a year earlier. The project code had since been regenerated with a newer version of Framework B.

- **Cause**: Framework B had been updated (the method became `static`), but the version number stayed the same. Maven saw that version already in my local repository and never fetched it again, so I was compiling against an outdated jar.
- **Fix**: back up `~/.m2/repository`, install the latest jars from the team, and confirm with `javap` that `static` now appears. The build passed.
- **Lesson**: **the same version number doesn't guarantee the same contents.** When a dependency behaves strangely, check what's actually inside the jar.

### Problem 4: The database import "succeeded" but had no tables

- **Symptom**: `psql -f backup.sql` ran without errors, but `\dt` said `Did not find any relations`.
- **Cause**: despite the `.sql` extension, the file was a `pg_dump` **custom-format** (binary) backup, which `psql` can't read.
- **How I found it**: `head -c 5 backup.sql` printed `PGDMP`, the signature of the custom format.
- **Fix**: import with `pg_restore` instead.
- **Lesson**: don't trust file extensions; look at the file's first bytes.

### Problem 5: A blank page with the title "null - null"

- **Symptom**: deployed as `DhorobhaServer.war`, the app at `http://localhost:8080/DhorobhaServer/` showed a blank page.
- **Cause**: the React build references its assets with absolute paths like `/static/js/main.xxx.js`. A leading `/` means "start from the site root," so the browser requested `http://localhost:8080/static/...`, which returned 404 because the app lived under `/DhorobhaServer/`.
- **How I found it**: viewed the page source (`Cmd + Option + U`), then opened the JS URL directly and got a 404.
- **Fix**: rename the war to **`ROOT.war`**. In Tomcat, the war filename becomes the URL path, and `ROOT` means the site root.

### Problem 6: Tomcat ran on Java 25 even though I set `JAVA_HOME`

- **Symptom**: the startup log said `JVM Version: 25`.
- **Cause**: Homebrew's `catalina` wrapper script forces Homebrew's own OpenJDK and ignores `JAVA_HOME`.
- **Fix**: call Tomcat's own script, `libexec/bin/catalina.sh`, with `JAVA_HOME` set on the same line (Step 7).

### Bonus: hundreds of changed files after every build

After `mvn clean package`, GitHub Desktop listed hundreds of changes under `target/`, Maven's build output folder. Some build output is tracked in the repository, so every build modifies it. I never commit anything under `target/`, and I discard those changes before pulling. For newly generated files, a local-only ignore rule helps without touching the shared `.gitignore`:

```bash
echo "target/" >> .git/info/exclude
```

`.git/info/exclude` works like `.gitignore` but only on my machine. Note that neither one affects files Git is already tracking.

## Takeaways

Looking back, the same approach solved every problem:

1. **Read the error message carefully.** `Address already in use` and `command not found` already say what's wrong.
2. **Look for patterns.** Many identical errors usually share one root cause.
3. **Form a hypothesis, then verify it with evidence**: `javap` for method signatures, `ls -la` for file dates, `head` for file contents, logs for runtime behavior.
4. **Change one thing at a time**: build first, then the database, then deployment.
5. **The code is the best documentation.** When no one could tell me where the config file belonged, the framework's source could.
6. **Back up before changing anything.** I backed up the Maven repository before replacing jars, and renamed Tomcat's default `ROOT` folder instead of deleting it.

Next up: starting on actual feature work. Questions or suggestions are welcome on [GitHub](https://github.com/sagitdev).
