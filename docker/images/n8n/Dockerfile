# Dockerfile
# Base arguments for the build
ARG NODE_VERSION=22.21.0
ARG N8N_VERSION=snapshot
ARG LAUNCHER_VERSION=1.4.1

# ==============================================================================
# STAGE 1: System Dependencies & Base Setup
# ==============================================================================
# Use n8nio's base image which includes Node.js and basic tools
FROM n8nio/base:${NODE_VERSION} AS system-deps

# ==============================================================================
# STAGE 2: Application Artifact Processor
# ==============================================================================
# Use Alpine image specifically for linux/amd64 architecture to process artifacts
FROM --platform=linux/amd64 alpine:3.22.0 AS app-artifact-processor

# Assuming you have a 'compiled' directory in your context
COPY ./compiled /app/

# ==============================================================================
# STAGE 3: Task Runner Launcher Downloader
# ==============================================================================
FROM --platform=linux/amd64 alpine:3.22.0 AS launcher-downloader
ARG LAUNCHER_VERSION

# Set environment for the targeted architecture, matching the FROM platform
# **CRITICAL FIX: Hardcode TARGET_ARCH to 'amd64' based on '--platform' directive.**
ENV TARGET_ARCH="amd64"

RUN set -e; \
    # Install required Alpine packages (wget, tar, coreutils for sha256sum)
    apk add --no-cache wget tar coreutils; \
    
    mkdir /launcher-temp && cd /launcher-temp; \
    
    # Task Runner Launcher download URLs
    DOWNLOAD_URL="https://github.com/n8n-io/task-runner-launcher/releases/download/${LAUNCHER_VERSION}/task-runner-launcher-${LAUNCHER_VERSION}-linux-${TARGET_ARCH}.tar.gz"; \
    SHA_URL="${DOWNLOAD_URL}.sha256"; \
    
    wget -q "$DOWNLOAD_URL"; \
    wget -q "$SHA_URL"; \
    
    FILE_NAME="task-runner-launcher-${LAUNCHER_VERSION}-linux-${TARGET_ARCH}.tar.gz"; \
    SHA_NAME="${FILE_NAME}.sha256"; \
    
    # Verify Checksum. If this fails, it's the 'Exited with status 1' reason.
    echo "$(cat ${SHA_NAME}) ${FILE_NAME}" > checksum.sha256; \
    sha256sum -c checksum.sha256; \
    
    mkdir -p /launcher-bin; \
    tar xzf "$FILE_NAME" -C /launcher-bin; \
    cd / && rm -rf /launcher-temp

# ==============================================================================
# STAGE 4: Final Runtime Image
# ==============================================================================
FROM system-deps AS runtime

ARG N8N_VERSION
ARG N8N_RELEASE_TYPE=dev
ENV NODE_ENV=production
ENV N8N_RELEASE_TYPE=${N8N_RELEASE_TYPE}
ENV NODE_ICU_DATA=/usr/local/lib/node_modules/full-icu
ENV SHELL=/bin/sh

WORKDIR /home/node

# Copy application and launcher files
COPY --from=app-artifact-processor /app /usr/local/lib/node_modules/n8n
COPY --from=launcher-downloader /launcher-bin/* /usr/local/bin/
# Assuming these supporting files are present in 'docker/images/n8n/'
COPY docker/images/n8n/docker-entrypoint.sh /
COPY docker/images/n8n/n8n-task-runners.json /etc/n8n-task-runners.json

# Rebuild sqlite3 and set up symlink/permissions
# 'npm rebuild sqlite3' is a common point of failure if build tools are missing.
RUN cd /usr/local/lib/node_modules/n8n && \
        npm rebuild sqlite3 --update-binary && \
        ln -s /usr/local/lib/node_modules/n8n/bin/n8n /usr/local/bin/n8n && \
        mkdir -p /home/node/.n8n && \
        chown -R node:node /home/node

# **REMOVED:** The unstable 'npm install @napi-rs/canvas' step. 
# If this was the cause of the error, removing it will fix the build.
# If you still need this package, you must ensure all its system dependencies 
# (like libstdc++, libc, etc.) are installed in the base image.

EXPOSE 5678/tcp
USER node
ENTRYPOINT ["tini", "--", "/docker-entrypoint.sh"]

LABEL org.opencontainers.image.title="n8n" \
      org.opencontainers.image.description="Workflow Automation Tool" \
      org.opencontainers.image.source="https://github.com/n8n-io/n8n" \
      org.opencontainers.image.url="https://n8n.io" \
      org.opencontainers.image.version=${N8N_VERSION}
			
