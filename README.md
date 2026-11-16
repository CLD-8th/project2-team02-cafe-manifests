# project2-team02-cafe-manifests

cafe 앱과 플랫폼 구성 요소의 쿠버네티스 배포 설정. **클러스터 안에 올리는 것은 전부 이 저장소 → Argo CD** 로만 바꿈.

```
bootstrap/root.yaml          app-of-apps — 손으로 apply 하는 유일한 파일
clusters/dev/                Argo CD Application · AppProject
  project-apps.yaml          wave -3  AppProject apps (cafe Namespace 만)
  alb-controller.yaml        wave -2  Helm aws-load-balancer-controller 3.5.0
  external-secrets.yaml      wave -2  Helm external-secrets 2.13.0
  secret-store.yaml          wave -1  platform/eks/external-secrets
  storage.yaml               wave -1  platform/eks/storage
  cafe.yaml                  wave  0  apps/cafe/overlays/eks
platform/eks/                ALB Controller 값 · ClusterSecretStore · gp3 StorageClass
apps/cafe/base               Deployment · Service · ConfigMap
apps/cafe/overlays/eks       Namespace · SA · ExternalSecret · Ingress · HPA · PDB · RBAC · 이미지 태그
```

## 01. 담당

| 담당 | 폴더 |
| --- | --- |
| A. Argo CD | `bootstrap/` · `clusters/dev/` |
| B. 플랫폼 | `platform/eks/` · `apps/cafe/overlays/eks/externalsecret.yaml` |
| C. 앱 매니페스트 | `apps/cafe/` |
| D. 앱 CI | 앱 저장소 `.github/workflows/build-and-update.yml` → `overlays/eks` 의 `newTag` |

## 02. 처음 한 번 — Argo CD 설치 (관리용 EC2 · Session Manager)

이 저장소는 public 이라 Argo CD 가 HTTPS 로 읽음 — 저장소 자격 증명을 등록하지 않음.

```bash
aws eks update-kubeconfig --region ap-northeast-2 --name team02-dev-eks
ARGOCD_VERSION=<강의 버전>
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/${ARGOCD_VERSION}/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
kubectl apply -f bootstrap/root.yaml      # 이후는 전부 Git
```

화면 접속 — 관리용 EC2 에서 `kubectl -n argocd port-forward svc/argocd-server 8080:443`,
내 PC 에서 `aws ssm start-session --target <admin_instance_id> --document-name AWS-StartPortForwardingSession --parameters portNumber=8080,localPortNumber=8080` → https://localhost:8080

## 03. 배포 흐름

```
앱 main commit → test → ECR push (태그 = SHA 앞 7자리) → 이 저장소 newTag 커밋
             → Argo CD 감지 (기본 3분 폴링) → 동기화 → 롤링 (maxUnavailable 0) → 새 버전 응답
```

## 04. Self Heal 확인

```bash
kubectl -n cafe set env deploy/cafe DEMO=drift     # Git 에 없는 값 → 잠시 뒤 사라짐
kubectl -n cafe delete svc cafe                    # 지운 리소스 → 다시 생김
```

replicas 는 HPA 가 정하므로 `ignoreDifferences` — `kubectl scale` 은 Self Heal 시연에 쓰지 않음.

## 05. 내릴 때 (terraform destroy 전)

```bash
kubectl -n argocd delete application cafe          # Ingress 삭제 → ALB Controller 가 ALB · TG 정리
aws elbv2 describe-load-balancers --names team02-dev-alb   # 없어질 때까지 확인
kubectl -n argocd delete application root          # 나머지 (finalizer 로 하위 리소스까지)
```

ALB Controller 가 먼저 지워지면 ALB 가 남아 VPC 삭제가 막힘 — 반드시 cafe → root 순서.

## 06. 알려진 문제

### ALB Controller webhook 인증서 불일치

**증상** — Service 생성이 거부됨 (다른 앱의 Service 까지 전부)

```
failed calling webhook "mservice.elbv2.k8s.aws": ... x509: certificate signed by unknown authority
```

**원인** — ALB 차트는 렌더링할 때마다 webhook 인증서를 새로 만듦. Argo CD 는 Helm `lookup` 을 쓰지 않아서
처음 동기화 중 재시도가 일어나면 Secret `aws-load-balancer-tls` 와 webhook `caBundle` 이 서로 다른 값으로 들어감.
한 번 짝이 맞게 만들어지면 `ignoreDifferences` + `RespectIgnoreDifferences` 로 유지됨 →
**ALB Controller 앱을 새로 만들 때(destroy 후 재구축 등)만** 생길 수 있음.

**조치** — Git 수정 없음. 지우고 Argo CD 가 한 번에 다시 만들게 함

```bash
kubectl delete mutatingwebhookconfiguration aws-load-balancer-webhook
kubectl delete validatingwebhookconfiguration aws-load-balancer-webhook
kubectl -n kube-system delete secret aws-load-balancer-tls
kubectl -n argocd annotate application aws-load-balancer-controller argocd.argoproj.io/refresh=hard --overwrite
# 위 3개가 다시 생긴 것 확인 후 — 컨트롤러가 새 인증서를 읽게 Pod 만 삭제 (rollout restart 는 Git 과 차이가 생김)
kubectl -n kube-system delete pod -l app.kubernetes.io/name=aws-load-balancer-controller
# 확인 — service/webhook-test 가 나오면 정상 (실제로 만들지 않음)
kubectl -n default create service clusterip webhook-test --tcp=80:80 --dry-run=server -o name
```

막혀 있던 앱(external-secrets · secret-store 등)은 재시도가 끝났으면 아래 수동 동기화로 다시 맞춤.

### 수동 동기화는 syncOptions 를 함께 넘김

`retry.limit` 을 다 쓴 앱은 같은 커밋에 대해 자동으로 다시 시도하지 않음 → 수동 동기화 필요.
수동 동기화는 앱에 적힌 `syncOptions` 를 따라가지 않으므로 **꼭 필요한 옵션은 같이 넘김**.
빠지면 ESO 처럼 CRD 가 큰 앱은 `metadata.annotations: Too long` 으로 실패함.

```bash
# external-secrets — ServerSideApply 필수
kubectl -n argocd patch application external-secrets --type merge \
  -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"syncOptions":["CreateNamespace=true","ServerSideApply=true"]}}}'

# 옵션이 없는 앱 (secret-store · storage 등)
kubectl -n argocd patch application <앱 이름> --type merge \
  -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{}}}'

# 결과 — Succeeded  Synced  Healthy
kubectl -n argocd get application <앱 이름> \
  -o jsonpath='{.status.operationState.phase}{"  "}{.status.sync.status}{"  "}{.status.health.status}{"\n"}'
```

화면에서 할 때는 SYNC → SYNC OPTIONS 에서 앱과 같은 옵션(예: Server-Side Apply)을 체크하고 SYNCHRONIZE.
