# deploy

**배포의 원본이에요. 로컬 compose, 쿠버네티스 매니페스트(Argo CD), GPU PC compose.**

[← 4cut story 소개](https://github.com/4cut-story) · [아키텍처](https://github.com/4cut-story/.github/blob/main/docs/architecture.md) · [하드웨어](https://github.com/4cut-story/.github/blob/main/docs/hardware.md)

## 하는 일

| 경로 | 내용 |
| --- | --- |
| `local/` | 개발용 실행 환경의 환경 변수 예시(`.env.example`) |
| `k8s/` | 운영 배포. Kustomize, Argo CD ApplicationSet, Ingress, NetworkPolicy |
| `external/gpu-stack/` | GPU PC에서 model-runtime과 Ollama를 띄우는 compose |
| `.github/workflows/sync-images.yml` | 서비스의 새 이미지 태그를 매니페스트에 반영 |

운영 배포는 이 저장소가 단일 원본이에요. `kubectl apply`로 직접 바꾸지 않아요.

## 시스템 안에서의 위치

```
서비스 main 머지 → CI가 GHCR에 :<sha> 이미지 → sync-images가 k8s/apps/<서비스> 태그 커밋 → Argo CD 자동 동기화
```

롤백은 이 저장소에서 해당 커밋을 되돌리면 돼요. `latest` 태그는 쓰지 않아요.

## 실행

클러스터에 처음 올릴 때는 루트 Application 하나만 만들면 나머지는 Argo CD가 만들어요. 순서는 [k8s 문서](k8s/README.md#처음-한-번-할-일)에 있어요.

## 자세한 문서

| 문서 | 내용 |
| --- | --- |
| [k8s](k8s/README.md) | 구조, 배포 흐름, 처음 한 번 할 일, 시크릿 |
| [local](local/README.md) | 개발용 실행 환경 |
| [gpu-stack](external/gpu-stack/README.md) | GPU PC compose와 워커 설정 |
| [아키텍처](https://github.com/4cut-story/.github/blob/main/docs/architecture.md) | 배포·로그인·GPU 작업 흐름 |
| [하드웨어](https://github.com/4cut-story/.github/blob/main/docs/hardware.md) | GPU PC와 쿠버네티스 클러스터 구성 |
