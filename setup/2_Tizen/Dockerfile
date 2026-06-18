# ============================================================================
# Tizen RPi4 OS Builder — Serverless Container
# ============================================================================
# Build image: Downloads Tizen source, modifies sdbd, builds RPi4 image
# Input: Custom sdbd handlers (mounted or injected)
# Output: Flashable Tizen RPi4 image (pushed to artifact repository)
# ============================================================================

FROM ubuntu:22.04 AS builder

ENV DEBIAN_FRONTEND=noninteractive \
 TIZEN_BRANCH=tizen \
 TIZEN_MANIFEST=unified_standard.xml \
 TIZEN_VERSION=10.0

# --- Install build dependencies ---
RUN echo "deb [trusted=yes] http://download.tizen.org/tools/latest-release/Ubuntu_22.04/ /" \
      > /etc/apt/sources.list.d/tizen.list \
 && apt-get update && apt-get install -y --no-install-recommends \
      git curl wget python3 python3-pip \
      gbs mic createrepo-c \
      build-essential gcc-aarch64-linux-gnu \
      libssl-dev libffi-dev \
      sudo kmod openssh-client \
 && rm -rf /var/lib/apt/lists/*

# --- Setup build user (GBS doesn't like root) ---
RUN useradd -m -u 1000 builder && \
 mkdir -p /home/builder/GBS-ROOT && \
 chown -R builder:builder /home/builder

# --- Working directories ---
RUN mkdir -p /workspace/source /workspace/sdbd-patch /workspace/output && \
 chown -R builder:builder /workspace

WORKDIR /workspace
USER builder

# --- Step 1: Clone full Tizen source ---
RUN echo "Cloning Tizen source (this may take 10-30 minutes)..." && \
 mkdir -p source && cd source && \
 repo init -u https://git.tizen.org/cgit/scm/manifest \
   -b ${TIZEN_BRANCH} -m ${TIZEN_MANIFEST} && \
 repo sync -j4

# --- Step 2: Apply sdbd modifications ---
COPY --chown=builder:builder sdbd-patch/ /workspace/sdbd-patch/
RUN cd /workspace/source && \
 if [ -f /workspace/sdbd-patch/sdbd.patch ]; then \
   patch -p1 < /workspace/sdbd-patch/sdbd.patch; \
 elif [ -d /workspace/sdbd-patch/handlers ]; then \
   cp -v /workspace/sdbd-patch/handlers/* platform/core/system/sdb/src/sdbd/; \
 fi

# --- Step 3: Build sdbd with GBS ---
RUN cd /workspace/source/platform/core/system/sdb && \
 gbs build -A aarch64 --profile=profile.xml 2>&1 | tee /tmp/build.log

# --- Step 4: Build full Tizen RPi4 image with MIC ---
RUN cd /workspace/source && \
 mic create fs \
   --arch=aarch64 \
   --image-name=tizen-rpi4-custom \
   --tmpfs=0 \
   profiles/unified/standard/standard-rpi4-image.xml

# --- Step 5: Package and output ---
RUN mkdir -p /workspace/output && \
 if [ -d /tmp/tizen-rpi4-custom ]; then \
   tar -czf /workspace/output/tizen-rpi4-custom-$(date +%Y%m%d-%H%M%S).tar.gz \
     -C /tmp tizen-rpi4-custom; \
 fi

# --- Final: Lightweight output stage ---
FROM ubuntu:22.04 AS output

RUN apt-get update && apt-get install -y --no-install-recommends curl  && \
    rm -rf /var/lib/apt/lists/*

COPY --from=builder /workspace/output /output

ENTRYPOINT ["/bin/bash"]
CMD ["-c", "echo 'Tizen RPi4 image ready in /output'; ls -lh /output/"]