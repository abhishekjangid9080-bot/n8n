ARG NODE_VERSION=22.21.0
ARG N8N_VERSION=snapshot
ARG LAUNCHER_VERSION=1.4.1

# ------------------------------------------------------------------------------
# STAGE 1: System Dependencies & Base Setup
# ------------------------------------------------------------------------------
FROM --platform=linux/amd64 node:${NODE_VERSION}-bullseye-slim AS system-deps

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    python3 \
    git \
    wget \
    tar \
    tini \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

ENV NODE_ICU_DATA=/usr/local/lib/node_modules/full-icu
ENV SHELL=/bin/sh

# ------------------------------------------------------------------------------
# STAGE 2: Application Artifact Processor
# ------------------------------------------------------------------------------
FROM --platform=linux/amd64 alpine:3.22.0 AS app-artifact-processor
COPY ./compiled /app/

# ------------------------------------------------------------------------------
# STAGE 3: Task Runner Launcher Downloader
# ------------------------------------------------------------------------------
FROM --platform=linux/amd64 alpine:3.22.0 AS launcher-downloader
ARG LAUNCHER_VERSION

RUN apk add --no-cache wget tar coreutils && \
    TARGET_ARCH="amd64" && \
    FILE_NAME="task-runner-launcher-${LAUNCHER_VERSION}-linux-${TARGET_ARCH}.tar.gz" && \
    DOWNLOAD_URL="https://github.com/n8n-io/task-runner-launcher/releases/download/${LAUNCHER_VERSION}/${FILE_NAME}" && \
    mkdir /launcher-temp && cd /launcher-temp && \
    wget -q "$DOWNLOAD_URL" && \
    tar xzf "$FILE_NAME" -C /launcher-bin && \
    cd / && rm -rf /launcher-temp

# ------------------------------------------------------------------------------
# STAGE 4: Final Runtime Image
# ------------------------------------------------------------------------------
FROM --platform=linux/amd64 system-deps AS runtime

ARG N8N_VERSION
ARG N8N_RELEASE_TYPE=dev

ENV NODE_ENV=production
ENV N8N_RELEASE_TYPE=${N8N_RELEASE_TYPE}

WORKDIR /home/node

COPY --from=app-artifact-processor /app /usr/local/lib/node_modules/n8n
COPY --from=launcher-downloader /launcher-bin/* /usr/local/bin/
COPY docker/images/n8n/docker-entrypoint.sh /
COPY docker/images/n8n/n8n-task-runners.json /etc/n8n-task-runners.json

RUN chown -R node:node /home/node

USER node

RUN cd /usr/local/lib/node_modules/n8n && \
    npm install --production --ignore-scripts && \
    npm rebuild sqlite3 --update-binary && \
    ln -s /usr/local/lib/node_modules/n8n/bin/n8n /usr/local/bin/n8n && \
    mkdir -p /home/node/.n8n

EXPOSE 5678/tcp

ENTRYPOINT ["tini", "--", "/docker-entrypoint.sh"]

LABEL org.opencontainers.image.title="n8n" \
      org.opencontainers.image.description="Workflow Automation Tool" \
      org.opencontainers.image.source="https://github.com/n8n-io/n8n" \
      org.opencontainers.image.url="https://n8n.io" \
      org.opencontainers.image.version=${N8N_VERSION}
			
