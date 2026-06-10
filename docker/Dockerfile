# syntax=docker/dockerfile:1.7

ARG UPTIME_KUMA_BASE_IMAGE=louislam/uptime-kuma:base2
ARG UPTIME_KUMA_BUILDER_GO_IMAGE=louislam/uptime-kuma:builder-go

# -----------------------------------------------------------------------------
# Build healthcheck binary
# -----------------------------------------------------------------------------
FROM ${UPTIME_KUMA_BUILDER_GO_IMAGE} AS build_healthcheck

# -----------------------------------------------------------------------------
# Build application from pinned source under app/
# -----------------------------------------------------------------------------
FROM ${UPTIME_KUMA_BASE_IMAGE} AS build

USER node
WORKDIR /app

ENV PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=1
ENV NODE_ENV=development

COPY --chown=node:node app/package.json app/package-lock.json ./
COPY --chown=node:node app/.npmrc .npmrc

RUN npm ci

COPY --chown=node:node app/ ./

COPY --chown=node:node --from=build_healthcheck /app/extra/healthcheck /app/extra/healthcheck

RUN npm run build

RUN npm prune --omit=dev

RUN mkdir -p /app/data

# -----------------------------------------------------------------------------
# Runtime image
# -----------------------------------------------------------------------------
FROM ${UPTIME_KUMA_BASE_IMAGE} AS runtime

WORKDIR /app

ENV NODE_ENV=production
ENV UPTIME_KUMA_IS_CONTAINER=1

LABEL org.opencontainers.image.title="uptime-kuma"
LABEL org.opencontainers.image.description="Uptime Kuma built from pinned upstream source"
LABEL org.opencontainers.image.source="https://github.com/ofekpensso/uptime-kuma-app"
LABEL org.opencontainers.image.vendor="ofekpensso"

COPY --chown=node:node --from=build /app /app

EXPOSE 3001

HEALTHCHECK --interval=60s --timeout=30s --start-period=180s --retries=5 CMD extra/healthcheck

ENTRYPOINT ["/usr/bin/dumb-init", "--"]
CMD ["node", "server/server.js"]