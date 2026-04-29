# TH07 CPU Universal Image Dockerfile
#
# FIPS-friendly Features:
# - Build tools are isolated in intermediate stage
# - Final image contains only runtime dependencies
# - Uses pip install (additive) to preserve base image packages
#
# Build Modes:
# - Midstream (default): DOWNSTREAM=false - installs system packages
# - Downstream: DOWNSTREAM=true - skips midstream-only sections (base image has them)

################################################################################
# Build Arguments
################################################################################
ARG BASE_IMAGE=quay.io/opendatahub/odh-workbench-jupyter-minimal-cpu-py312-ubi9:3.4-v1.43
ARG PYTHON_VERSION=3.12
ARG DOWNSTREAM=false

################################################################################
# Build Stage - Install Python Dependencies
################################################################################
FROM ${BASE_IMAGE} AS builder

USER 0
WORKDIR /tmp/deps

# Copy requirements files
COPY --chown=1001:0 pyproject.toml requirements.txt ./

# Switch to user 1001 for pip installations
USER 1001
WORKDIR /opt/app-root/src

# Install dependencies from AIPCC GA index
RUN uv pip install --no-cache-dir \
    --index-strategy=unsafe-best-match \
    --index-url=https://console.redhat.com/api/pypi/public-rhai/rhoai/3.4/cpu-ubi9/simple/ \
    -r /tmp/deps/requirements.txt

# Trim site-packages: remove caches, bytecode, tests, and strip .so debug symbols
ARG PYTHON_VERSION
USER 0
RUN find /opt/app-root/lib/python${PYTHON_VERSION}/site-packages \
        \( -type d -name '__pycache__' -o \
           -type d -name 'tests' -o \
           -type d -name 'test' -o \
           -type f -name '*.pyc' -o \
           -type f -name '*.pyo' \) \
        -exec rm -rf {} + 2>/dev/null; \
    find /opt/app-root/lib/python${PYTHON_VERSION}/site-packages \
        -type f -name '*.so' -exec strip --strip-debug {} + 2>/dev/null; \
    rm -f /opt/app-root/bin/uv \
 && chmod -R g+w /opt/app-root/lib/python${PYTHON_VERSION}/site-packages \
 && fix-permissions /opt/app-root -P

# Clean up
RUN rm -rf /tmp/deps

################################################################################
# Final Stage - FIPS-friendly Runtime
################################################################################
FROM ${BASE_IMAGE} AS final

LABEL name="universal:th07-cpu-py312-torch210" \
      summary="TH07 CPU Python 3.12 image with PyTorch 2.10.0" \
      description="Universal CPU image combining minimal Jupyter workbench and runtime ML stack (PyTorch 2.10.0) on UBI9" \
      io.k8s.display-name="TH07 CPU Python 3.12 (Workbench + Runtime)" \
      io.k8s.description="Universal CPU image: Jupyter workbench by default; runtime when command provided."

USER 0
WORKDIR /opt/app-root/src

################################################################################
# MIDSTREAM ONLY: System packages
# Controlled by ARG DOWNSTREAM (default: false)
# - DOWNSTREAM=false (midstream): Installs system packages
# - DOWNSTREAM=true: Skips this section (AIPCC base image has everything pre-configured)
################################################################################
ARG DOWNSTREAM

RUN if [ "${DOWNSTREAM}" != "true" ]; then \
    echo "MIDSTREAM BUILD: Installing system packages..." && \
    dnf install -y --setopt=install_weak_deps=False \
        perl \
        mesa-libGL \
        protobuf \
        skopeo && \
    dnf clean all && rm -rf /var/cache/dnf/*; \
else \
    echo "DOWNSTREAM BUILD: Skipping system packages (provided by base image)"; \
fi
################################################################################
# END MIDSTREAM ONLY
################################################################################

# Copy Python site-packages and CLI entry points from builder stage
# This excludes any build artifacts (FIPS friendly)
ARG PYTHON_VERSION
COPY --from=builder /opt/app-root/lib/python${PYTHON_VERSION}/site-packages /opt/app-root/lib/python${PYTHON_VERSION}/site-packages
COPY --from=builder /opt/app-root/bin /opt/app-root/bin

# Copy license file
COPY LICENSE.md /licenses/cpu-license.md

# Copy entrypoint
COPY --chmod=0755 entrypoint-universal.sh /usr/local/bin/entrypoint-universal.sh

USER 1001
WORKDIR /opt/app-root/src

ENTRYPOINT ["/usr/local/bin/entrypoint-universal.sh"]
CMD ["start-notebook.sh"]
