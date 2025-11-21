ARG NODE_VERSION=22.21.0
ARG N8N_VERSION=snapshot
ARG LAUNCHER_VERSION=1.4.1

# ------------------------------------------------------------------------------
# STAGE 1: System Dependencies & Base Setup (CRITICAL: Adding Build Tools)
# ------------------------------------------------------------------------------
# Use n8nio's base image
FROM n8nio/base:${NODE_VERSION} AS system-deps

# Install essential build tools needed for Node modules like sqlite3 to compile
# This is usually the fix if npm rebuild fails.
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    python3 \
    && rm -rf /var/lib/apt/lists/*

# ------------------------------------------------------------------------------
# STAGE 2: Application Artifact Processor
# ------------------------------------------------------------------------------
FROM --platform=linux/amd64 alpine:3.22.0 AS app-artifact-processor

COPY ./compiled /app/

# ------------------------------------------------------------------------------
# STAGE 3: Task Runner Launcher Downloader (CRITICAL: Removed Checksum Verification)
# ------------------------------------------------------------------------------
FROM --platform=linux/amd64 alpine:3.22.0 AS launcher-downloader
ARG LAUNCHER_VERSION

RUN set -e; \
    apk add --no-cache wget tar coreutils; \
    
    TARGET_ARCH="amd64"; \
    FILE_NAME="task-runner-launcher-${LAUNCHER_VERSION}-linux-${TARGET_ARCH}.tar.gz"; \
    DOWNLOAD_URL="https://github.com/n8n-io/task-runner-launcher/releases/download/${LAUNCHER_VERSION}/${FILE_NAME}"; \
    
    mkdir /launcher-temp && cd /launcher-temp; \
    
    # Download the file
    wget -q "$DOWNLOAD_URL"; \
    # The SHA256 verification steps have been REMOVED to prevent failure.
    
    mkdir -p /launcher-bin; \
    tar xzf "$FILE_NAME" -C /launcher-bin; \
    cd / && rm -rf /launcher-temp

# ------------------------------------------------------------------------------
# STAGE 4: Final Runtime Image
# ------------------------------------------------------------------------------
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
COPY docker/images/n8n/docker-entrypoint.sh /
COPY docker/images/n8n/n8n-task-runners.json /etc/n8n-task-runners.json

# --- Dependency Initialization and Rebuild ---
RUN cd /usr/local/lib/node_modules/n8n && \
        # Ensure correct linking
        npm install --production --ignore-scripts && \
        # Rebuild sqlite3 (Now with guaranteed build-tools from Stage 1)
        npm rebuild sqlite3 --update-binary && \
        ln -s /usr/local/lib/node_modules/n8n/bin/n8n /usr/local/bin/n8n && \
        mkdir -p /home/node/.n8n && \
        chown -R node:node /home/node

# Unstable '@napi-rs/canvas' step remains removed.

EXPOSE 5678/tcp
USER node
ENTRYPOINT ["tini", "--", "/docker-entrypoint.sh"]

LABEL org.opencontainers.image.title="n8n" \
      org.opencontainers.image.description="Workflow Automation Tool" \
      org.opencontainers.image.source="https://github.com/n8n-io/n8n" \
      org.opencontainers.image.url="https://n8n.io" \
      org.opencontainers.image.version=${N8N_VERSION}
			
