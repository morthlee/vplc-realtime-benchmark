# Embedded-friendly Dockerfile for vPLC Real-time Benchmark
# Optimized for resource-constrained platforms (RISC-V, ARM64)

FROM debian:bookworm-slim

LABEL maintainer="vPLC Benchmark Authors"
LABEL description="Lightweight container for real-time latency measurement on embedded platforms"

ENV WS_DIR="/benchmark"
WORKDIR ${WS_DIR}

SHELL ["/bin/bash", "-c"]

ARG DEBIAN_FRONTEND=noninteractive

# Minimal dependencies for latency measurement
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    ca-certificates \
    curl \
    gnuplot-nox \
    libnuma-dev \
    make \
    python3-minimal \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

# Install rt-tests (cyclictest, hwlatdetect, etc.)
RUN apt-get update && apt-get install -y --no-install-recommends \
    rt-tests \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

# Download and configure mklatencyplot script
RUN curl -SLO --fail https://www.osadl.org/uploads/media/mklatencyplot.bash \
    && sed -i -E "s/(cores=).*/\1\$(nproc)/" mklatencyplot.bash \
    && sed -i 's/set terminal png/set terminal pngcairo size 800,600/' mklatencyplot.bash \
    && chmod +x mklatencyplot.bash

# Create output directory
RUN mkdir -p ${WS_DIR}/results

# Default entrypoint for interactive use
ARG DEBIAN_FRONTEND=dialog
CMD ["/bin/bash"]