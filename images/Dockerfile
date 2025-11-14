# syntax=docker/dockerfile:experimental

# Start with the NVIDIA CUDA base image
FROM nvidia/cuda:12.2.0-base-ubuntu22.04 as base

# Disable buffering for Python output
ENV PYTHONUNBUFFERED=1
ENV DEBIAN_FRONTEND=noninteractive

# Install system dependencies, including Python 3.10
RUN apt-get update && apt-get install -y --no-install-recommends software-properties-common \
    && add-apt-repository -y ppa:deadsnakes/ppa \
    && apt-get update && apt-get install --allow-downgrades --allow-change-held-packages --no-install-recommends -y \
    build-essential \
    cmake \
    curl \
    g++ \
    gcc \
    git \
    nvidia-container-toolkit \
    libc-dev \
    libpq-dev \
    libgl1 \
    python3.10 \
    python3.10-dev \
    python3.10-distutils \
    && apt-get clean && rm -rf /var/lib/apt/lists/*


RUN apt-get update -y \
    && apt-get install -y --no-install-recommends sudo lsb-release gpg-agent software-properties-common curl jq unzip \
    && rm -rf /var/lib/apt/lists/*

# Remove unnecessary Python packages
RUN apt-get remove -y python3-blinker python3-cffi-backend

# Create a soft-link for python3.10 to python
RUN ln -s -f /usr/bin/python3.10 /usr/bin/python

# Install pip for Python 3.10
RUN curl https://bootstrap.pypa.io/get-pip.py | python

# Continue from your original Dockerfile
FROM base as build

ARG TARGETOS=linux
ARG RUNNER_VERSION=2.328.0
ARG RUNNER_CONTAINER_HOOKS_VERSION=0.6.1
ARG DOCKER_VERSION=27.1.1
ARG BUILDX_VERSION=0.16.2

WORKDIR /actions-runner

# Install curl and unzip
RUN apt-get update && apt-get install -y curl unzip

# Download and extract GitHub Actions runner
RUN export RUNNER_ARCH=x64 \
    && curl -f -L -o runner.tar.gz https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-${TARGETOS}-${RUNNER_ARCH}-${RUNNER_VERSION}.tar.gz \
    && tar xzf ./runner.tar.gz \
    && rm runner.tar.gz

# Download and set up container hooks
RUN curl -f -L -o runner-container-hooks.zip https://github.com/actions/runner-container-hooks/releases/download/v${RUNNER_CONTAINER_HOOKS_VERSION}/actions-runner-hooks-k8s-${RUNNER_CONTAINER_HOOKS_VERSION}.zip \
    && unzip ./runner-container-hooks.zip -d ./k8s \
    && rm runner-container-hooks.zip

# Download and set up Docker and Buildx
RUN export RUNNER_ARCH=x64 \
    && curl -fLo docker.tgz https://download.docker.com/linux/static/stable/x86_64/docker-${DOCKER_VERSION}.tgz \
    && tar zxvf docker.tgz \
    && rm -rf docker.tgz \
    && mkdir -p /usr/local/lib/docker/cli-plugins \
    && curl -fLo /usr/local/lib/docker/cli-plugins/docker-buildx \
        "https://github.com/docker/buildx/releases/download/v${BUILDX_VERSION}/buildx-v${BUILDX_VERSION}.linux-amd64" \
    && chmod +x /usr/local/lib/docker/cli-plugins/docker-buildx

# Final image
FROM base

ENV DEBIAN_FRONTEND=noninteractive
ENV RUNNER_MANUALLY_TRAP_SIG=1
ENV ACTIONS_RUNNER_PRINT_LOG_TO_STDOUT=1
ENV ImageOS=ubuntu22

# Install essential packages
RUN apt-get update -y \
    && apt-get install -y --no-install-recommends sudo lsb-release gpg-agent software-properties-common curl jq unzip \
    && rm -rf /var/lib/apt/lists/*

# Configure git-core/ppa and install git
RUN add-apt-repository ppa:git-core/ppa \
    && apt-get update -y \
    && apt-get install -y --no-install-recommends git

# Create a non-root user with Docker access
RUN adduser --disabled-password --gecos "" --uid 1001 runner \
    && groupadd docker --gid 123 \
    && usermod -aG sudo runner \
    && usermod -aG docker runner \
    && echo "%sudo   ALL=(ALL:ALL) NOPASSWD:ALL" > /etc/sudoers \
    && echo "Defaults env_keep += \"DEBIAN_FRONTEND\"" >> /etc/sudoers

WORKDIR /home/runner

# Copy the runner and Docker binaries from the build stage
COPY --chown=runner:docker --from=build /actions-runner .
COPY --from=build /usr/local/lib/docker/cli-plugins/docker-buildx /usr/local/lib/docker/cli-plugins/docker-buildx

# Install Docker binaries
RUN install -o root -g root -m 755 docker/* /usr/bin/ && rm -rf docker

USER runner
