# grails_nV Build Guide

This guide explains how to build **grails_nV** (a deliberately vulnerable job listing web application written in Grails) on any machine: Linux, macOS (Intel or Apple Silicon), or Windows, on x86_64 or ARM64.

All paths in this guide are relative to the project root (the folder that contains `application.properties`) unless stated otherwise. Replace placeholders such as `<GRAILS_INSTALL_DIR>` with locations of your choice.

---

## 1. What you are building

| Item | Value |
|------|-------|
| Framework | Grails **2.5.0** (from `application.properties`, key `app.grails.version`) |
| Language | Groovy 2.4 (bundled with Grails 2.5.0) |
| Required JDK | **Java 8 (1.8)** |
| Build tool | The Grails 2 command line (`grails`). No Gradle, no Maven. |
| Dependency resolution | Maven-style resolver configured in `grails-app/conf/BuildConfig.groovy` |
| Database | Embedded H2, stored in files (no external database needed). MySQL is optional, see section 7.5. |
| Schema | Managed by the `database-migration` plugin; migrations run automatically on startup |
| Mail | In-memory GreenMail server (no SMTP setup needed) |
| Output | `target/grails_nV-2.4.1.war` |

> **Important:** Java 9 or newer will NOT work. Grails 2.5.x fails on modern JDKs (module system, removed PermGen flags, ASM/Groovy incompatibilities). Use a Java 8 JDK for every step in this guide, even if a newer JDK is your system default. Java 7 is also not recommended.

---

## 2. Prerequisites

You need:

1. **A Java 8 JDK** (not a JRE).
2. **Grails 2.5.0** distribution.
3. **Internet access** on the first build (plugins and jars are downloaded from `repo.grails.org` and Maven Central, then cached).
4. Around **1 GB of free disk** for caches and build output.
5. `curl` and `unzip` (or equivalents) for the manual steps below.

Docker is an alternative to installing anything locally (see section 6).

### 2.1 Do not use `grailsw` / `grailsw.bat`

The repository ships a Grails wrapper, but `wrapper/grails-wrapper.properties` points to `http://dist.springframework.org.s3.amazonaws.com/release/GRAILS/`, which no longer hosts Grails. The wrapper cannot download Grails, so install Grails 2.5.0 as described below instead.

### 2.2 The README's GVM instructions are outdated

The project README suggests `gvm install grails`. GVM was renamed to SDKMAN, and `install grails` without a version installs a modern, incompatible Grails. Always install version **2.5.0** explicitly.

---

## 3. Install Java 8

Pick ONE option for your platform. Any OpenJDK 8 build works (Eclipse Temurin, Azul Zulu, Amazon Corretto, Liberica, etc.).

### Option A: SDKMAN (Linux, macOS, WSL, Git Bash/Cygwin)

```bash
curl -s "https://get.sdkman.io" | bash
# open a new shell, then:
sdk list java | grep -E "8\.0\."      # pick an identifier from this list
sdk install java <8.x.y-vendor>        # e.g. a "-tem" (Temurin) or "-zulu" (Zulu) 8 build
sdk use java <8.x.y-vendor>
```

### Option B: OS package manager

| Platform | Command |
|----------|---------|
| Debian / Ubuntu | `sudo apt install openjdk-8-jdk` (available on most releases; otherwise use Option A or C) |
| Fedora / RHEL | `sudo dnf install java-1.8.0-openjdk-devel` |
| Arch | `sudo pacman -S jdk8-openjdk` |
| macOS (Homebrew) | `brew install --cask temurin@8` (Intel) or `brew install --cask zulu@8` (Apple Silicon, see note) |
| Windows (winget) | `winget install EclipseAdoptium.Temurin.8.JDK` |
| Windows (Chocolatey) | `choco install temurin8` |

### Option C: Manual download

Download a JDK 8 archive for your OS and CPU from https://adoptium.net/temurin/releases/?version=8 or https://www.azul.com/downloads/?version=java-8-lts, extract it anywhere, and point `JAVA_HOME` at the extracted folder.

**Apple Silicon (M1/M2/M3/...) note:** Eclipse Temurin does not publish a native ARM64 JDK 8 for macOS. Use **Azul Zulu 8** (native ARM64) or run an x86_64 JDK 8 under Rosetta 2. Linux ARM64 (aarch64) JDK 8 builds are available from Temurin, Zulu and Corretto.

### Verify

```bash
java -version
# must print: openjdk version "1.8.0_xxx"
```

---

## 4. Install Grails 2.5.0

### Option A: SDKMAN

```bash
sdk install grails 2.5.0
sdk use grails 2.5.0
```

### Option B: Manual (all platforms)

1. Download https://github.com/grails/grails-core/releases/download/v2.5.0/grails-2.5.0.zip
2. Extract it to a folder of your choice, `<GRAILS_INSTALL_DIR>`. It creates `<GRAILS_INSTALL_DIR>/grails-2.5.0`.

### 4.1 Environment variables

**Linux / macOS / WSL (bash, zsh):**

```bash
export JAVA_HOME=<path-to-your-jdk8>
export GRAILS_HOME=<GRAILS_INSTALL_DIR>/grails-2.5.0
export PATH="$JAVA_HOME/bin:$GRAILS_HOME/bin:$PATH"
```

**Windows (PowerShell):**

```powershell
$env:JAVA_HOME   = "<path-to-your-jdk8>"
$env:GRAILS_HOME = "<GRAILS_INSTALL_DIR>\grails-2.5.0"
$env:Path        = "$env:JAVA_HOME\bin;$env:GRAILS_HOME\bin;$env:Path"
```

**Windows (cmd.exe):**

```bat
set JAVA_HOME=<path-to-your-jdk8>
set GRAILS_HOME=<GRAILS_INSTALL_DIR>\grails-2.5.0
set PATH=%JAVA_HOME%\bin;%GRAILS_HOME%\bin;%PATH%
```

On Windows, the Grails launcher is `grails.bat`; the commands below work the same when typed as `grails ...`.

### 4.2 Verify

```bash
grails -version
# Grails version: 2.5.0
java -version
# openjdk version "1.8.0_xxx"
```

---

## 5. Build

Run every command from the project root.

### 5.1 Compile only

```bash
grails -non-interactive -plain-output compile
```

The first run downloads the Grails plugins listed in `BuildConfig.groovy` (tomcat, codenarc, cache, scaffolding, asset-pipeline, mail, greenmail, markdown, bcrypt, hibernate4, database-migration, jquery) and their dependencies. Later runs use the local caches (`~/.grails` and `~/.m2`, i.e. `%USERPROFILE%\.grails` and `%USERPROFILE%\.m2` on Windows).

### 5.2 Build the WAR

```bash
grails -non-interactive -plain-output dev war
```

Output: `target/grails_nV-2.4.1.war` (about 60 MB). Deploy it on a Servlet 3.0+ container running Java 8 (for example Tomcat 7, 8 or 8.5).

> **Why `dev war` and not `grails war` / `prod war`?**
> `grails war` uses the `production` environment by default. In this project the production datasource in `grails-app/conf/DataSource.groovy` combines `pooled = false` with connection-pool `properties` (`jmxEnabled`, `maxActive`, ...). The WAR compiles fine but **fails at startup** with:
>
> `Invalid property 'jmxEnabled' of bean class [org.springframework.jdbc.datasource.DriverManagerDataSource]`
>
> The development environment uses the same H2 database without that block, and it is the environment the project's README uses (`grails run-app`). Building with `dev war` produces a WAR that starts correctly. The WAR is otherwise a normal deployable WAR.

### 5.3 Optional: initialize the database ahead of time

`initial-setup.sh` runs `grails compile` and `grails dbm-update`. It is optional, because `grails.plugin.databasemigration.updateOnStart = true` in `Config.groovy` applies all migrations automatically when the app starts. If you want to run it:

```bash
grails -non-interactive -plain-output compile
grails -non-interactive -plain-output dbm-update
```

(On Linux/macOS you can also run `chmod +x initial-setup.sh && ./initial-setup.sh`.)

### 5.4 Clean

```bash
grails clean
# or simply delete the target/ folder
```

---

## 6. Build with Docker (no local Java or Grails)

Works on any OS and CPU that runs Docker (the `eclipse-temurin:8-jdk` image is multi-arch: amd64, arm64, and others). From the project root:

**Linux / macOS / WSL:**

```bash
docker run --rm -v "$PWD":/app -w /app eclipse-temurin:8-jdk bash -c '
  set -e
  apt-get update -qq && apt-get install -y -qq unzip curl >/dev/null
  curl -fsSL -o /tmp/grails.zip https://github.com/grails/grails-core/releases/download/v2.5.0/grails-2.5.0.zip
  unzip -q /tmp/grails.zip -d /opt
  export PATH=/opt/grails-2.5.0/bin:$PATH
  grails -non-interactive -plain-output dev war
'
```

**Windows PowerShell:** replace `"$PWD":/app` with `"${PWD}:/app"`.

The WAR appears in `target/` on the host. Files created by the container may be owned by `root` on Linux; delete them with `sudo` or run the container with `--user "$(id -u):$(id -g)" -e HOME=/tmp` (you then need an image that already has `unzip` and `curl`, or download Grails on the host and mount it).

---

## 7. Run

Use a Java 8 runtime for all of these.

### 7.1 Development mode (from source)

```bash
grails run-app
# http://localhost:8080/grails_nV/
```

To use another port: `grails -Dserver.port=9090 run-app`.

### 7.2 Servlet container

Copy `target/grails_nV-2.4.1.war` into the container's deployment folder, renaming it to `grails_nV.war` if you want the context path `/grails_nV` (for Tomcat: `webapps/grails_nV.war`). The container must run on Java 8.

Quick test with Docker (any OS/CPU):

```bash
docker run --rm -p 8080:8080 \
  -v "$PWD/target/grails_nV-2.4.1.war":/usr/local/tomcat/webapps/grails_nV.war:ro \
  tomcat:8.5-jdk8
# http://localhost:8080/grails_nV/
```

### 7.3 Database files

The H2 URL is `jdbc:h2:devDb`, a file database relative to the **current working directory** of the Java process. Running `grails run-app` from the project root creates `devDb.*` files there; a servlet container creates them in its own working directory. Delete those files to reset all data (the seed data in `BootStrap.groovy` is recreated on the next start).

### 7.4 Useful URLs and default accounts

| URL | Purpose |
|-----|---------|
| `/grails_nV/` | Application (redirects to `/grails_nV/user/signin`) |
| `/grails_nV/greenmail` | View mail sent by the app (in-memory GreenMail) |

Accounts created by `grails-app/conf/BootStrap.groovy` on an empty database:

| Email | Password | Access level |
|-------|----------|--------------|
| admin@grails.com | password | 1 (admin) |
| john@grails.com | haX0r! | 2 |
| jack@l33t.com | password | 2 |
| james@megacorp.gov | password | 0 |
| joseph@megacorp.gov | password | 0 |
| jeremy@goatsllc.org | password | 0 |

The sign-in form uses a synchronizer token (`useToken`), so scripted logins must first GET `/grails_nV/user/signin` and send back the `SYNCHRONIZER_TOKEN` and `SYNCHRONIZER_URI` hidden fields along with `signin_email` and `signin_password`.

### 7.5 Optional: MySQL instead of H2

`DataSource.groovy` and `initial-setup.sh` contain commented-out MySQL settings (the MySQL JDBC driver is already a runtime dependency). To use MySQL, uncomment the MySQL `dataSource` block and the `url` lines, comment out the H2 ones, create the database (e.g. `grails_nV-dev`) and rebuild.

---

## 8. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| `Unsupported major.minor version`, `UnsupportedClassVersionError`, `Unrecognized VM option 'MaxPermSize'`, `Could not initialize class org.codehaus.groovy...`, or other strange startup errors | Not running on Java 8. Check `java -version` and `JAVA_HOME` in the same shell. |
| `grailsw` fails to download Grails | The wrapper's download URL is dead (section 2.1). Install Grails 2.5.0. |
| `grails: command not found` | `$GRAILS_HOME/bin` is not on `PATH`. |
| WAR deploys but the app does not start; log shows `Invalid property 'jmxEnabled' ... DriverManagerDataSource` | The WAR was built for the production environment. Rebuild with `grails dev war` (section 5.2). |
| Dependency download errors / SSL errors on first build | Check internet/proxy access to `https://repo.grails.org` and `https://repo1.maven.org`. Behind a proxy, configure it with `grails add-proxy` / `grails set-proxy`, or set `-Dhttps.proxyHost` / `-Dhttps.proxyPort` in `JAVA_OPTS`. |
| `OutOfMemoryError` / `PermGen space` during build | Increase memory, e.g. `export GRAILS_OPTS="-Xmx1g -XX:MaxPermSize=512m"`. |
| Corrupted cache after an interrupted first build | Delete `target/` and `~/.grails/ivy-cache` (or the whole `~/.grails`) and rebuild. |
| H2 `Database may be already in use` / lock errors | Another process (a previous `run-app`, a container) has the `devDb` files open. Stop it, or delete the `devDb.*` files. |
| Port 8080 already in use | `grails -Dserver.port=<free port> run-app`, or map another host port in Docker (`-p <port>:8080`). |

---

## 9. Quick reference

```bash
# 1. Java 8 + Grails 2.5.0 on PATH (sections 3 and 4)
java -version          # 1.8.0_xxx
grails -version        # 2.5.0

# 2. Build
grails -non-interactive -plain-output compile
grails -non-interactive -plain-output dev war     # -> target/grails_nV-2.4.1.war

# 3. Run
grails run-app                                    # http://localhost:8080/grails_nV/
# login: admin@grails.com / password
```
