---
title: Build Tools
date: 2025-10-23
tags: ["devops", "build-tools", "maven", "gradle", "npm", "docker"]
draft: false
weight: 5
---

**Build tools** are software utilities that automate the process of transforming source code into executable programs. They manage dependencies, run tests to catch bugs early, and package applications into deployable formats such as JAR files, NPM packages, or Docker images.

After setting up my Kubernetes environment and before continuing with the next project, I wanted a better understanding of how applications are built, tested, and packaged before deployment. In this project, I explore how different build tools such as Maven, Gradle, npm, and Docker fit into the DevOps workflow.

## Build Tools

Each programming language has its own preferred set of build tools:

- **Java**: Maven and Gradle dominate. Maven uses XML-based configuration, while Gradle relies on Groovy for more flexible and dynamic builds.
- **JavaScript**: Tools like npm, Webpack, and Vite are commonly used for bundling and managing frontend and backend applications.
- **Python**: Developers typically rely on pip for dependency management, with setuptools handling packaging.


## What Build Tools Do

Building an application means packaging source code into a deployable artifact containing compiled code, dependencies, and metadata—all bundled into one manageable file.

Regardless of the tool, their **core responsibilities** are similar:

- **Install Dependencies**: Automatically download and manage external libraries.
- **Run Tests**: Execute unit and integration tests during the build process.
- **Compile Code**: Convert human-readable source code into machine-readable bytecode.
- **Package Artifacts**: Bundle and compress compiled code into distributable formats like `.jar`, `.tar`, or Docker images.
- **Integrate with CI/CD**: Work with Jenkins, GitLab CI, or GitHub Actions to ensure builds run automatically whenever code changes.


## Maven and Gradle

For Java applications, build tools are part of the workflow. Maven and Gradle both handle building, testing, and dependency management, but they differ in approach:

- **Maven**: Uses **XML** for configuration, follows convention over configuration, and has a fixed project structure and lifecycle. It’s easy to learn but less flexible.
- **Gradle**: Uses **Groovy**, enabling more dynamic and customizable builds. It’s powerful but comes with a steeper learning curve.

Both provide CLI utilities, making them easy to integrate into CI/CD pipelines or run locally.

### Prerequisites

Before working with Maven or Gradle, ensure you have:

- **Java JDK** installed (OpenJDK 21)
- **Maven** (`mvn -v` to verify)
- **Gradle** (`gradle -v` to verify)
- A **Java project** with a `pom.xml` file for Maven or a `build.gradle` for Gradle.

### Building with Maven

Maven organizes builds into **lifecycles**, each with sequential **phases** such as `compile`, `test`, and `package`.

**To create an artifact**:
```bash
mvn clean package
```

This command:
1. Runs the **clean lifecycle**, removing the `target` directory (old compiled files).
2. Executes the **default lifecycle** up to the `package` phase: validating, compiling, testing, and packaging into a JAR file inside `target/`.

### Building with Gradle

Gradle also follows a lifecycle but divides it into three stages:

1. **Initialization**: Reads `settings.gradle` and determines which projects to build.
2. **Configuration**: Evaluates build scripts and creates a task graph.
3. **Execution**: Runs tasks in the correct order.

**To build an artifact**:
```bash
gradle clean build
```

This command:
- Runs the **clean** task, removing the `build/` directory.
- Executes the **build** task, resulting in a JAR artifact stored in `build/libs/`.

### Running a Java Artifact

After building, artifacts can be tested locally or deployed to servers.

**Run JAR files**:
```bash
# Maven
java -jar target/<APPLICATION_NAME>.jar  

# Gradle
java -jar build/libs/<APPLICATION_NAME>.jar
```

This starts the Java application directly from the artifact, perfect for server setups where only Java needs to be installed.


## JavaScript Build Tools

Unlike Java, JavaScript applications don’t produce JAR or WAR files. Instead, they are often packaged as **TAR** files.

Dependency management relies on:

- **npm (Node Package Manager)**, the most common.
- **Yarn**, a popular alternative.

Both use the `package.json` file, which defines dependencies, scripts, and metadata. However, **npm and Yarn are package managers, not build tools**. They install dependencies but don’t transpile or bundle code by themselves.

### Backend Builds (Node.js)

Node.js backend projects often define build scripts in the `package.json` file.

**Installing Webpack for bundling**:
```bash
npm install --save-dev webpack webpack-cli
```

**Adding a build script to `package.json`**:
```json
"scripts": {
  "build": "webpack --mode production --target=node --entry ./server.js --output-path ./dist --output-filename server.js"
}
```

**Run the build command**:
```
npm run build
```
This command build, bundles and optimizes `server.js` and its dependencies into a production-ready file in `dist/` directory.

### Frontend Builds (React)

Frontend applications (React, Vue, Angular) need extra steps to ensure browser compatibility:

- **Transpilation**: Convert modern JavaScript into browser-supported syntax.
- **Bundling**: Combine multiple files.
- **Minification**: Compress assets for performance.

React projects come with a build script preconfigured:
```json
"scripts": {
  "build": "vite build"
}
```

**Build the application**:
```bash
npm run build
```
This command bundles, transpiles, and optimizes the app into a ready-to-deploy production build.


## Docker the Universal Build Artifact

Traditionally, different applications required different artifact types: JARs for Java, TARs for JavaScript, etc. This complicated deployment, Docker simplifies this by standardizing all builds into **Docker images**.

### Docker Images

A Docker image contains everything the application needs: code, dependencies, and runtime. Instead of maintaining artifact repositories for JAR, WAR, and TAR files, you only need a **Docker registry** (Docker Hub, AWS ECR, Nexus) to pull and start the application.

### Dockerizing Applications

With Docker, this build process becomes uniform. You define a Dockerfile, and Docker packages everything into a **Docker image**, and that image is what gets deployed.

#### Java

First, build a JAR file with Maven or Gradle, then package it into a Docker image with a Dockerfile.

**Create a Dockerfile**:
```dockerfile
FROM openjdk:21-jdk-slim  
WORKDIR /app  
COPY target/maven-0.0.1-SNAPSHOT.jar /app  
EXPOSE 8090  
CMD ["java", "-jar", "maven-0.0.1-SNAPSHOT.jar"]
```
- `FROM`: Uses `openjdk` version `21` as the base image.
- `WORKDIR`: Sets `/app` as the working directory inside the container.
- `COPY`: Copies the JAR file from the `target` directory into the container's `/app` directory.
- `EXPOSE`: Open port `8090` on the container.
- `CMD`: Starts the run command when the container starts.

**To build the image, run**:
```bash
docker build -t java-maven .
```
- `docker build`: Command to build a Docker image from a Dockerfile.
- `-t java-maven`: Tags the image with the name `java-maven`.
- `.`: Specifies the location (current location), where Docker looks for the Dockerfile.

**To run the Docker image as a container**:
```bash
 docker run -p 8080:8090 java-maven:latest 
```
- `docker run`: Creates and starts a new container from the image.
- `-p 8080:8090`: Port mapping flag that forwards traffic:
    - `8080`: Is the port on your host machine (your computer).
    - `8090`: Is the port inside the container.

Requests to `localhost:8080` in the browser get forwarded to port 8090 inside the Docker container.

#### JavaScript

For the JavaScript project, I will use the React applications. You can read the build steps for the Node.js project in the [README](https://gitlab.com/devops8614042/build-tools/-/tree/main/nodejs?ref_type=heads) on GitLab.

**For Node.js apps**:
```dockerfile
FROM node:22-alpine  
WORKDIR /app  
COPY package.json package-lock.json* ./  
RUN npm install  
COPY . .  
EXPOSE 6000  
CMD ["npm", "run", "start"]
```

**To build the image, run**:
```bash
docker build -t react .
```
- `docker build`: Command to build a Docker image from a Dockerfile.
- `-t`: Tags the image with the name `react`.
- `.`: Specifies the location (current location), where Docker looks for the Dockerfile.

**To run the Docker image as a container**:
```bash
docker run --name react-app -p 5173:5173 react:0.0.1
```

With Docker, deployment becomes consistent: build the app, package it as an image, and run it anywhere with a single `docker run` command. No need to install Node.js or Java on servers; the container brings its own runtime.

### Docker Compose

Docker Compose is a build tool that simplifies the management of multiple containers by defining them in a single configuration file. Instead of starting each container individually with its own configuration, Docker Compose lets you treat them as a unified service. This improves efficiency in deployment, scaling, and overall management.

#### Starting Containers with Docker Compose

To start containers with Docker Compose, you create a `docker-compose.yml` file. This file defines all the services (containers) you want to run, including the Docker images they are built from, their networks, and exposed ports.

In the [Docker Compose file](https://gitlab.com/devops8614042/build-tools/-/blob/main/docker-compose.yml), I defined how to:
- Build Docker images from their respective Dockerfiles.
- Create a shared network for communication between the React and Node.js containers.
- Expose ports so the services can be accessed from a browser.

**Run the containers in one single command**:
```bash
docker compose -f docker-compose.yml up -d
```

**To stop the containers**:
```bash
docker compose down
```


## Bash Script

To make the workflow even easier, I created a [Bash script](https://gitlab.com/devops8614042/build-tools/-/blob/main/automation-build.sh) that automates the entire process. It builds the JAR files, starts Docker Compose, that creates the Docker images, and starts all four containers with a single command.

**Run the script**:
```bash
# make the script executable
chmod u+x automation-build.sh 
# start the script
./automation-build.sh
```


## What I Learned

Completing this project gave me a better understanding of how build tools fit into the bigger DevOps picture. Some of the key takeaways for me were:

- **Maven vs. Gradle**: I already used Maven extensively in my daily work, but experimenting with Gradle showed me how flexible and powerful it can be, especially for highly customizable builds.

- **JavaScript Build Tools**: When I first learned React, npm was just the tool I used to start development scripts. Now, I better understand how npm, Webpack, and Vite handle packaging, bundling, and optimizing applications for production.

- **Docker**: This was the biggest step forward. Before the bootcamp, I only knew Docker at a surface level. Now I can package applications into images, run them consistently on any environment, and even orchestrate multiple services with Docker Compose.

- **Automation**: Writing a Bash script to tie everything together showed me how powerful automation can be. Instead of running a dozen commands manually, I can now build and deploy a complete stack with a single script.

This project helped me better understand the difference between what I already know as a Full-Stack Developer and the DevOps principles I learned during the bootcamp. It also gives me a solid foundation for building the next project with Nexus.

## Conclusion

Modern build tools like **Maven**, **Gradle**, **npm**, **Webpack**, and **Vite** automate the **building**, testing, and packaging of applications.

When combined with Docker, they **standardize delivery** and create a consistent workflow across different languages and environments.

Together, they form the foundation of a **reliable**, **automated DevOps workflow** that simplifies the entire development, testing, and deployment process.

## GitLab

> [**Build Tools Repository on GitLab**](https://gitlab.com/devops8614042/build-tools)