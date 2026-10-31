# k8s

prod 배포의 단일 원본. Argo CD **ApplicationSet**이 `apps/*` 디렉터리마다 Application을 만들고 **자동 동기화**(prune + selfHeal)한다. `kubectl apply`로 앱을 직접 바꾸지 않는다.

## 구조

```
argocd/applicationset.yaml   # 클러스터에 한 번만 적용. apps/* 를 스캔해 앱별 Application 생성
apps/
  api-gateway/               # 8080
  auth-service/              # 8080
  conversation-service/      # 8080
  content-service/           # 8080
  web/                       # 3000 (외부 공개는 이것만)
  external/                  # GPU PC의 model-runtime(8001), ollama(11434)를 Service + EndpointSlice로 등록
  network-policies/          # 들어오는 트래픽 기본 차단 + 허용 방향
```

서비스 디렉터리는 `kustomization.yaml`(이미지 태그, ConfigMap) + `deployment.yaml` + `service.yaml`. 이름·포트는 `common/service-catalog.yaml`과 같다. 네임스페이스는 `fourcut`.

## 배포 흐름

1. 서비스 레포 main에 머지 → CI(common 재사용 워크플로)가 테스트·정적 분석 → GHCR `:<sha>` 이미지 푸시
2. 같은 CI가 이 레포 `k8s/apps/<서비스>/kustomization.yaml`의 `images[].newTag`를 sha로 커밋
3. ApplicationSet이 변경을 감지해 자동 동기화
4. 롤백: 이 레포에서 해당 커밋을 되돌린다

## 처음 한 번 할 일

1. `kubectl -n argocd apply -f k8s/argocd/applicationset.yaml`
2. 네임스페이스 `fourcut`에 시크릿 생성 (레포에 넣지 않는다. Rancher에서 만들거나 Sealed Secrets/SOPS로 관리, D9)
   - `ghcr-pull` (docker-registry, read:packages PAT): GHCR 패키지가 비공개일 때만 필요
   - `<서비스>-properties` (Opaque): `auth-service`, `conversation-service`, `content-service`, `web`
     - DB: `DB_URL`, `DB_USERNAME`, `DB_PASSWORD`, `DB_SCHEMA`
     - MinIO(conversation, content): `MINIO_ENDPOINT`, `MINIO_REGION`, `MINIO_ACCESS_KEY`, `MINIO_SECRET_KEY`, `MINIO_BUCKET`
     - OAuth·JWT(auth), 세션 키(web)는 각 서비스 `application-prod.yml` 기준
3. `apps/external/gpu-pc.yaml`의 IP를 GPU PC 고정 IP로 바꾼다
4. web Ingress(cert-manager, Let's Encrypt)는 도메인이 정해지면 `apps/web/`에 추가한다

## crowfoot-manifests에서 바꾼 점

[crowfoot-erd/crowfoot-manifests](https://github.com/crowfoot-erd/crowfoot-manifests) 구조(ApplicationSet + `apps/*`, 자동 동기화, 서비스별 properties Secret)를 따르고 아래만 바꿨다.
- **이미지 태그를 kustomization `images`에서 관리**: 이 레포의 `.github/workflows/sync-images.yml`이 10분마다 각 서비스의 마지막 성공 이미지를 찾아 `kustomize edit set image`로 바꿔 커밋한다(수동 실행도 가능). 서비스 CI가 이 레포에 쓰지 않으므로 개인 토큰이 필요 없다. deployment.yaml을 `sed`로 고치지 않는다.
- **`latest` 태그 없음**: sha 태그만 쓴다.
- **ConfigMap은 configMapGenerator**: 설정이 바뀌면 이름 해시가 바뀌어 자동으로 롤링 재시작된다.
- **probe를 liveness/readiness로 분리**: `/actuator/health/liveness`, `/actuator/health/readiness`. DB가 잠깐 끊겨도 재시작되지 않고 트래픽만 빠진다.
- **보안 기본값**: `runAsNonRoot`, 권한 상승 금지, capability 전부 제거, seccomp RuntimeDefault.
- **NetworkPolicy**: 호출 방향만 허용한다 (설계 원칙 3).
- **GPU PC를 Service + EndpointSlice로 등록**: 호출하는 쪽이 실제 위치를 몰라도 된다.
- **replicas 1부터**: 규모에 맞게 시작하고 필요하면 늘린다.
