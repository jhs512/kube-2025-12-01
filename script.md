```bash
hostnamectl set-hostname ec2-1
```

```bash
sudo tee /etc/profile.d/ec2_metadata_env.sh > /dev/null << 'EOF'
#!/bin/bash

# IMDSv2 토큰 요청
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# 토큰이 정상적으로 발급된 경우에만 변수 설정
if [ -n "$TOKEN" ]; then
  # 공인 IPv4
  export EC2_PUBLIC_IP=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/public-ipv4)

  # 공인 Hostname
  export EC2_PUBLIC_HOSTNAME=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/public-hostname)
fi
EOF
```

```bash
sudo chmod +x /etc/profile.d/ec2_metadata_env.sh
```

```bash
source /etc/profile.d/ec2_metadata_env.sh
```

```bash
echo $EC2_PUBLIC_IP
echo $EC2_PUBLIC_HOSTNAME
```

```bash
sudo yum install -y bash-completion

# bash-completion 로드
echo '[[ $PS1 && -f /usr/share/bash-completion/bash_completion ]] && \
  source /usr/share/bash-completion/bash_completion' >> ~/.bashrc

# kubectl completion
echo 'source <(kubectl completion bash)' >> ~/.bashrc

# alias
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc

source ~/.bashrc
```

```bash
kubeadm init --control-plane-endpoint "$EC2_PUBLIC_HOSTNAME:6443" --upload-certs --pod-network-cidr=10.10.0.0/16 -v=5
```

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```bash
kubectl describe node ec2-1 | grep -i Taints # 확인
kubectl taint nodes ec2-1 node-role.kubernetes.io/control-plane- # 제거
kubectl describe node ec2-1 | grep -i Taints # 확인
```

```bash
# ==========================================
# Calico 설치 (Tigera Operator 권장 방식)
# - 환경: AWS EC2, kubeadm, containerd
# - CNI: Calico (VXLAN), BGP 끔
# - 추후 워커 노드 수동 추가, MetalLB + Ingress-Nginx 사용 예정
# - pod CIDR: 10.10.0.0/16 (kubeadm init --pod-network-cidr 와 동일해야 함)
# ==========================================

# 설치할 Calico 버전 (문서 기준 최신 세대)
CALICO_VERSION=v3.31.2

# ------------------------------------------
# 1. Tigera Operator CRD + Operator 설치
#    - CRD: Calico가 쓸 커스텀 리소스 타입(Installation, APIServer 등)을 K8s에 등록
#    - Operator: Calico 설치/업그레이드/구성을 관리하는 컨트롤러
# ------------------------------------------
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests/operator-crds.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests/tigera-operator.yaml

# ------------------------------------------
# 2. Calico 설치/구성용 Custom Resource YAML 작성
#    - /kube/calico-resources.yaml 에 Installation + APIServer 정의
#    - bgp: Disabled  → BGP 라우팅 끄고, VXLAN 오버레이만 사용 (소규모 + MetalLB L2 에 적합)
#    - encapsulation: VXLAN → 노드 추가 시에도 자동으로 Pod 간 통신 가능
#    - cidr: 10.10.0.0/16 → kubeadm pod-network-cidr 와 반드시 동일
#    - natOutgoing: Enabled → Pod → 외부(인터넷/외부 API) 통신 시 SNAT 수행 (EC2 환경에서 필수)
# ------------------------------------------
mkdir -p /kube
cd /kube

cat << 'EOF' > /kube/calico-resources.yaml
# Calico 기본 설치 설정
# 참고: https://docs.tigera.io/calico/latest/reference/installation/api#operator.tigera.io/v1.Installation
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default  # 싱글턴 리소스, 이름은 항상 default
spec:
  calicoNetwork:
    # BGP 끔
    # - 단일 마스터 + 소수 워커 + MetalLB L2 모드에 적합
    # - Calico가 노드 간 라우팅을 BGP로 하지 않고 VXLAN 오버레이로 처리
    bgp: Disabled

    # Pod IP 풀 설정
    ipPools:
      - name: default-ipv4-ippool  # IP 풀 이름 (원하면 변경 가능)
        # blockSize: 노드에게 할당되는 최소 IP 블록 크기 (/26 = 64 IP)
        # - 소규모 클러스터에서는 기본값(/26)이 가장 안정적
        blockSize: 26

        # Pod 대역 (CIDR)
        # - kubeadm init 의 --pod-network-cidr 와 반드시 동일해야 함
        #   예) kubeadm init --pod-network-cidr=10.10.0.0/16 ...
        cidr: 10.10.0.0/16

        # VXLAN-only 모드
        # - 모든 노드 간 Pod 트래픽을 VXLAN 오버레이로 처리
        # - BGP 없이 멀티 노드 통신 가능
        encapsulation: VXLAN

        # Pod → 외부(인터넷, 외부 API 등) 통신 시 SNAT 수행
        # - AWS EC2, 온프렘 스타일(별도 라우팅 없음)에서는 필수
        natOutgoing: Enabled

        # 이 IP 풀을 사용할 노드 선택
        # - all() 이면 클러스터의 모든 노드에 적용
        nodeSelector: all()

---

# Calico API 서버 설정
# - calicoctl 등에서 Calico 전용 API를 쓰고 싶을 때 필요
# - 리소스가 있어도 성능 부담 거의 없음
# 참고: https://docs.tigera.io/calico/latest/reference/installation/api#operator.tigera.io/v1.APIServer
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default  # 마찬가지로 싱글턴
spec: {}
EOF

# ------------------------------------------
# 3. Calico 설치 리소스 적용
#    - Installation / APIServer 리소스를 생성하면
#      Tigera Operator 가 이를 보고 Calico 구성 요소들을 자동 배포
# ------------------------------------------
kubectl create -f /kube/calico-resources.yaml

# ------------------------------------------
# 4. (옵션) containerd 재시작
#    - /etc/containerd/config.toml 수정(SystemdCgroup=true 등) 직후라면 재시작 필수
#    - 단순 Calico 설치만 했다면 굳이 재시작할 필요는 없음
# ------------------------------------------
# systemctl restart containerd

# ------------------------------------------
# [참고] 나중에 워커 노드 추가할 때
# 1) 워커 EC2 에서 containerd / kubelet / kubeadm 설치
# 2) 마스터에서 kubeadm join 명령 재발급:
#      kubeadm token create --print-join-command
# 3) 워커에서 해당 join 명령 실행
# 4) Tigera Operator 가 새 노드에도 자동으로 calico-node DaemonSet 배포
#    → 별도 Calico 설정 변경 없이 멀티 노드로 확장 가능
# ------------------------------------------
```

```bash
kubectl wait --for=condition=Ready node/ec2-1 --timeout=300s
```

```bash
# 1) MetalLB 설치
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml

# 2) MetalLB 컨트롤러가 살아날 때까지 대기
#    - 웹훅이 준비되기 전에 IPAddressPool 만들면 connection refused 에러 나서 이거 하나만 추가
kubectl wait -n metallb-system \
  --for=condition=Available deployment/controller \
  --timeout=300s

# 3) MetalLB IP 풀 + L2 광고 설정 파일 생성
#    - EC2_PUBLIC_IP 환경변수에 들어있는 공인 IP 한 개만 풀로 사용
cat <<EOF | tee metallb-config.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: public-pool
  namespace: metallb-system
spec:
  addresses:
    - "${EC2_PUBLIC_IP}-${EC2_PUBLIC_IP}"
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: public-ad
  namespace: metallb-system
spec:
  ipAddressPools:
    - public-pool
EOF

# 4) 설정 적용
kubectl apply -f metallb-config.yaml

# 5) 확인용(선택)
kubectl get pods -n metallb-system
kubectl get ipaddresspool -n metallb-system
kubectl get l2advertisement -n metallb-system
```

```bash
# Helm 설치 스크립트 내려받기
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4

# 스크립트에 실행 권한 추가
chmod 700 get_helm.sh

# 스크립트 실행 → 최신 Helm 설치
./get_helm.sh

# 설치 확인
helm version
```

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

helm repo update

helm search repo ingress-nginx

mkdir -p /kube/ingress-nginx

cd /kube/ingress-nginx

helm show values ingress-nginx/ingress-nginx > nginx-ingress.yaml

sed -i '
  # hostNetwork true로
  s/  hostNetwork: .*/  hostNetwork: true/;

  # hostPort 블록 내부의 enabled만 true로 교체
  /hostPort:/,/ports:/ s/^ *enabled: .*/    enabled: true/;

  # kind를 Deployment → DaemonSet
  s/  kind: Deployment/  kind: DaemonSet/;
' nginx-ingress.yaml

helm install ingress-nginx ingress-nginx/ingress-nginx -f nginx-ingress.yaml -n ingress-nginx --create-namespace
```

```bash
# ingress-nginx 네임스페이스의 controller Pod 준비 대기 (최대 120초)
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  -l app.kubernetes.io/component=controller \
  --timeout=120s
```

```bash
mkdir -p /kube/ingress-test && cd /kube/ingress-test

cat <<EOF > ingress-test.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy-main
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-main
  template:
    metadata:
      labels:
        app: nginx-main
    spec:
      containers:
      - image: nginx
        name: nginx
---

apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: nginx-main
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80

---

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: ${EC2_PUBLIC_HOSTNAME}   # ← 여기서 변수 확장됨
    http:
      paths:
      - path: /
        pathType: ImplementationSpecific
        backend:
          service:
            name: my-service
            port:
              number: 80
EOF

# 적용
kubectl apply -f ingress-test.yaml
```