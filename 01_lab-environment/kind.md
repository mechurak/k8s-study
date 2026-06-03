# kind (Kubernetes IN Docker)

로컬 멀티노드 클러스터 실습의 **메인 도구**. 컨테이너 하나를 노드처럼 띄워 vanilla k8s 클러스터를 구성한다. 가볍고, 깨고 다시 만들기 쉬워서 학습/실습에 최적.

> kind = **K**ubernetes **IN** **D**ocker. 각 "노드"가 도커 컨테이너이고, 그 안에서 kubelet/containerd가 돌며 파드를 실행한다.

---

## ⚡ 빠른 참조 (실습 시작·종료 커맨드)

### 🔧 최초 1회 — 환경 만들기
```bash
colima start --cpu 4 --memory 8 --disk 60   # VM 생성 (사양은 이때만 적용, k3s는 끈 채로)
kind create cluster --name study             # 클러스터 생성
kubectl get nodes                            # 노드 Ready 확인
```
> 멀티노드로 만들려면 `kind create cluster --name study --config kind-config.yaml` (아래 [멀티노드](#멀티노드-클러스터-실습-메인-구성) 참고).

### ▶️ 실습 시작할 때 (매번)
```bash
colima start          # VM 켜기 (저장된 사양 재사용 — 플래그 불필요)
kubectl get nodes     # 노드가 Ready 되면 시작
```

### ⏹️ 실습 끝낼 때 (매번)
```bash
colima stop           # VM 정지 — 클러스터/데이터 보존, 호스트 자원 회수
```

> 완전히 치우려면 `kind delete cluster --name study`(클러스터만) 또는 `colima delete`(VM까지). 자세한 건 아래 [일상 사용](#일상-사용-시작--종료-루틴).

---

## 언제 쓰고, 언제 안 쓰나

- 👍 멀티노드 토폴로지, 스케줄링, RBAC, NetworkPolicy, Ingress, CNI 교체 실습
- 👍 매니페스트를 빠르게 적용하고 클러스터를 통째로 버리고 새로 만들기
- 👎 kubeadm 업그레이드 / etcd 백업·복구 같은 "노드 OS 레벨" 시나리오는 제한적 → 이건 Multipass VM에서 (`README.md` 참고)

---

## 사전 요구사항: 컨테이너 런타임

kind는 **Docker** 또는 **Podman** 런타임이 필요하다. 노드가 컨테이너이므로 런타임이 먼저 떠 있어야 한다.

### macOS

옵션 두 가지 — 둘 중 하나만 있으면 된다.

**(A) colima (추천, 가볍고 무료)**
```bash
brew install colima docker kind kubectl
# 최초 1회만: VM을 이 사양으로 생성 (k8s는 끄고 도커 런타임만 제공)
colima start --cpu 4 --memory 8 --disk 60
docker info   # 런타임 정상 확인

# 이후부터는 플래그 없이 — 저장된 사양(4/8/60)을 그대로 재사용
colima start
```

> **플래그는 최초 생성 시에만 적용된다.** colima는 한 번 만든 VM의 사양을 저장하므로:
> - **최초 생성**: 플래그 없이 켜면 기본값(CPU 2 / **메모리 2GB** / 디스크 100GB)으로 만들어진다. 2GB는 멀티노드 kind엔 부족하니 **첫 start엔 반드시 플래그를 줄 것.**
> - **두 번째부터**: 그냥 `colima start` — 저장된 사양을 재사용하며 기본값으로 초기화되지 않는다.
> - **사양 변경**: `colima stop && colima start --memory 10 ...` 로 재시작. 단 디스크 축소는 불가(증가만 가능). 현재 사양은 `colima status` / `colima list`로 확인.

> **리소스 할당 가이드**
> colima는 경량 리눅스 VM을 띄우고 거기에 자원을 배정한다. VM이 켜진 동안 메모리는 호스트에서 비교적 단단히 잡히고(CPU는 놀 때 호스트가 회수), 디스크는 상한값이라 실제로는 쓰는 만큼만 늘어난다.
> - `--cpu 4 --memory 8 --disk 60`은 멀티노드 kind + 앱 몇 개에 적당한 중간값. **16GB+ 머신 기준** 안전한 기본값.
> - **호스트 몫을 남겨야 한다** — 메모리는 최소 6GB, CPU는 2~4코어를 macOS/앱용으로 남길 것. 전체 자원을 다 주지 말 것.
> - 내 머신 한계 확인: `sysctl -n hw.ncpu`(코어), `sysctl -n hw.memsize`(메모리 바이트), `df -h /`(디스크 여유).

**(B) Docker Desktop**
```bash
brew install --cask docker   # 설치 후 앱 실행
brew install kind kubectl
```
> Docker Desktop은 리소스 할당을 GUI(Settings → Resources)에서 조절. 멀티노드면 CPU 4 / Memory 8GB 이상 권장.

> ⚠️ Apple Silicon(arm64): kind 노드 이미지는 arm64를 지원한다. 다만 amd64 전용 컨테이너 이미지를 파드로 띄우면 에뮬레이션이 필요할 수 있으니, 실습 이미지는 멀티아키(`linux/arm64`) 지원 여부를 확인.

### Ubuntu

도커 엔진을 네이티브로 설치한다(Mac과 달리 VM 불필요).

```bash
# Docker Engine 설치 (공식 convenience script)
curl -fsSL https://get.docker.com | sh

# 현재 사용자를 docker 그룹에 추가 → sudo 없이 docker 사용
sudo usermod -aG docker $USER
newgrp docker          # 또는 로그아웃 후 재로그인
docker info            # 정상 확인
```

> rootless Docker나 Podman을 쓸 경우 kind에 추가 설정(cgroup, `KIND_EXPERIMENTAL_PROVIDER=podman` 등)이 필요할 수 있다. 처음엔 일반 Docker가 가장 무난.

---

## kind 설치

### macOS
```bash
brew install kind
```

### Ubuntu (바이너리 직접 설치)
```bash
# arch에 맞는 바이너리 (amd64 예시; arm64면 kind-linux-arm64)
[ "$(uname -m)" = "aarch64" ] && ARCH=arm64 || ARCH=amd64
curl -Lo ./kind "https://kind.sigs.k8s.io/dl/latest/kind-linux-${ARCH}"
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind version
```

### kubectl (공통)
```bash
# macOS
brew install kubectl

# Ubuntu
[ "$(uname -m)" = "aarch64" ] && ARCH=arm64 || ARCH=amd64
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/${ARCH}/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
kubectl version --client
```

---

## 첫 클러스터 (단일 노드)

```bash
kind create cluster --name study
kubectl get nodes
kubectl cluster-info --context kind-study
```

- kind는 클러스터 생성 후 kubeconfig 컨텍스트를 `kind-<name>`으로 자동 추가/전환한다.
- 삭제: `kind delete cluster --name study`

---

## 멀티노드 클러스터 (실습 메인 구성)

`kind-config.yaml`:
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

```bash
kind create cluster --name study --config kind-config.yaml
kubectl get nodes -o wide
```

### 특정 k8s 버전으로 띄우기 (CKA 버전 맞추기)

노드 이미지 태그로 버전을 고정한다. 태그는 kind 릴리스 노트에서 확인.
```bash
kind create cluster --name study \
  --image kindest/node:v1.32.0 \
  --config kind-config.yaml
```

---

## 자주 쓰는 부가 설정

### 포트 매핑 (로컬에서 Ingress/NodePort 접근)

`extraPortMappings`로 호스트 포트를 control-plane 노드에 연결한다.
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
  - role: worker
  - role: worker
```

### Ingress (ingress-nginx)
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```
> 위 포트 매핑(80/443)과 함께 쓰면 `http://localhost`로 Ingress 동작을 확인할 수 있다.

### NetworkPolicy 실습 (기본 CNI는 정책 미적용)

kind 기본 CNI(kindnet)는 NetworkPolicy를 강제하지 않는다. 정책 실습을 하려면 기본 CNI를 끄고 Calico 등을 설치한다.
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true        # kindnet 비활성화
  podSubnet: "192.168.0.0/16"    # Calico 기본 대역
nodes:
  - role: control-plane
  - role: worker
```
```bash
kind create cluster --name netpol --config kind-config.yaml
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

### 로컬 이미지를 클러스터로 로드

레지스트리 push 없이 로컬 빌드 이미지를 바로 쓸 수 있다.
```bash
docker build -t myapp:dev .
kind load docker-image myapp:dev --name study
# 매니페스트에서 imagePullPolicy: IfNotPresent 로 설정
```

---

## 일상 사용 (시작 / 종료 루틴)

실습 환경은 **3층 구조**다. 위층이 꺼지면 아래층도 멈추므로 **시작은 위→아래, 정리는 아래→위** 순서로 한다.

```
colima (리눅스 VM)  ──▶  docker (런타임)  ──▶  kind 클러스터 (컨테이너 = 노드)
```

> kind 클러스터는 colima VM 디스크에 저장되므로, **매번 새로 만들 필요 없다.** colima만 다시 켜면 노드 컨테이너가 자동 재시작되어 클러스터가 복구된다.

### 시작할 때
> 최초 1회는 사양 플래그(`--cpu 4 --memory 8 --disk 60`)를 줘서 VM을 만들고, 이후부터는 아래처럼 플래그 없이 켜면 저장된 사양을 재사용한다.

```bash
colima start                # 1) VM + docker 켜기 (저장된 사양 재사용)
kind get clusters           # 2) study 가 보이면 클러스터 살아있음
kubectl get nodes           # 3-a) 있으면: 노드 Ready 확인 후 실습 (복구에 수십 초 걸릴 수 있음)
# 없으면(처음/삭제했으면):
kind create cluster --name study --config kind-config.yaml   # 3-b) 생성
```

### 끝낼 때 — 두 가지 선택지

| 상황 | 명령 | 보존되는 것 |
|------|------|------------|
| **잠깐 중단 (일상, 추천)** | `colima stop` | 클러스터 + 모든 데이터 |
| 클러스터만 새로 | `kind delete cluster --name study` → 재생성 | VM (docker) |
| 디스크까지 비우기 (완전 초기화) | `colima delete` | 아무것도 |

```bash
# 가장 흔한 일상 루틴
colima start && kubectl get nodes     # 시작
colima stop                           # 종료 (호스트 CPU/메모리 회수, 데이터 보존)
```

---

## 클러스터 관리 명령 모음

```bash
kind get clusters                       # 클러스터 목록
kind get nodes --name study             # 노드(컨테이너) 목록
kubectl config get-contexts             # 컨텍스트 확인
kubectl config use-context kind-study   # 컨텍스트 전환
kind delete cluster --name study        # 삭제
kind delete clusters --all              # 전부 삭제
```

kubeconfig를 따로 뽑기:
```bash
kind get kubeconfig --name study > study.kubeconfig
export KUBECONFIG=$PWD/study.kubeconfig
```

---

## 트러블슈팅

| 증상 | 원인 / 해결 |
|------|-------------|
| `ERROR: failed to create cluster ... Docker not running` | 런타임 미기동 → macOS: `colima start` 또는 Docker Desktop 실행 / Ubuntu: `sudo systemctl start docker` |
| `colima start` 실패: `dependency check failed for docker: docker not found` | docker가 설치돼 있어도 brew link가 안 된 상태. 흔한 원인은 **`docker-completion`(deprecated)** 이 자동완성 파일을 선점해 충돌. 해결: `brew uninstall docker-completion`(요즘 `docker` 포뮬러가 자동완성 자체 포함) 후 `brew link docker`. `which docker`로 확인. |
| `colima start` 로그에 `runtime: docker+k3s` | 저장된 설정에 colima 내장 k3s가 켜져 있음. kind를 쓸 거면 불필요 → `colima start --kubernetes=false`로 끄기 |
| `colima start` 로그: `disk size cannot be reduced, ignoring` | colima는 디스크 **축소 불가**(증가만 가능). 기존 VM이 더 큰 디스크로 만들어졌을 때 뜨는 정상 경고, 무시해도 됨 |
| Ubuntu에서 `permission denied ... docker.sock` | `sudo usermod -aG docker $USER` 후 재로그인 |
| 노드가 `NotReady` | CNI 미설치(특히 `disableDefaultCNI: true`로 만든 경우) → Calico 등 설치 |
| 파드가 `ImagePullBackOff` (arm64 Mac) | amd64 전용 이미지 → 멀티아키 이미지 사용 또는 `kind load`로 직접 로드 |
| 메모리 부족으로 파드 Pending/Evicted | 런타임 리소스 상향 (colima `--memory`, Docker Desktop Settings) |
| inotify 한계로 다수 파드 생성 실패(Linux) | `sysctl fs.inotify.max_user_watches`, `max_user_instances` 상향 |

---

## macOS vs Ubuntu 요약

| 항목 | macOS | Ubuntu |
|------|-------|--------|
| 컨테이너 런타임 | colima 또는 Docker Desktop (경량 VM 내부) | Docker Engine 네이티브 |
| 리소스 할당 | colima/Docker Desktop 설정에서 명시 | 호스트 자원 직접 사용 |
| 아키텍처 주의 | Apple Silicon=arm64 이미지 확인 | 보통 amd64 (서버) |
| 호스트 네트워크 접근 | VM 경유 → `extraPortMappings` 권장 | 비교적 직접적이나 동일 방식 권장 |

---

## 참고 자료

- [kind 공식 문서](https://kind.sigs.k8s.io/)
- [kind – Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [kind – Ingress](https://kind.sigs.k8s.io/docs/user/ingress/)
- [kind – Loading an Image](https://kind.sigs.k8s.io/docs/user/local-registry/)
- [colima](https://github.com/abiosoft/colima)
