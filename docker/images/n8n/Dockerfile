ARG NODE_VERSION=22.21.0
ARG N8N_VERSION=snapshot
ARG LAUNCHER_VERSION=1.4.1

# ------------------------------------------------------------------------------
# STAGE 1: System Dependencies & Base Setup
# ------------------------------------------------------------------------------
# Use n8nio's base image
FROM n8nio/base:${NODE_VERSION} AS system-deps

# ------------------------------------------------------------------------------
# STAGE 2: Application Artifact Processor
# ------------------------------------------------------------------------------
# Ensure artifacts are built for the target platform (amd64)
FROM --platform=linux/amd64 alpine:3.22.0 AS app-artifact-processor

# Assuming you have a 'compiled' directory in your context
COPY ./compiled /app/

# ------------------------------------------------------------------------------
# STAGE 3: Task Runner Launcher Downloader (CRITICAL FIXES HERE)
# ------------------------------------------------------------------------------
FROM --platform=linux/amd64 alpine:3.22.0 AS launcher-downloader
ARG LAUNCHER_VERSION

# Hardcode architecture to 'amd64' to prevent variable resolution errors.
# We ensure the correct base utilities are present in Alpine.
RUN set -e; \
    apk add --no-cache wget tar coreutils; \
    
    # Define filenames and URLs based on the hardcoded 'amd64' architecture
    TARGET_ARCH="amd64"; \
    FILE_NAME="task-runner-launcher-${LAUNCHER_VERSION}-linux-${TARGET_ARCH}.tar.gz"; \
    
    DOWNLOAD_URL="https://github.com/n8n-io/task-runner-launcher/releases/download/${LAUNCHER_VERSION}/${FILE_NAME}"; \
    SHA_URL="${DOWNLOAD_URL}.sha256"; \
    
    mkdir /launcher-temp && cd /launcher-temp; \
    
    # Download files
    wget -q "$DOWNLOAD_URL"; \
    wget -q "$SHA_URL"; \
    
    # Checksum verification (If this fails, the file on GitHub is updated, but not the SHA256)
    echo "$(cat ${FILE_NAME}.sha256) ${FILE_NAME}" > checksum.sha256; \
    sha256sum -c checksum.sha256; \
    
    # Extract
    mkdir -p /launcher-bin; \
    tar xzf "$FILE_NAME" -C /launcher-bin; \
    cd / && rm -rf /launcher-temp

# ------------------------------------------------------------------------------
# STAGE 4: Final Runtime Image (CRITICAL FIXES HERE)
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

# --- CRITICAL FIX: Ensure all Node dependencies are installed/linked before rebuilding sqlite3 ---
RUN cd /usr/local/lib/node_modules/n8n && \
        npm install --production --ignore-scripts && \
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
			
