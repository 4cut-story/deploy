# k8s

prod 배포의 단일 원본. Argo CD **ApplicationSet**이 `apps/*` 디렉터리마다 Application을 만들고 **자동 동기화**(prune + selfHeal)한다. `kubectl apply`로 앱을 직접 바꾸지 않는다.

## 구조

```
bootstrap/root-app.yaml      # 루트 Application. UI나 kubectl로 한 번만 만든다. argocd/ 를 동기화
argocd/applicationset.yaml   # apps/* 를 스캔해 앱별 Application 생성 (루트 앱이 관리)
apps/
  api-gateway/               # 8080
  auth-service/              # 8080
  conversation-service/      # 8080
  content-service/           # 8080
  web/                       # 3000 (외부 공개는 이것만)
  network-policies/          # 들어오는 트래픽 기본 차단 + 허용 방향
```

서비스 디렉터리는 `kustomization.yaml`(이미지 태그, ConfigMap) + `deployment.yaml` + `service.yaml`. 이름·포트는 `common/service-catalog.yaml`과 같다. 네임스페이스는 `4cut` (ApplicationSet이 정한다. 매니페스트에는 namespace를 적지 않는다).

## 배포 흐름

1. 서비스 레포 main에 머지 → CI(common 재사용 워크플로)가 테스트·정적 분석 → GHCR `:<sha>` 이미지 푸시
2. 같은 CI의 `notify-deploy` 잡이 이 레포 `sync-images.yml`을 바로 실행 → `k8s/apps/<서비스>/kustomization.yaml`의 `images[].newTag`를 sha로 커밋 (토큰이 없거나 실패하면 10분 폴링이 대신 반영)
3. ApplicationSet이 변경을 감지해 자동 동기화 (Argo CD 기본 git 확인 주기는 3분. 아래 5번으로 줄인다)
4. 롤백: 이 레포에서 해당 커밋을 되돌린다

## 처음 한 번 할 일

1. ApplicationSet 만들기. Argo CD에 레포를 등록하는 것만으로는 아무 앱도 생기지 않는다(레포 등록 = 접속 정보만 저장). 루트 Application 하나를 만들면 그것이 `k8s/argocd/`의 ApplicationSet을 만들고, ApplicationSet이 `apps/*`마다 앱을 만든다(app-of-apps).
   - UI: **+ NEW APP** → Application Name `fourcut-root`, Project `default`, Sync Policy `Automatic`(PRUNE, SELF HEAL 체크) → Repository URL `https://github.com/4cut-story/deploy.git`, Revision `main`, Path `k8s/argocd` → Cluster URL `https://kubernetes.default.svc`, Namespace `argocd` → **CREATE**
     - 여기 Namespace `argocd`는 루트 앱이 ApplicationSet을 만드는 곳이다. ApplicationSet·Application 객체는 Argo CD가 읽는 `argocd` 네임스페이스에 있어야 한다(apps-in-any-namespace를 켜지 않은 기본 설정). 서비스 pod·Service는 ApplicationSet의 destination인 `4cut`에 만들어지고, 없으면 자동 생성된다(CreateNamespace).
   - 또는 CLI: `kubectl -n argocd apply -f k8s/bootstrap/root-app.yaml` (같은 내용)
   - 결과: `fourcut-root` 아래 ApplicationSet `fourcut`, 그 아래 `fourcut-api-gateway` 등 앱 6개. 이미지가 아직 없는 서비스는 replica 0으로 동기화되고, 첫 이미지가 나오면 sync-images가 1로 올린다.
2. 네임스페이스 `4cut`에 시크릿 생성 (레포에 넣지 않는다. Rancher에서 만들거나 Sealed Secrets/SOPS로 관리, D9)
   - `ghcr-pull` (docker-registry, read:packages PAT): GHCR 패키지가 비공개일 때만 필요
   - `<서비스>-properties` (Opaque): `api-gateway`(GPU_WORKER_TOKEN), `auth-service`, `conversation-service`, `content-service`, `web`
     - DB: `DB_URL`, `DB_USERNAME`, `DB_PASSWORD`, `DB_SCHEMA`
     - MinIO(conversation, content): `MINIO_ENDPOINT`, `MINIO_REGION`, `MINIO_ACCESS_KEY`, `MINIO_SECRET_KEY`, `MINIO_BUCKET`
     - OAuth·JWT(auth), 세션 키(web)는 각 서비스 `application-prod.yml` 기준
3. GPU PC는 클러스터에 등록하지 않는다. GPU PC 워커가 게이트웨이의 워커 경로(`/internal/gpu-jobs/**`)로 작업을 가져간다(docs 아키텍처 8절). Ingress(`apps/web/ingress.yaml`)는 게이트웨이의 이 경로만 연다. GPU PC 워커의 `GPU_WORKER_API_URL`은 `https://4cut.java21.net` (HTTPS를 붙인 뒤. 토큰이 평문으로 다니지 않게 한다).
4. Ingress는 `apps/web/ingress.yaml` (호스트 `4cut.java21.net`, `/` → web, `/internal/gpu-jobs` → api-gateway). Argo CD가 관리한다. HTTPS(cert-manager)는 정해지면 추가한다
5. 배포 지연 줄이기 (둘 중 하나)
   - Argo CD가 외부에서 접근되면: 이 레포 Settings → Webhooks에 `https://<argocd 주소>/api/webhook` (push 이벤트) 추가. 커밋 즉시 동기화된다.
   - 접근되지 않으면: `argocd-cm`에 `timeout.reconciliation: 60s`를 넣고 `argocd-repo-server`·`argocd-application-controller`를 재시작한다. 최대 1분.

## crowfoot-manifests에서 바꾼 점

[crowfoot-erd/crowfoot-manifests](https://github.com/crowfoot-erd/crowfoot-manifests) 구조(ApplicationSet + `apps/*`, 자동 동기화, 서비스별 properties Secret)를 따르고 아래만 바꿨다.
- **이미지 태그를 kustomization `images`에서 관리**: 이 레포의 `.github/workflows/sync-images.yml`이 `kustomize edit set image`로 바꿔 커밋한다. 서비스 CI가 이미지를 올린 직후 이 워크플로를 바로 실행하고(토큰은 이 워크플로 실행 권한만), 10분 폴링이 백업으로 남는다. 커밋은 이 레포 `GITHUB_TOKEN`으로 한다. deployment.yaml을 `sed`로 고치지 않는다.
- **`latest` 태그 없음**: sha 태그만 쓴다.
- **ConfigMap은 configMapGenerator**: 설정이 바뀌면 이름 해시가 바뀌어 자동으로 롤링 재시작된다.
- **probe를 liveness/readiness로 분리**: `/actuator/health/liveness`, `/actuator/health/readiness`. DB가 잠깐 끊겨도 재시작되지 않고 트래픽만 빠진다.
- **보안 기본값**: `runAsNonRoot`, 권한 상승 금지, capability 전부 제거, seccomp RuntimeDefault.
- **NetworkPolicy**: 호출 방향만 허용한다 (설계 원칙 3).
- **GPU PC는 pull 방식**: 클러스터가 GPU PC를 부르지 않으므로 GPU PC 주소·방화벽 설정이 필요 없다.
- **replicas 1부터**: 규모에 맞게 시작하고 필요하면 늘린다.
