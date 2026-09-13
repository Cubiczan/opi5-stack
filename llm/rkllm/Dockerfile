# RKLLM server image for RK3588 (aarch64).
# Builds on Ubuntu 22.04 with the rkllm runtime + a small FastAPI shim
# that exposes an OpenAI-compatible /v1/chat/completions endpoint.
FROM arm64v8/ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
        python3 python3-pip python3-venv git curl ca-certificates \
        libgomp1 libstdc++6 \
    && rm -rf /var/lib/apt/lists/*

# --- rkllm runtime ---------------------------------------------------------
# Pull librkllmrt.so + the Python bindings from airockchip's rknn-llm repo.
# Pin a release tag in production builds.
RUN git clone --depth 1 https://github.com/airockchip/rknn-llm.git /opt/rknn-llm \
    && install -m 0755 /opt/rknn-llm/rkllm-runtime/Linux/librkllm_api/aarch64/librkllmrt.so \
        /usr/local/lib/librkllmrt.so \
    && ldconfig

ENV PYTHONPATH=/opt/rknn-llm/rkllm-toolkit/examples/rkllm_server

# --- API shim --------------------------------------------------------------
WORKDIR /app
RUN pip3 install --no-cache-dir fastapi==0.115.* uvicorn[standard]==0.30.* \
                                pydantic==2.* sse-starlette==2.* \
                                "cubiczan-resilience[fastapi] @ git+https://github.com/icohangar-ops/cubiczan-resilience.git#subdirectory=python"

COPY server.py /app/server.py
EXPOSE 8080

# Drop to a non-root user before the server starts. Port 8080 is
# unprivileged; /app is the only path the process must own. The
# `video` group matches typical /dev/rknpu ownership on RK3588.
RUN groupadd --gid 10001 rkllm \
    && useradd --uid 10001 --gid 10001 --groups video \
        --home-dir /app --shell /usr/sbin/nologin rkllm \
    && chown rkllm:rkllm /app
USER rkllm

CMD ["uvicorn", "server:app", "--host", "0.0.0.0", "--port", "8080"]
