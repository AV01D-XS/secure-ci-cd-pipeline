# ---- Build stage ----
FROM python:3.11-slim AS builder

# set nonroot user variables (uid/gid chosen large to avoid conflicts)
ARG APP_USER=appuser
ARG APP_UID=10001
ARG APP_GID=10001

ENV VENV_PATH=/opt/venv

# install build deps
RUN apt-get update && \
    apt-get install -y --no-install-recommends build-essential gcc git curl && \
    python -m venv ${VENV_PATH} && \
    ${VENV_PATH}/bin/pip install --upgrade pip setuptools wheel && \
    rm -rf /var/lib/apt/lists/*

# create non-root user
RUN groupadd -g ${APP_GID} ${APP_USER} && \
    useradd -m -u ${APP_UID} -g ${APP_GID} -s /bin/bash ${APP_USER}

WORKDIR /src

# copy only requirements first for better caching
COPY src/app/requirements*.txt ./
COPY src/app/requirements-dev.txt ./
RUN ${VENV_PATH}/bin/pip install --no-cache-dir -r requirements.txt

# copy application and tests
COPY src/app/ ./
COPY tests/ ./tests/

# ---- Test stage ----
FROM builder AS test
# Install test dependencies
RUN ${VENV_PATH}/bin/pip install --no-cache-dir -r requirements-dev.txt

# ---- Runtime stage ----
FROM python:3.11-slim AS runtime

ARG APP_USER=appuser
ARG APP_UID=10001
ARG APP_GID=10001
ENV VENV_PATH=/opt/venv
ENV PATH="${VENV_PATH}/bin:$PATH"
ENV PYTHONUNBUFFERED=1

# minimal runtime deps including curl for healthcheck
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates curl && rm -rf /var/lib/apt/lists/*

# create non-root user and group
RUN groupadd -g ${APP_GID} ${APP_USER} && \
    useradd -m -u ${APP_UID} -g ${APP_GID} -s /bin/bash ${APP_USER}

# copy virtualenv from builder
COPY --from=builder ${VENV_PATH} ${VENV_PATH}
# copy code (excluding tests in production)
COPY --from=builder /src /app

WORKDIR /app
RUN chown -R ${APP_USER}:${APP_USER} /app ${VENV_PATH}

USER ${APP_USER}

# runtime port (documented)
EXPOSE 8080

# Healthcheck with curl (now available)
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/healthz || exit 1

# start command - adapt to your app
CMD ["python", "main.py"]
