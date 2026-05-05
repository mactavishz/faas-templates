FROM --platform=${TARGETPLATFORM:-linux/amd64} ghcr.io/openfaas/of-watchdog:0.11.5 AS watchdog
FROM --platform=${TARGETPLATFORM:-linux/amd64} node:24-bookworm-slim AS ship

ARG TARGETPLATFORM
ARG BUILDPLATFORM

COPY --from=watchdog /fwatchdog /usr/bin/fwatchdog
RUN chmod +x /usr/bin/fwatchdog

RUN apt-get update -qy \
    && if [ "${UPGRADE_PACKAGES}" = "true" ] || [ "${UPGRADE_PACKAGES}" = "1" ]; then apt-get upgrade -qy; fi \
    && apt-get install -qy --no-install-recommends curl ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Add non root user
RUN addgroup --system app \
    && adduser app --system --ingroup app  --home /home/app \
    && chown app:app /home/app

# Turn down the verbosity to default level.
ENV NPM_CONFIG_LOGLEVEL=warn

RUN chmod 777 /tmp

USER app

RUN mkdir -p /home/app/function

# Entrypoint
WORKDIR /home/app
COPY --chown=app:app package.json ./

# This ordering means the npm installation is cached for the outer function handler.
RUN npm i

# Copy outer function handler
COPY --chown=app:app index.js ./

# COPY function node packages and install, adding this as a separate
# entry allows caching of npm install

WORKDIR /home/app/function
COPY --chown=app:app function/*.json ./

RUN npm i

# COPY function files and folders
COPY --chown=app:app function/ ./

# Run any tests that may be available
RUN npm test

# Set correct permissions to use non root user
WORKDIR /home/app/

ENV cgi_headers="true"
ENV fprocess="node index.js"
ENV mode="http"
ENV upstream_url="http://127.0.0.1:3000"
ENV ready_path="/health"

ENV exec_timeout="60s"
ENV write_timeout="60s"
ENV read_timeout="60s"

ENV prefix_logs="false"

HEALTHCHECK --interval=3s CMD [ -e /tmp/.lock ] || exit 1

CMD ["fwatchdog"]
