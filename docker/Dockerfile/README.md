# Building Images with Dockerfiles & Best Practices

This guide covers the process of creating Docker images using a `Dockerfile`, the distinction between different build instructions, and a comprehensive list of best practices for creating secure, efficient, and maintainable images.

## Methods for Creating Docker Images

There are two primary ways to create a Docker image.

### 1. Using a `Dockerfile` (Recommended Method)

A `Dockerfile` is a text file that contains a series of instructions on how to build a Docker image. This is the standard, recommended method because it is:
*   **Reproducible:** Anyone can recreate the exact same image from the `Dockerfile`.
*   **Version-Controlled:** You can track changes to your image's definition in Git.
*   **Automated:** It's perfect for CI/CD pipelines.

### 2. Using `docker container commit` (For Debugging or Quick Snapshots)

You can also create an image from the current state of a running container. This is useful for debugging or saving a specific state, but it is not recommended for production workflows because the process is manual and not easily reproducible.

```bash
# Syntax: docker container commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]
docker container commit my_running_container my_app_image:snapshot-v1
```

## Optimizing Build Context with `.dockerignore`

The `.dockerignore` file works just like a `.gitignore` file. It allows you to specify a list of files and directories that should be excluded from the build context sent to the Docker daemon.

**Why is this important?**
*   **Faster Builds:** Prevents large or unnecessary files (like `node_modules`, log files, or `.git`) from being sent to the daemon, speeding up the `docker build` process.
*   **Smaller Images:** Avoids accidentally copying unneeded files into your image.
*   **Security:** Prevents sensitive files or credentials from being included in the image layers.

## Build-Time vs. Run-Time Instructions

Understanding the difference between instructions is crucial for writing effective Dockerfiles.

| Instruction          | Execution Time      | Purpose                                                                       |
| -------------------- | ------------------- | ----------------------------------------------------------------------------- |
| **`RUN`**            | **Build-Time**      | Executes a command and creates a new layer in the image. Used for installing packages, compiling code, etc. |
| **`CMD` & `ENTRYPOINT`** | **Run-Time**        | Defines the default command that will be executed when a container is started from the image. |

-   Use `RUN` for setting up the image environment.
-   Use `ENTRYPOINT` and/or `CMD` to specify what the container does when it runs.

## The Build and Run Workflow

### Step 1: Build the Image from a `Dockerfile`

The `docker image build` command builds an image from a `Dockerfile` and a "context". The context is the set of files at the specified PATH or URL.

```bash
# Syntax: docker image build -t <repository_name>:<tag> <path_to_context>
docker image build -t my-web-app:v1.0 .
```
*   `-t` assigns a tag (name and version) to the image.
*   `.` specifies that the build context is the current directory.

> **Note:** Dockerfile builds are non-interactive. You must pre-configure all answers to installation prompts or user inputs directly within the `Dockerfile` syntax (e.g., using `ARG` or passing flags like `-y` to package managers).

### Step 2: Run a Container from the Image

After building the image, you can create and run a container from it.

```bash
# Syntax: docker run [OPTIONS] IMAGE[:TAG]
docker run -d -p 8080:80 my-web-app:v1.0
```

---

## Dockerfile Best Practices

Follow these guidelines to create professional-grade Docker images.

### 1. Run as a Non-Root User
By default, containers run as the `root` user. This is a security risk. If an attacker compromises your application, they will have root access inside the container. Always create a dedicated user and group to run your application.

```Dockerfile
# Create a dedicated user and group
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

# ... copy application files and set ownership ...
RUN chown -R appuser:appgroup /app

# Switch to the non-root user
USER appuser
```

### 2. Avoid Hardcoding UIDs
Do not bind a specific User ID (UID) in your Dockerfile (e.g., `useradd -u 1001`). Some container platforms like OpenShift run containers with arbitrary UIDs for security reasons. Your image should be flexible enough to run with any UID. The official NGINX and Prometheus Dockerfiles are excellent examples of how to handle permissions correctly.

### 3. Use Multi-Stage Builds
Multi-stage builds allow you to use multiple `FROM` statements in your Dockerfile. This lets you separate the build environment (with all its dependencies, compilers, and tools) from the final runtime environment. The result is a significantly smaller, more secure final image.

```Dockerfile
# --- Build Stage ---
FROM golang:1.17 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp .

# --- Final Stage ---
FROM alpine:latest
WORKDIR /root/
# Copy only the compiled binary from the builder stage
COPY --from=builder /app/myapp .
CMD ["./myapp"]
```

### 4. Use Minimal Base Images (`distroless` or `scratch`)
Start with the smallest possible base image that can run your application.
-   **`distroless`**: Google's distroless images contain only your application and its runtime dependencies. They do not contain package managers, shells, or other utilities, which drastically reduces the attack surface.
-   **`FROM scratch`**: An empty image, perfect for statically compiled binaries (like Go) that have no external dependencies.

### 5. Start with Trusted Base Images
Always use official images from trusted registries (like Docker Hub's official images) as your base. These images are regularly scanned for vulnerabilities and maintained by the community or parent organization.

### 6. Rebuild and Update Images Frequently
Dependencies and base images receive security patches over time. Regularly rebuild your images to ensure you are including the latest security updates. Automate this process in your CI/CD pipeline.

### 7. Use `EXPOSE` for Documentation
The `EXPOSE` instruction does not actually publish the port. It functions as a type of documentation between the person who builds the image and the person who runs it, about which ports are intended to be published. To actually open a port, you must use the `-p` or `-P` flag on `docker run`.

```Dockerfile
EXPOSE 8080
```

### 8. Keep Credentials and Secrets Out of the Image
**Never** hardcode credentials, tokens, or any confidential information in your `Dockerfile`. Use run-time mechanisms to provide secrets to your container:
-   Environment variables (`docker run -e`)
-   Docker secrets
-   Build-time arguments (`ARG`) for non-sensitive data

### 9. Optimize Layer Caching
Docker builds images in layers. It caches each layer and will reuse it if the instruction has not changed. To optimize build speed:
-   **Order matters:** Place instructions that change less frequently (like installing dependencies with `apt-get`) *before* instructions that change often (like `COPY . .`).
-   Consolidate `RUN` commands with `&&` to reduce the number of layers.

### 10. Add Metadata with `LABEL`
Use the `LABEL` instruction to add metadata to your image, such as the maintainer, version, or a link to the source code repository. This helps with organization and automation.

```Dockerfile
LABEL maintainer="Arash Alghasi <youremail@example.com>" \
      version="1.0" \
      description="This is a sample web application."
```

### 11. Scan Images for Vulnerabilities
Integrate security scanning into your development workflow ("shift-left" security). Scan your images locally before pushing them to a registry. Many container registries, like **Harbor**, include built-in vulnerability scanners that can check your images for known CVEs.