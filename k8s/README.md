# k8s

prod의 단일 원본. Kustomize base/overlay, Argo CD app-of-apps, Ingress(web만 공개), NetworkPolicy, ConfigMap, 암호화 Secret.
`kubectl apply`로 직접 적용하지 않는다. 이미지 태그 변경 PR → 머지 → Argo CD 수동 Sync.
