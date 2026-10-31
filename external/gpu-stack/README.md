# external/gpu-stack

GPU PC에서 model-runtime(GPU 워커)과 ollama를 docker compose로 실행한다 (`restart: unless-stopped`).

- 워커가 conversation-service의 작업을 가져간다(pull). 클러스터는 GPU PC를 부르지 않으므로 **들어오는 포트를 열지 않는다.**
- 워커 설정: `GPU_WORKER_API_URL`(게이트웨이 주소), `GPU_WORKER_TOKEN`(클러스터 `api-gateway-properties`와 같은 값), `OLLAMA_URL=http://ollama:11434`(compose 내부)
- ollama 포트도 compose 밖으로 내지 않는다.
- 자세한 흐름: docs 레포 `docs/architecture.md` 8절 "GPU 작업 가져가기"
