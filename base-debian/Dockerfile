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

RUN if [ "$TARGETARCH" = "arm64" ]; then \
    PULUMI_URL="https://get.pulumi.com/releases/sdk/pulumi-${PULUMI_VERSION}-linux-arm64.tar.gz"; \
  else \
    PULUMI_URL="https://get.pulumi.com/releases/sdk/pulumi-${PULUMI_VERSION}-linux-x64.tar.gz"; \
  fi && \
  if command -v wget >/dev/null 2>&1; then \
    wget -q -O /tmp/pulumi.tar.gz "$PULUMI_URL"; \
  elif command -v curl >/dev/null 2>&1; then \
    curl -fsSL -o /tmp/pulumi.tar.gz "$PULUMI_URL"; \
  else \
    echo "Neither wget nor curl is available to download Pulumi" >&2; \
    exit 1; \
  fi && \
  tar -xf /tmp/pulumi.tar.gz && \
  mv pulumi/* /usr/local/bin/ && \
  rm -rf /tmp/pulumi.tar.gz pulumi

USER spacelift
