# AWS Load Balancer Controller & TargetGroupBinding

> EKS에서 **외부 트래픽을 어떻게 받나**의 실무판. on-prem의 [ingress-nginx + MetalLB](../04_services-networking/ingress.md#controller는-외부와-어떻게-연결되나--노출-방식-특히-on-prem) 모델과 대비해서 본다.
> 관련: [04 ingress.md](../04_services-networking/ingress.md) · [04 metallb.md](../04_services-networking/metallb.md) · [README](./README.md)

## 개념 — EKS엔 MetalLB가 필요 없다

on-prem은 `Service type: LoadBalancer`에 IP를 채워줄 [MetalLB](../04_services-networking/metallb.md) 같은 구현체가 필요했다. **EKS는 AWS가 그 역할을 한다** — 클러스터에 **AWS Load Balancer Controller**(이하 LB 컨트롤러)를 깔면, k8s 리소스를 보고 **실제 AWS ALB/NLB·Target Group을 만들거나 연결**해준다.

> 💡 LB 컨트롤러는 보통 `kube-system`에 Deployment로 뜨고, **IRSA**(IAM Roles for Service Accounts)로 ELB API 권한을 받는다. ALB/NLB를 직접 프록시하는 게 아니라, **AWS API를 호출해 LB를 조작**하는 컨트롤 플레인 역할.

## LB 컨트롤러가 일하는 3가지 방식

| 방식 | 누가 LB/TG를 만드나 | 트리거 | LB 종류 |
|---|---|---|---|
| **Ingress** | 컨트롤러가 **ALB를 새로 생성** | `Ingress` + `alb.ingress.kubernetes.io/*` annotation | ALB (L7) |
| **Service type: LoadBalancer** | 컨트롤러가 **NLB를 새로 생성** | `Service` + `service.beta.kubernetes.io/aws-load-balancer-*` annotation | NLB (L4) |
| **TargetGroupBinding** | **당신(Terraform/콘솔)이 미리 만든** LB·TG·리스너 | 기존 **Target Group ARN**을 CRD로 연결 | 무엇이든(기존 것) |

> 사실 Ingress·Service 방식도 내부적으로는 **TargetGroupBinding을 자동 생성**해서 동작한다. TGB가 가장 low-level이고, 나머지는 그 위의 편의 계층.

## TargetGroupBinding — "LB는 IaC가, 타깃만 k8s가"

**이미 존재하는 AWS Target Group에 이 k8s Service의 대상(파드/노드)을 자동 등록/해제**하라고 시키는 CRD.

```yaml
# 📖 AWS LB Controller: https://kubernetes-sigs.github.io/aws-load-balancer-controller/
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: my-app
  namespace: my-ns
spec:
  serviceRef:
    name: my-app          # 이 k8s Service의
    port: 80
  targetGroupARN: arn:aws:elasticloadbalancing:ap-northeast-2:...:targetgroup/my-app-tg/abc123
  targetType: ip          # ip | instance
```

→ 컨트롤러가 이 Service의 엔드포인트를 감시하다가, 그 ARN의 Target Group에 **파드 IP(또는 노드:NodePort)를 AWS API로 등록**한다. 파드가 뜨고 죽을 때마다 타깃을 **동기화**.

### 왜 이 방식을 쓰나

- **LB 수명주기를 k8s 밖(Terraform/네트워크팀)에서 통제**하고 싶을 때. ALB·TG·리스너·보안그룹·WAF·인증서는 IaC가 소유하고, k8s 컨트롤러에는 **"타깃 등록"만** 맡긴다.
- LB를 여러 서비스/계정이 공유하거나, k8s 밖 대상(EC2 등)과 **같은 TG에 섞어** 넣어야 할 때.
- 엔터프라이즈에서 흔한 패턴 — "클러스터를 지워도 LB·DNS·인증서는 그대로 남는다".

## ⚠️ 데이터 경로 — `ip` 모드는 파드로 직행

| target type | 등록 대상 | 트래픽 경로 |
|---|---|---|
| **`ip`** (EKS 기본·권장) | **파드 IP 직접** (VPC CNI가 파드에 진짜 VPC IP 부여) | LB → **파드 IP로 바로** |
| **`instance`** | 노드 + NodePort | LB → 노드:NodePort → kube-proxy → 파드 |

```
[ip 모드]
인터넷 → ALB/NLB → (Target Group: 파드 IP들) → 파드 ENI로 직접 → 파드
                                                 ↑ kube-proxy·ClusterIP를 '안 거침'
```

> 💡 `ip` 모드에서 **Service(ClusterIP)는 데이터 경로 홉이 아니라 "어떤 파드들인지 알아내는 명단"으로만** 쓰인다. [metallb.md의 "Service는 홉이 아니다"](../04_services-networking/metallb.md#️-트래픽-경로--metallbservice는-홉이-아니다)와 같은 맥락 — 다만 EKS ip 모드는 **kube-proxy조차 안 거치고** LB가 파드 IP로 직행한다(VPC CNI라 파드가 VPC 1급 IP를 가져 가능).
>
> ⚠️ 그래서 **보안그룹이 LB → 파드 방향을 허용**해야 한다. TGB의 `spec.networking`으로 컨트롤러가 SG 규칙을 관리하게 하거나, 노드/파드 SG에 직접 열어준다.

## on-prem(ingress-nginx + MetalLB) 모델과 비교

| | on-prem ([04](../04_services-networking/ingress.md)) | EKS (이 문서) |
|---|---|---|
| 외부 진입 IP | **MetalLB**가 IP 광고 | **AWS ALB/NLB** (관리형) |
| L7 라우팅 주체 | **ingress-nginx Pod** | **ALB 리스너 규칙** (또는 뒤에 ingress-nginx를 또 둘 수도) |
| 타깃 연결 | kube-proxy(iptables)가 Service→Pod | **LB 컨트롤러가 파드 IP를 TG에 등록** |
| 파드까지 | 노드 거쳐 DNAT | **LB → 파드 IP 직행**(ip 모드) |
| LB 수명주기 | 클러스터 안 | **AWS(+IaC) 쪽**, TGB로 연결만 |

## 확인 / 진단

```bash
# TargetGroupBinding 목록 — 어떤 Service가 어떤 TG에 묶였나
kubectl get targetgroupbinding -A
kubectl -n <ns> get targetgroupbinding <name> \
  -o jsonpath='svc={.spec.serviceRef.name}:{.spec.serviceRef.port}  type={.spec.targetType}  tg={.spec.targetGroupARN}{"\n"}'

# 컨트롤러가 떠 있나 (+ IRSA 권한 문제는 이 로그에)
kubectl -n kube-system get deploy aws-load-balancer-controller
kubectl -n kube-system logs deploy/aws-load-balancer-controller

# 그 Service의 실제 파드(= TG에 등록되는 대상)
kubectl -n <ns> get endpointslices -l kubernetes.io/service-name=<svc> -o wide

# AWS 쪽에서 타깃 헬스까지 (CLI)
aws elbv2 describe-target-health --target-group-arn <arn>
```

| 증상 | 원인 | 확인 |
|---|---|---|
| TG에 타깃이 안 올라옴 | 컨트롤러 IRSA 권한 부족 / Service 셀렉터 불일치 | 컨트롤러 로그, EndpointSlice 비었나 |
| 등록은 됐는데 **unhealthy** | 보안그룹이 LB→파드 차단 / 헬스체크 경로 틀림 | SG 규칙, TG 헬스체크 설정 |
| 파드 교체 시 502 순간 | deregistration delay·readiness 미설정 | `pod readiness gate`, TG draining |

> 💡 **Pod readiness gate** — LB 컨트롤러가 "타깃이 TG에서 healthy 될 때까지 파드를 Ready로 안 친다"를 넣어주면 롤링 배포 중 트래픽 유실을 막는다(`elbv2.k8s.aws/pod-readiness-gate-inject: enabled` 네임스페이스 라벨).

## 시험·실무 팁

- **CKA 범위 아님** — 순수 AWS/EKS 실무. vanilla k8s엔 이 CRD가 없다.
- **`ip` 모드가 EKS 표준** (VPC CNI 전제). `instance` 모드는 NodePort 경유라 홉이 하나 더 생기고 SG가 단순한 대신 파드 직접 분산은 아니다.
- **TargetGroupBinding을 봤다 = LB가 k8s 밖(IaC)에서 관리된다는 신호.** 그 LB·리스너·DNS는 Terraform/콘솔에서 찾아야 한다(매니페스트엔 ARN만 있음).
- 인증서(ACM)·WAF·로깅 등은 **LB(AWS) 쪽 속성** — k8s가 아니라 AWS 콘솔/IaC에서 본다.

## 참고

- [AWS Load Balancer Controller (공식)](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [TargetGroupBinding 스펙](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/targetgroupbinding/targetgroupbinding/)
- [Target type: ip vs instance](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/service/annotations/#target-type)
- [Pod readiness gate](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/deploy/pod_readiness_gate/)
- 관련 문서 → [04 ingress.md](../04_services-networking/ingress.md) · [04 metallb.md](../04_services-networking/metallb.md)
