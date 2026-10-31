# k8s

prod의 단일 원본. Kustomize base/overlay, Argo CD app-of-apps, Ingress(web만 공개), NetworkPolicy, ConfigMap, 암호화 Secret.
`kubectl apply`로 직접 적용하지 않는다. 이미지 태그 변경 PR → 머지 → Argo CD 수동 Sync.

## 참고 구조

매니페스트를 처음 작성할 때 [crowfoot-erd/crowfoot-manifests](https://github.com/crowfoot-erd/crowfoot-manifests)의 디렉터리 구조와 작성 방식을 참고한다 (2026-10-31 결정). 그 레포를 기준으로 삼되, 이 레포의 원칙(Kustomize base/overlay, Argo CD 수동 Sync, Ingress는 web만 공개, NetworkPolicy, 암호화 Secret)과 다르면 이 레포의 원칙을 따른다.
