ARG BUILD_FROM
FROM ${BUILD_FROM}

# Set shell
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# ntfy version to install (overridden by build.yaml args)
ARG NTFY_VERSION=v2.26.0
# Provided by the Home Assistant add-on builder
ARG BUILD_ARCH
ARG BUILD_VERSION

# -----------------------------------------------------------------------------
# Download, checksum-verify and install the official ntfy release binary.
# The tarballs are the statically-linked Go binaries from the GitHub release,
# which run fine on the Alpine (musl) base image.
# SHA256 sums are for the v2.26.0 linux tarballs (github.com/binwiederhier/ntfy).
# -----------------------------------------------------------------------------
RUN apk add --no-cache --virtual .ntfy-build-deps curl \
    && set -eux \
    && case "${BUILD_ARCH}" in \
        aarch64) NTFY_ARCH="arm64"; NTFY_SHA256="3f5b62454eddd5d992641b711fc56c4244564daa0c9dfbbcb0d2db0792b74d19" ;; \
        amd64)   NTFY_ARCH="amd64"; NTFY_SHA256="0a1c4ed14e96e4e4ae8773151ff032d276932f8f193cf37c7a3d82b0ab595673" ;; \
        armv7)   NTFY_ARCH="armv7"; NTFY_SHA256="918cd74ebdc279a9d4df51a9e7ece89ef01d2dafc9aec9667590327d15e6bcb7" ;; \
        *) echo "Unsupported architecture: ${BUILD_ARCH}" && exit 1 ;; \
    esac \
    && NTFY_PLAIN_VERSION="${NTFY_VERSION#v}" \
    && NTFY_DIR="ntfy_${NTFY_PLAIN_VERSION}_linux_${NTFY_ARCH}" \
    && NTFY_TARBALL="${NTFY_DIR}.tar.gz" \
    && NTFY_URL="https://github.com/binwiederhier/ntfy/releases/download/${NTFY_VERSION}/${NTFY_TARBALL}" \
    && curl -sSL -o /tmp/ntfy.tar.gz "${NTFY_URL}" \
    && echo "${NTFY_SHA256}  /tmp/ntfy.tar.gz" | sha256sum -c - \
    && tar -xzf /tmp/ntfy.tar.gz -C /tmp "${NTFY_DIR}/ntfy" \
    && mv "/tmp/${NTFY_DIR}/ntfy" /usr/bin/ntfy \
    && chmod +x /usr/bin/ntfy \
    && rm -rf /tmp/ntfy.tar.gz "/tmp/${NTFY_DIR}" \
    && apk del --no-cache --purge .ntfy-build-deps

# Copy the add-on root filesystem (s6 services, etc.)
COPY rootfs /

# Persistent data directory and executable service scripts
RUN mkdir -p /data \
    && chmod a+x \
        /etc/s6-overlay/s6-rc.d/init-ntfy/run \
        /etc/s6-overlay/s6-rc.d/ntfy/run \
        /etc/s6-overlay/s6-rc.d/ntfy/finish

# ntfy HTTP API / web UI
EXPOSE 80

# Standard Home Assistant add-on labels
LABEL \
    io.hass.name="ntfy" \
    io.hass.description="Self-hosted ntfy push notification server" \
    io.hass.type="addon" \
    io.hass.version="${BUILD_VERSION}" \
    io.hass.arch="aarch64|amd64|armv7" \
    maintainer="PineappleEmperor <pineapple.hass@gmail.com>"
