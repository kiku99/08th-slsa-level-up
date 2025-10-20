# 정책 기반 배포 고도화 실습
> SLSA-Study 4주차 과제

---

## 1. 정책 고도화

> demo에서의 라벨 검증 외의 배포 시 주의할 점에 대해 고민  
  기준 : [ 보안, 무결성, 자원 관리 ]

### 1-1. 보안 (Security)
> 컨테이너에 root, privileged 등 높은 권한 부여 시,  
  호스트에 대한 권한도 획득 가능

### 1-2. 무결성 (Integrity)
> 취약한 레지스트리 차단 가능,  
  latest 태그는 계속해서 갱신되기 때문에 안정성 보장 불가

### 1-3. 자원 관리 (Resource)
> 자원의 요청/제한을 걸어두지 않을 시,  
  엄격한 자원 관리가 힘들어 클러스터 안정성 보장에 어려움

### 1-4. 정책 설정 요약

| 구분 | 목표 | 핵심 내용 |
|------|------|-----------|
| **보안(Security)** | 루트 권한 및 특권 모드 금지 | `runAsNonRoot: true`, `privileged: false` |
| **무결성(Integrity)** | 허용된 레지스트리 사용, `latest` 태그 금지 | `ghcr.io`, `index.docker.io`만 허용 |
| **자원 관리(Resource)** | CPU/메모리 요청 및 제한 필수 | `requests`와 `limits` 모두 지정 |

---

## 2. 정책 적용 및 테스트

```bash
# 정책 적용
kubectl apply -f secure-deploy-policies.yaml
kubectl get clusterPolicy secure-deploy-policies
kubectl describe clusterPolicy secure-deploy-policies

# 테스트
kubectl apply -f secure-demo.yaml
kubectl get po secure-demo-app
kubectl describe clusterPolicy secure-demo-app
```

### 2-1. 정책 적용 네임스페이스 설정
> 위에 정책 적용 후 describe시 기존 pod에서 위반 탐지 경고 발생

#### > 적용 네임스페이스 설정
```yaml
match:
  resources:
    kinds:
      - Pod
    namespaces:
      - default # default 네임스페이스만 검사
```

#### > 특정 네임스페이스 제외 설정
```yaml
match:
  resources:
    kinds:
      - Pod
exclude:
  resources:
    namespaces:
      - kube-system
      - kyverno
```