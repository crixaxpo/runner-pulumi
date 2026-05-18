ARG REPOSITORY_BASE_PATH
ARG RELEASE_VERSION
FROM ${REPOSITORY_BASE_PATH}/runner-pulumi-base-debian:${RELEASE_VERSION}

USER root
RUN apt-get update -o Acquire::Retries=5 -o Acquire::http::No-Cache=true -o Acquire::http::Pipeline-Depth=0 \
  && apt-get install -y --no-install-recommends git python3 python3-pip python3-venv \
  && ln -sf python3 /usr/bin/python \
  && rm -rf /var/lib/apt/lists/*
COPY requirements.txt /tmp/requirements.txt
RUN python3 -m venv /opt/pulumi-python \
  && /opt/pulumi-python/bin/python -m pip install --upgrade pip setuptools wheel \
  && /opt/pulumi-python/bin/python -m pip install --no-cache-dir --no-compile -r /tmp/requirements.txt \
  && find /opt/pulumi-python -type d -name "__pycache__" -prune -exec rm -rf {} + \
  && rm /tmp/requirements.txt
ENV VIRTUAL_ENV="/opt/pulumi-python"
ENV PYTHONDONTWRITEBYTECODE="1"
ENV PATH="/opt/pulumi-python/bin:$PATH:./.pulumi/bin"
USER spacelift
