# deploy

배포 원본. `local/`(compose + 가짜 런타임, `.env.example`), `k8s/`(Kustomize, Argo CD Application, Ingress, NetworkPolicy, 암호화 시크릿), `external/gpu-stack/`(GPU PC compose). prod는 이 레포가 단일 원본이고 `kubectl apply`로 직접 적용하지 않는다.

- 기술: docker compose, Kustomize, Argo CD, cert-manager, Sealed Secrets 또는 SOPS(D9).
- 설계와 스펙: [docs](https://github.com/4cut-story/docs)
