# MeshCore repeater for the Bobcat Miner 300, bridging to the host's already-
# running lora_pkt_fwd_1302 process via the Semtech UDP packet-forwarder
# protocol (see ../Research_beginning_notes.md and meshcore/variants/linux_udp_bridge/).
#
# Two-stage build: debian:bookworm-slim for both stages (glibc + libsodium-dev
# availability over Alpine's smaller size - reproducibility for other Bobcat
# 300 owners building this themselves matters more here than image size).
#
# Default workflow is building THIS Dockerfile directly on the device
# (already has root SSH + Docker) - no cross-compilation toolchain to set up:
#   docker build -t meshcore-repeater -f meshcore-deploy/Dockerfile .
# From an x86 dev machine instead, use buildx with QEMU emulation:
#   docker buildx build --platform linux/arm64 -t meshcore-repeater -f meshcore-deploy/Dockerfile .

FROM debian:bookworm-slim AS build

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential cmake pkg-config libsodium-dev ca-certificates \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /src
COPY meshcore/ meshcore/

RUN cmake -B /build -S meshcore/variants/linux_udp_bridge -DCMAKE_BUILD_TYPE=Release \
    && cmake --build /build -j"$(nproc)"

FROM debian:bookworm-slim AS runtime

RUN apt-get update && apt-get install -y --no-install-recommends \
    libsodium23 \
    && rm -rf /var/lib/apt/lists/*

COPY --from=build /build/meshcore_repeater /usr/local/bin/meshcore_repeater

# Bind-mounted at runtime to a real, persistent directory on the host eMMC
# (e.g. /update/meshcore-data) - see docker-compose.yml. Created here so the
# container doesn't fail if the mount is momentarily absent.
RUN mkdir -p /data
ENV LINUX_MESHCORE_DATA_DIR=/data

ENTRYPOINT ["/usr/local/bin/meshcore_repeater"]
