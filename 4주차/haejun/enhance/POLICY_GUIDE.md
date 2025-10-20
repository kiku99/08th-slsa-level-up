# 4주차 과제 - 정책 기반 배포 고도화

## 고민한 점

기본 예제가 단순히 라벨 체크하는 정도라서, 실제로 배포 환경을 보호하려면 뭐가 더 필요할까 생각해봤다.

**"최소한 어떤 제약이 있어야 할까?"**

크게 두 가지 측면으로 나눠서 생각했다:
1. **필수로 막아야 하는 것** (안 막으면 진짜 위험한 것)
2. **권장사항** (운영 편의성을 위해)

---

## 적용한 정책

### 1. Root 실행 금지 (필수)

컨테이너가 root로 돌면 탈취당했을 때 호스트까지 위험해진다.
최소 권한 원칙의 기본이라고 생각해서 필수로 넣었다.

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
```

### 2. 리소스 제한 필수 (필수)

limits 안 걸면 한 Pod가 노드 전체 리소스 먹어버릴 수 있다.
실제로 이거 때문에 노드 다운된 경험이 있어서 필수라고 생각했다.

```yaml
resources:
  limits:
    memory: "128Mi"
    cpu: "200m"
```

### 3. 호스트 네임스페이스 격리 (권장)

호스트 네트워크나 PID 공유하면 컨테이너 격리가 의미 없어진다.
다른 컨테이너나 호스트 프로세스까지 접근 가능해져서 위험하다.

```yaml
hostNetwork: false
hostPID: false
hostIPC: false
```

### 4. latest 태그 금지 (권장)

latest 쓰면 언제 뭐가 바뀔지 모르고, 문제 생겼을 때 추적도 어렵다.
재현 가능한 배포를 위해 명시적인 버전을 쓰도록 강제했다.

```yaml
# 이건 안됨
image: nginx:latest

# 이렇게 써야 함
image: nginx:1.25
```

---

## 테스트

### 정책 적용
```bash
kubectl apply -f enhanced-security-policies.yaml
kubectl get clusterpolicy
```

### 성공 케이스
모든 정책을 지킨 Pod는 잘 배포됨:
```bash
kubectl apply -f demo-secure.yaml
kubectl get pod secure-demo-app
```

### 실패 케이스
정책 위반하면 차단됨:
```bash
kubectl apply -f demo-fail-cases.yaml
# 각각 해당하는 정책에서 막힘
```

---

## 정리

| 정책 | 우선순위 | 이유 |
|------|---------|------|
| Root 금지 | 필수 | 호스트 침해 방지 |
| 리소스 제한 | 필수 | 노드 안정성 |
| 호스트 격리 | 권장 | 컨테이너 격리 |
| latest 금지 | 권장 | 재현 가능성 |

이 정도면 기본적인 보안은 확보하면서도 너무 복잡하지 않게 관리할 수 있을 것 같다.

실전에서는 audit 모드로 먼저 켜서 영향도 파악하고, 단계적으로 enforce로 전환하는 게 좋을 것 같다.
