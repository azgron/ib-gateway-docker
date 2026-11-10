##############################################################################
# Setup Stage: install apps
#
# This is a dedicated stage so that donwload archives don't end up on
# production image and consume unnecessary space.
##############################################################################

# hadolint global ignore=DL3008,DL4006
# Base pinned by digest: the multi-arch index of ubuntu:24.04 (= noble-20261024.1), read 2026-11-10 with
# `docker buildx imagetools inspect ubuntu:24.04`. Bump it on purpose; `apt-get upgrade` in the final stage still
# installs the Ubuntu security updates published after this snapshot.
ARG UBUNTU_IMAGE=ubuntu:24.04@sha256:fe81560990fae86dc4743202b2cb4d3df98d0bcf949173cecef15ce1127ef852
FROM ${UBUNTU_IMAGE} AS setup

ENV IB_GATEWAY_VERSION=10.50.1f
ENV IB_GATEWAY_CHANNEL=stable
ENV IBC_VERSION=3.24.2
ARG DEBIAN_FRONTEND=noninteractive
ARG IB_GATEWAY_REPO="https://github.com/gnzsnz/ib-gateway-docker"
ARG IBC_FILE="IBCLinux-${IBC_VERSION}.zip"
ARG IBC_REPO="https://github.com/IbcAlpha/IBC"
ARG IBC_URL="${IBC_REPO}/releases/download/${IBC_VERSION}/${IBC_FILE}"
# Checksums pinned HERE, not fetched beside the files they check (gnzsnz's .sha256 comes from the same release as the
# installer; IbcAlpha publishes none). A version bump must change these too, or the build fails (by design).
# IBC 3.24.2: GitHub's upload digest of IBCLinux-3.24.2.zip, re-hashed 2026-11-10.
ARG IBC_SHA256="cc097ca1dfa75413a5fbe02e4743050af15f3308eb6a97c769b3cff9850c80c5"
# IB Gateway 10.50.1f: gnzsnz release ibgateway-stable@10.50.1f (assets unchanged since the 2026-10-06 upload).
ARG IB_GATEWAY_SHA256_X64="2b868362238cef71ee0f83068c6e5777ae38f59eae585ff84f75ca1d8b2e06d5"
ARG IB_GATEWAY_SHA256_ARM="f61ff3e175c4f9c3db658274ca41165595fcf9adfbe61834f622fec8b1c6cb2d"

WORKDIR /tmp/setup

# Prepare system
RUN apt-get update -y && \
  apt-get install --no-install-recommends --yes \
  curl ca-certificates unzip && \
  apt-get clean && \
  rm -rf /var/lib/apt/lists/* && \
  if [ "$(uname -m)" = "aarch64" ]; then \
     ARCH="arm" ; \
     IB_GATEWAY_SHA256="${IB_GATEWAY_SHA256_ARM}" ; \
  else \
     ARCH="x64" ; \
     IB_GATEWAY_SHA256="${IB_GATEWAY_SHA256_X64}" ; \
  fi && \
  IB_GATEWAY_FILE="ibgateway-${IB_GATEWAY_VERSION}-standalone-linux-${ARCH}.sh" && \
  IB_GATEWAY_URL="${IB_GATEWAY_REPO}/releases/download/ibgateway-${IB_GATEWAY_CHANNEL}%40${IB_GATEWAY_VERSION}/${IB_GATEWAY_FILE}" && \
# Install IB Gateway
# Use this instead of "RUN curl .." to install a local file:
#COPY ibgateway-${IB_GATEWAY_VERSION}-standalone-linux-x64.sh .
  curl -fsSOL --proto '=https' "${IB_GATEWAY_URL}" && \
  echo "${IB_GATEWAY_SHA256}  ${IB_GATEWAY_FILE}" | sha256sum --check --strict - && \
  chmod a+x "./${IB_GATEWAY_FILE}" && \
  "./${IB_GATEWAY_FILE}" -q -dir "/root/Jts/ibgateway/${IB_GATEWAY_VERSION}" && \
  if [ -d "/root/Jts/ibgateway/${IB_GATEWAY_VERSION}/jre" ]; then \
    echo "/home/ibgateway/Jts/ibgateway/${IB_GATEWAY_VERSION}/jre" > "/root/Jts/ibgateway/${IB_GATEWAY_VERSION}/.install4j/inst_jre.cfg" ; \
  fi && \
  # Install IBC
  curl -fsSOL --proto '=https' "${IBC_URL}" && \
  echo "${IBC_SHA256}  ${IBC_FILE}" | sha256sum --check --strict - && \
  mkdir /root/ibc && \
  unzip "./${IBC_FILE}" -d /root/ibc && \
  chmod -R u+x /root/ibc/*.sh && \
  chmod -R u+x /root/ibc/scripts/*.sh

COPY ./config/ibgateway/jts.ini.tmpl /root/Jts/jts.ini.tmpl
COPY ./config/ibc/config.ini.tmpl /root/ibc/config.ini.tmpl

# Copy scripts
COPY ./scripts /root/scripts

##############################################################################
# Build Stage: build production image
##############################################################################

FROM ${UBUNTU_IMAGE}

ENV IB_GATEWAY_VERSION=10.50.1f
# IB Gateway user constants
ARG USER_ID="${USER_ID:-1000}"
ARG USER_GID="${USER_GID:-1000}"
# IBC env vars
ENV HOME=/home/ibgateway
ENV TWS_MAJOR_VRSN=${IB_GATEWAY_VERSION}
ENV TWS_PATH=${HOME}/Jts
ENV TWS_INI=jts.ini
ENV TWS_INI_TMPL=${TWS_INI}.tmpl
ENV IBC_PATH=${HOME}/ibc
ENV IBC_INI=${HOME}/ibc/config.ini
ENV IBC_INI_TMPL=${IBC_INI}.tmpl
ENV SCRIPT_PATH=${HOME}/scripts
ENV GATEWAY_OR_TWS=gateway
ARG DEBIAN_FRONTEND=noninteractive
# Copy files
COPY --from=setup /usr/local/ /usr/local/
COPY --chown=${USER_ID}:${USER_GID} --from=setup /root/ ${HOME}

# Prepare system
RUN apt-get update -y && \
  apt-get upgrade -y && \
  apt-get install --no-install-recommends --yes \
  gettext-base socat xvfb x11vnc ca-certificates \
  libatk-bridge2.0-0 libatk1.0-0 libatspi2.0-0 libc6 libcairo2 libcups2 libdbus-1-3 \
  libdrm2 libexpat1 libgbm1 libglib2.0-0 libgtk-3-0 libnspr4 libnss3 libpango-1.0-0 \
  libx11-6 libxcb1 libxcomposite1 libxdamage1 libxext6 libxfixes3 libxkbcommon0 \
  libxrandr2 fonts-liberation libasound2t64 libxtst6 xdg-utils && \
  apt-get clean && \
  rm -rf /var/lib/apt/lists/* && \
  if id ubuntu; then \
    userdel -rf ubuntu \
  ;fi && \
  groupadd --gid ${USER_GID} ibgateway && \
  useradd -ms /bin/bash --uid ${USER_ID} --gid ${USER_GID} ibgateway && \
  chmod a+x ${SCRIPT_PATH}/*.sh && \
  mkdir -p /tmp/.X11-unix && chmod 1777 /tmp/.X11-unix

USER ${USER_ID}:${USER_GID}
WORKDIR ${HOME}

# Start run script
CMD ["/home/ibgateway/scripts/run.sh"]

LABEL org.opencontainers.image.source=https://github.com/azgron/ib-gateway-docker
LABEL org.opencontainers.image.url=https://github.com/azgron/ib-gateway-docker
LABEL org.opencontainers.image.description="Docker image with IB Gateway and IBC "
LABEL org.opencontainers.image.licenses="Apache License Version 2.0"
LABEL org.opencontainers.image.version=${IB_GATEWAY_VERSION}-stable
