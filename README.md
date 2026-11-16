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
