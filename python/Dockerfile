ARG REPOSITORY_BASE_PATH
ARG RELEASE_VERSION
FROM ${REPOSITORY_BASE_PATH}/runner-pulumi-base-debian:${RELEASE_VERSION}

USER root
RUN apt-get install -y git python3 python3-pip python3-dev python3-venv python3-setuptools python3-wheel && ln -sf python3 /usr/bin/python
COPY requirements.txt /tmp/requirements.txt
RUN python3 -m venv /opt/pulumi-python \
  && /opt/pulumi-python/bin/python -m pip install --upgrade pip setuptools wheel \
  && /opt/pulumi-python/bin/python -m pip install --no-cache-dir -r /tmp/requirements.txt \
  && rm /tmp/requirements.txt
ENV VIRTUAL_ENV="/opt/pulumi-python"
ENV PATH="/opt/pulumi-python/bin:$PATH:./.pulumi/bin"
USER spacelift
