# Chainguard Libraries for Java Workshop
- [Chainguard Libraries for Java Workshop](#chainguard-libraries-for-java-workshop)
  - [🧩 Overview](#-overview)
  - [🧰 Prerequisites](#-prerequisites)
  - [⚙️ Setup](#️-setup)
  - [🧱 Step 1 — Baseline Build (Upstream Maven Central)](#-step-1--baseline-build-upstream-maven-central)
    - [1. Inspect dependencies](#1-inspect-dependencies)
    - [2. View the Dockerfile](#2-view-the-dockerfile)
    - [3. Set build variables](#3-set-build-variables)
    - [4. Build the image](#4-build-the-image)
    - [5. Run and test](#5-run-and-test)
    - [6. Copy the built JAR and scan](#6-copy-the-built-jar-and-scan)
    - [7. Cleanup](#7-cleanup)
  - [🧱 Step 2 — Build Using Chainguard Libraries](#-step-2--build-using-chainguard-libraries)
    - [1. Create a pull token](#1-create-a-pull-token)
    - [2. Review `settings.xml`](#2-review-settingsxml)
    - [3. Set build variables](#3-set-build-variables-1)
    - [4. Review the updated Dockerfile](#4-review-the-updated-dockerfile)
    - [5. Build using Chainguard Libraries](#5-build-using-chainguard-libraries)
    - [6. Run, test, and scan](#6-run-test-and-scan)
    - [7. Cleanup](#7-cleanup-1)
  - [🧹 Final Cleanup](#-final-cleanup)
  - [🧩 Troubleshooting Tips](#-troubleshooting-tips)


This walkthrough demonstrates:
1. How to rebuild an existing Java application using Chainguard’s verified Maven repository.
2. How to compare dependency sourcing before and after migration.

**Note:** To run the guided demo magic script instead of following this readme, simply execute the `demo.sh` script in this directory.

---

## 🧩 Overview

We’ll work through two main stages:

1. **Baseline Build (Upstream Maven & Temurin Images):**  
   Build an existing Java app using upstream Maven and Eclipse Temurin images with dependencies from Maven Central.

2. **Chainguard Libraries Migration (Same Upstream Builders):**  
   Build the same app using Chainguard Libraries for dependencies, while keeping the same upstream Maven and Temurin images.

For this demo all builds will use **containerized environments** to avoid needing to have a local Maven.

---

## 🧰 Prerequisites

- **chainctl** is installed and user has access to the chainguard org with java ecosystem entitlements. 
> NOTE: The user must have the `libraries.java.pull` role in order to access libraries from `https://libraries.cgr.dev/java/` and `https://libraries.cgr.dev/java-remediated/` e.g. `owner` role. Chainctl install docs can be found [here](https://edu.chainguard.dev/chainguard/chainctl-usage/how-to-install-chainctl/)
- **chainctl** CLI tool to scan artifacts.  chainctl install docs can be found [here](https://edu.chainguard.dev/chainguard/chainctl-usage/how-to-install-chainctl/)
- **jq** installed for JSON parsing  
- A **Chainguard organization name** (e.g. `myorg.com`) that has Java ecosystem entitlements. 
- Network Access to `https://libraries.cgr.dev/java/` and `https://libraries.cgr.dev/java-remediated/`
- Docker installed to run images
---

## ⚙️ Setup
Start by setting some environment variables that will be needed for the demo. 

```bash
# Ecosystem to create a token for, in this case java
ECOSYSTEM="java"
# Name associated with the token
TOKEN_NAME="java-libraries-workshop-token-$USER"
# How long the token should be valid for in hours
TTL="8760h"
# The name of the Chainguard organizaiton with entitlements to the ecosystem.
export ORG_NAME="<YOUR Chainguard ORG name>"
```

---

## 🧱 Step 1 — Baseline Build (Upstream Maven Central)
This first stage builds and runs the application exactly as most Java developers would today — using upstream Maven and Eclipse Temurin images and pulling all dependencies from Maven Central.
It provides a baseline for functionality and scanning before any Chainguard content is introduced.


### 1. Inspect dependencies
View the Java dependencies in the pom file:

```bash
# Change Directory into the step1-orig folder
cd chainguard-libraries-java/step1-orig

# Print only the <dependencies> section from pom.xml to see what the app depends on.
awk '/<dependencies>/,/<\/dependencies>/' pom.xml
```

### 2. View the Dockerfile
View the dockerfile, this is a straightforward dockerfile that builds the app in the first stage and copies it to the runtime image.

```bash
cat Dockerfile
```

### 3. Set build variables

```bash
# Define image tags and names for the baseline build.
TAG="upstream"
BUILDER_IMAGE="maven"
RUNTIME_IMAGE="eclipse-temurin"
```

### 4. Build the image

```bash
# Build the app using the upstream Maven builder and Temurin runtime.
docker build \
  --build-arg BASE_IMAGE=$BUILDER_IMAGE \
  --build-arg RUNTIME_IMAGE=$RUNTIME_IMAGE \
  -t java-lib-example:$TAG .
```

### 5. Run and test

```bash
# Start the container and expose port 8081.
docker run -d -p 8081:8080 --name java-lib-example-1 java-lib-example:$TAG

# Verify that the app is responding correctly.
curl http://localhost:8081
```

### 6. Copy the built JAR and scan

```bash
# Copy the generated JAR file out of the running container.
docker cp java-lib-example-1:/app/java-demo-app-1.0.0.jar .

# Analyze the JAR with chainctl to confirm dependencies come from Maven Central.
chainctl libraries verify --parent $ORG_NAME java-demo-app-1.0.0.jar
```

### 7. Cleanup

```bash
# Stop and remove the running container, and delete the copied JAR.
docker stop java-lib-example-1 && docker rm java-lib-example-1 && rm java-demo-app-1.0.0.jar
```

---

## 🧱 Step 2 — Build Using Chainguard Libraries
In this step, you’ll rebuild the same application but redirect Maven to pull its dependencies from Chainguard Libraries.
The build and runtime containers remain the same, proving that you can switch dependency sources without breaking builds or changing code.
After the build, you’ll scan to verify that dependencies now originate from Chainguard’s repository.

### 1. Create a pull token

```bash
# Request a Chainguard library token for the Java ecosystem.
CREDS_OUTPUT=$(chainctl auth pull-token \
  --library-ecosystem="${ECOSYSTEM}" \
  --parent="${ORG_NAME}" \
  --name="${TOKEN_NAME}" \
  --ttl="${TTL}" \
  -o json)

# Extract credentials for Maven authentication.
export CGR_MAVEN_USER=$(echo $CREDS_OUTPUT | jq -r ".identity_id")
export CGR_MAVEN_PASS=$(echo $CREDS_OUTPUT | jq -r ".token")
```

### 2. Review `settings.xml`
View the settings.xml file which shows that we are setting `https://libraries.cgr.dev/java-remediated/` as the primary repo and `https://libraries.cgr.dev/java/` as the second, with Maven Central disabled (`invalid` URL). The same repos are set under both `<repositories>` and `<pluginRepositories>`, since Maven resolves plugins and their dependencies (e.g. the Spring jars `spring-boot:repackage` bundles) through `<pluginRepositories>`.  Notice that we are referencing the credentials as environment variables that will be passed in as docker build secrets when building the image.

```bash
# Show settings.xml, which references Chainguard’s Maven repo and uses your credentials.
cat settings.xml

```

### 3. Set build variables

```bash
# Define a new tag to differentiate this build.
TAG="upstream-cg-libs"
BUILDER_IMAGE="maven"
RUNTIME_IMAGE="eclipse-temurin"
# Settings path for the upstream Maven image (the Dockerfile defaults to the Chainguard Maven image path)
MAVEN_SETTINGS_PATH="/usr/share/maven/conf/settings.xml"
```

### 4. Review the updated Dockerfile
Notice here that we have modified the `RUN` statement for the build to reference the build secrets for the credentials to authenticate to `https://libraries.cgr.dev/`. We also modified the maven command to `mvn clean package -U` the -U flag ignores the local cache to ensure we pick up all available dependencies from Chainguard Libraries. The Dockerfile defaults to the Chainguard Maven and JRE images, and to `/usr/share/java/maven/conf/settings.xml`, the path the Chainguard Maven image reads its settings from.

```bash
# View the Dockerfile configured to use build secrets for credentials.
cat ../step2-cg-build/Dockerfile
```

### 5. Build using Chainguard Libraries

```bash
# Build the app using Maven with Chainguard Libraries for dependency resolution.
docker build \
  -t java-lib-example:$TAG \
  --build-arg BUILDER_IMAGE=$BUILDER_IMAGE \
  --build-arg RUNTIME_IMAGE=$RUNTIME_IMAGE \
  --build-arg MAVEN_SETTINGS_PATH=$MAVEN_SETTINGS_PATH \
  --secret id=cgr_user,src=<(echo -n "$CGR_MAVEN_USER") \
  --secret id=cgr_pass,src=<(echo -n "$CGR_MAVEN_PASS") \
  -f ../step2-cg-build/Dockerfile .
```

### 6. Run, test, and scan

```bash
# Start the container and verify it still functions as before.
docker run -d -p 8081:8080 --name java-lib-example-1 java-lib-example:$TAG
curl http://localhost:8081

# Copy the JAR file out for scanning with chainctl.
docker cp java-lib-example-1:/app/java-demo-app-1.0.0.jar .

# Scan with chainctl to determine dependency coverage sourced from Chainguard.
chainctl libraries verify --parent $ORG_NAME java-demo-app-1.0.0.jar
```

### 7. Cleanup

```bash
# Stop and remove the container, then delete the JAR file.
docker stop java-lib-example-1 && docker rm java-lib-example-1 && rm java-demo-app-1.0.0.jar
```

---

## 🧹 Final Cleanup
This removes temporary tokens, images, and jar artifacts created during the workshop.

```bash
# Look up and delete the temporary identity used for the Chainguard pull token.
ID=$(chainctl iam ids ls --parent="$ORG_NAME" -o json | jq -r --arg name "$TOKEN_NAME" '.items[] | select(.name | startswith($name)) | .id')
chainctl iam identities delete "$ID" --parent "$ORG_NAME" --yes

# Remove the Docker images that were created
docker image ls | grep java-lib-example | awk '{print $3}' | xargs docker image rm $1
```
---

## 🧩 Troubleshooting Tips

- **Docker BuildKit required:**  
  Make sure BuildKit is enabled (`DOCKER_BUILDKIT=1 docker build …`) when using secrets.
- **Credential issues:**  
  If you see 401 or 403 errors from `libraries.cgr.dev`, ensure the `CGR_MAVEN_USER` and `CGR_MAVEN_PASS` environment variables are correctly set.
- **Port conflicts:**  
  If port `8081` is in use, update the `docker run` command (e.g., `-p 8082:8080`).
- **Maven Central fallback:**  
  If a large number of dependencies still resolve from Maven Central, in the Chainguard builds, verify your `settings.xml`, credentials, and clear any Artifactory cache. If the build log shows no `libraries.cgr.dev` downloads at all, `settings.xml` was copied to a path the builder image does not read; check `MAVEN_SETTINGS_PATH`.
