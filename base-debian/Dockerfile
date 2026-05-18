ARG BASE_IMAGE=debian:latest
FROM $BASE_IMAGE
ARG PULUMI_VERSION
ARG TARGETARCH
ARG SKIP_APT_INSTALL=false

USER root
WORKDIR /home/spacelift
RUN if [ "$SKIP_APT_INSTALL" != "true" ]; then \
    apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get update -o Acquire::Retries=5 -o Acquire::http::No-Cache=true -o Acquire::http::Pipeline-Depth=0 \
    && apt-get install -y --no-install-recommends adduser wget procps ca-certificates \
    && rm -rf /var/lib/apt/lists/*; \
  fi

RUN echo "hosts: files dns" > /etc/nsswitch.conf \
  && if command -v adduser >/dev/null 2>&1; then \
    adduser --disabled-password --gecos "" --uid=1983 spacelift; \
  elif command -v useradd >/dev/null 2>&1; then \
    useradd --create-home --uid=1983 --shell /bin/sh spacelift; \
  else \
    echo "spacelift:x:1983:1983:spacelift:/home/spacelift:/bin/sh" >> /etc/passwd \
    && echo "spacelift:x:1983:" >> /etc/group \
    && mkdir -p /home/spacelift; \
  fi \
  && chown -R 1983:1983 /home/spacelift

ADD https://get.pulumi.com/releases/sdk/pulumi-${PULUMI_VERSION}-linux-x64.tar.gz /tmp/pulumi-linux-amd64.tar.gz
ADD https://get.pulumi.com/releases/sdk/pulumi-${PULUMI_VERSION}-linux-arm64.tar.gz /tmp/pulumi-linux-arm64.tar.gz
RUN if [ "$TARGETARCH" = "arm64" ]; then \
    tar -xf /tmp/pulumi-linux-arm64.tar.gz; \
  else \
    tar -xf /tmp/pulumi-linux-amd64.tar.gz; \
  fi && \
  mv pulumi/* /usr/local/bin/ && \
  rm -rf /tmp/pulumi-linux-*.tar.gz pulumi

USER spacelift
