# GitHub Actions Workflows

This document explains the two GitHub Actions workflows used in this repository to automate builds, releases, and container image publishing. This repository provides two distinct automated workflows for two different use cases:

1.  **Go Release Workflow**: For developers who want to download and run the application as a native executable on their machine.
2.  **Docker Image Workflow**: For users who prefer to run the application in a containerized environment using Docker for a "ready-to-use-out-of-the-box" experience.

## Go Release Workflow (`.github/workflows/go.yml`)

This workflow automates the creation of GitHub releases with pre-compiled binaries for various operating systems and architectures. The output of this workflow is a set of `.zip` files, each containing a native executable for a specific operating system and CPU architecture. Developers can download the appropriate file for their system, extract it, and run the application directly without needing to install Go or any other dependencies.

### Triggers

This workflow is triggered automatically when a new tag matching the pattern `v*` (e.g., `v1.0.0`, `v2.3.1-beta`) is pushed to the repository.

### Workflow Breakdown

The workflow consists of several jobs that run in parallel to build the application, followed by a final job to create the release.

#### 1. Build Jobs

A separate job is run for each target platform to compile the Go binary. The supported platforms are:

-   **Windows (x86_64):**
    -   Job: `build-windows`
    -   Runner: `windows-latest`
    -   Artifact: `apple-music-downloader_windows-x86_64.zip`
-   **Linux (x86_64):**
    -   Job: `build-linux`
    -   Runner: `ubuntu-latest`
    -   Artifact: `apple-music-downloader_linux-x86_64.zip`
-   **Linux (ARM64):**
    -   Job: `build-linux-arm64`
    -   Runner: `ubuntu-latest` (cross-compiles for ARM64)
    -   Artifact: `apple-music-downloader_linux-arm64.zip`
-   **macOS (Apple Silicon):**
    -   Job: `build-macos-apple-silicon`
    -   Runner: `macos-latest`
    -   Artifact: `apple-music-downloader_macos-apple-silicon.zip`
-   **macOS (Intel):**
    -   Job: `build-macos-intel`
    -   Runner: `macos-15-intel`
    -   Artifact: `apple-music-downloader_macos-intel-x86_64.zip`

Each build job performs the following steps:
1.  Checks out the repository code.
2.  Sets up the Go environment.
3.  Builds the Go binary for the specific target.
4.  Packages the binary along with other necessary files (`agent.js`, `agent-arm64.js`, `config.yaml`, `README.md`) into a `.zip` archive.
5.  Uploads the zip archive as a build artifact.

#### 2. Release Job

-   **Job:** `release`
-   **Depends on:** All `build-*` jobs.

This job runs only after all the build jobs have completed successfully. It performs these steps:
1.  Downloads all the zipped artifacts created by the build jobs.
2.  Uses the `softprops/action-gh-release` action to create a new **draft** GitHub Release.
3.  The release is created based on the pushed tag.
4.  Attaches all the downloaded zip archives as release assets, making them available for download.

## Docker Image Workflow (`.github/workflows/docker.yml`)

This workflow automates the building and publishing of multi-architecture Docker images to Docker Hub. This workflow produces a Docker image that encapsulates the application and its entire environment. This is ideal for users who want a simple, one-step deployment using Docker. The resulting image is self-contained and will run identically across any system with Docker installed, abstracting away the underlying OS details.

### Triggers

This workflow is triggered on a push to the following branches or tags:
-   `main` branch
-   `master` branch
-   Any tag matching the pattern `v*`

### Workflow Breakdown

The workflow consists of a single job, `build-and-push-images`, which handles the entire process.

-   **Runner:** `ubuntu-latest`

#### Steps

1.  **Setup:**
    -   Checks out the repository code.
    -   Sets up QEMU and Docker Buildx to enable multi-platform builds.
2.  **Login:**
    -   Logs into Docker Hub using the `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` secrets stored in the repository settings.
3.  **Versioning:**
    -   A version tag for the Docker image is determined:
        -   If triggered by a **tag** (e.g., `v1.2.3`), the tag itself is used as the version.
        -   If triggered by a **branch push**, the version is `sha-` followed by the short commit SHA (e.g., `sha-1a2b3c4`).
4.  **Build and Push:**
    -   Uses the `docker/build-push-action` to build the image from `Dockerfile.release`. The `Dockerfile.release` is specifically designed to create a minimal, production-ready image.
    -   **Multi-Platform Build:** It builds the image for two different architectures simultaneously:
        -   `linux/amd64`
        -   `linux/arm64`
    -   **Push to Docker Hub:** The resulting multi-arch image is pushed to Docker Hub with two tags:
        -   `<username>/apple-music-downloader:latest`
        -   `<username>/apple-music-downloader:<version>` (using the version generated in the previous step).