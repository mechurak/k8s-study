# 09. AWS EKS

업무에서 운영하는 **관리형 k8s(EKS)** 실무 메모. CKA의 vanilla k8s와 다른 점(관리형 control plane, AWS 통합)에 집중.

## 학습 목표

- [ ] EKS 아키텍처 — 관리형 control plane vs 데이터플레인(노드그룹/Fargate)
- [ ] 클러스터 생성 — `eksctl`, Terraform, 콘솔
- [ ] 인증/인가 — IAM ↔ k8s RBAC 매핑, **IRSA**(IAM Roles for Service Accounts), EKS Pod Identity
- [ ] 네트워킹 — VPC CNI, 보안그룹, LoadBalancer ↔ AWS LB Controller
- [ ] 스토리지 — EBS / EFS CSI 드라이버
- [ ] 노드 관리 — Managed Node Group, Karpenter, 오토스케일링
- [ ] 업그레이드 전략 (control plane / 노드)
- [ ] 관측성 — CloudWatch, Container Insights
- [ ] 비용 관리 ⚠️

## 빠른 시작 / 정리

```bash
brew install eksctl awscli
eksctl create cluster --name dev --nodes 2 --node-type t3.medium

# ⚠️ 실습 끝나면 반드시 삭제 (control plane + EC2 시간당 과금)
eksctl delete cluster --name dev
```

> ⚠️ CKA 학습 자체는 EKS에서 하지 말 것 — 관리형이라 etcd/kubeadm 등 시험 범위를 만질 수 없다. EKS는 "실무 운영" 관점으로 분리.

## 정리

> (실무하며 채워나갈 자리)

## 참고

- [Amazon EKS 사용 설명서](https://docs.aws.amazon.com/eks/latest/userguide/)
- [eksctl](https://eksctl.io/)
- [EKS Best Practices Guide](https://aws.github.io/aws-eks-best-practices/)
