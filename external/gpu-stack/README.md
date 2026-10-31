# external/gpu-stack

GPU PC에서 model-runtime(8001)과 ollama(11434)를 docker compose로 실행한다 (`restart: unless-stopped`).
방화벽에서 8001, 11434를 클러스터 노드 IP에서만 허용한다.
