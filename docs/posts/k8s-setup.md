---
date: 2025-09-27
categories:
  - Computer Science
  - DevOps
---

# Kubernetes

![k8s.svg](../assets/k8s.svg)


Kubernetes(K8s) 는 컨테이너화된 애플리케이션의 배포, 확장, 관리를 자동화해주는 오픈소스 클러스터 서비스이다.
구글이 내부적으로 사용하던 Borg 클러스터 설계 노하우를 바탕으로 제작되었으며, 현재는 Container Orchestration 분야의 표준으로 자리잡았다.
Docker 가 애플리케이션을 "어떻게 배포할 것인지"를 해결했다면, 쿠버네티스는 "수많은 컨테이너를 어떻게 운영하고 자동화할지"를 해결한다.

1. **자동 회복**: 컨테이너가 다운되거나 응답하지 않을 경우 자동으로 재시작
2. **로드 밸런싱**: 트래픽을 여러 파드로 분산시켜 안정적인 서비스를 제공
3. **오토 스케일링**: 자원 사용량에 따라 자동으로 컨테이너 개수를 늘리거나 줄이고 트래픽 요구사항에 대응
4. **무중단 배포**: 서비스 중단 없이 롤링 업데이트를 진행하고, 문제 발생 시 이전 버전으로 롤백

무중단 서비스를 운영하거나 규모 있는 개발을 진행한다면 DevOps 와 함께 쿠버네티스 도입을 적극 고려해보자.
필자는 클러스터를 직접 구축했던 경험을 가이드라인 형태로 공유하고자 한다.

<!-- more -->

# 쿠버네티스 설치

## 사전 준비단계
사전 준비에 해당되는 내용들은 master node, worker node 에 모두 적용된다.

쿠버네티스(K8s) 작업을 수행하는 모든 노드에 접속해서 빠짐없이 실행해야 한다!

### Docker Engine 설치
먼저 [Docker 공식문서](https://docs.docker.com/engine/install/)를 참조하여 도커를 설치한다.

도커가 설치되면 컨테이너 런타임 `Containerd` 도 자동으로 설치된다. K8s 가 컨테이너를 관리할 때 이것이 사용된다.

### 통신모듈 설정

K8s 클러스터 내부에서 통신이 가능하려면 리눅스 네트워크 설정을 변경해야 한다.

먼저 커널 공간에서 패킷을 필터링하는 `br_netfilter` 모듈을 활성화한다.

```shell
sudo modprobe br_netfilter
```

시스템 재부팅 시 초기화되는 것을 막으려면 `modules-load` 설정이 필요하다.

```shell
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF
```

그리고 커널 브릿지를 통과하는 패킷이 `iptables` 방화벽도 거치도록 sysctl 설정을 변경한다. 

```shell
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

### 방화벽 설정
K8s 컴포넌트가 정상 작동하려면 API 요청이 방화벽에서 차단되지 않도록 포트를 개방해야 한다.

[각 프토로콜에 대응하는 포트 번호](https://kubernetes.io/docs/reference/networking/ports-and-protocols/)를 참조하여 클러스터 내부 트래픽이 해당 포트를 사용할 수 있도록 방화벽을 설정하자.

### 스왑 메모리 해제
리눅스는 기본적으로 메모리의 초과 사용을 대비해서 디스크 공간 일부를 메모리로 사용한다.

하지만 K8s 자원관리 체계는 스왑 메모리를 고려하지 않고 설계되었기 때문에 [이 기능을 해제](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#swap-configuration)해야 한다.

### kubeadm CLI 설치
[공식 문서를 참조](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)하여 클러스터를 생성하고 K8s 컴포넌트를 제어하는 명령줄 도구를 설치해보자.

컨테이너 런타임과 kubelet 이 사용하는 `cgroup driver`가 일치하는지 반드시 확인해보자.

## 클러스터 생성하기
Control plane 은 마스터 노드에서 설치해야 한다.

방화벽 설정에서 Kubernetes API 서버용 포트 6443 이 loopback 에서 개방되었는지 확인해보자.

```shell
nc 127.0.0.1 6443 -zv -w 2
```

### Control plane 생성
```shell
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=<내부IP>
```

위 명령을 실행해서 클러스터 및 control plane 이 성공적으로 생성되면 다음과 같은 문구가 뜬다.

```
Your Kubernetes control-plane has initialized successfully!

...

You can now join any number of machines by running the following on each node
as root:

  kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

문구 맨 아래에 `kubeadm join ...` 명령을 복사해서 메모장에 기록해두자. 나중에 worker node 를 생성하고 붙일 때 필요하다.

이제 kubectl 명령어 접근권한을 취득하기 위해 아래 명령을 입력하자.

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

kubectl 작동을 확인해보는 차원에서 모든 namespace 공간에 존재하는 Pod 목록을 조회해보자.

```shell
kubectl get pods -A
```

```
NAMESPACE      NAME                             READY   STATUS    RESTARTS       AGE
kube-system    etcd-master                      1/1     Running   5              78m
kube-system    kube-apiserver-master            1/1     Running   5              78m
kube-system    kube-controller-manager-master   1/1     Running   5              78m
kube-system    kube-proxy-gd4da                 1/1     Running   2 (45m ago)    75m
kube-system    kube-scheduler-master            1/1     Running   5              78m
```

### CNI 플러그인 설치
클러스터 DNS 를 포함해서 Pod 끼리 통신할 수 있는 네트워크를 구축하려면 CNI(Container Network Interface) 가 필요하다.

K8s 클러스터와 호환되는 CNI 플러그인 목록은 [여기에서 참고](https://kubernetes.io/docs/concepts/cluster-administration/addons/#networking-and-network-policy).
필자는 설치 및 관리가 편리한 flannel 플러그인을 선택하겠다.

1. flannel 공식 저장소에서 [kube-flannel.yml](https://github.com/flannel-io/flannel/blob/master/Documentation/kube-flannel.yml) 파일을 내려받기
2. `kubectl apply -f kube-flannel.yml` 을 실행해서 flannel 플러그인 설치
3. 만약 설치후 `/run/flannel/subnet.env` 파일이 없다면 직접 추가하기


4. `kubectl get pods -n kube-flannel` 을 실행해서 생성된 Pod 확인
```
NAME                    READY   STATUS    RESTARTS       AGE
kube-flannel-ds-5sd3d   1/1     Running   0              80m
```

#### 주의사항
- flannel 설치 후 `/run/flannel/subnet.env` 파일이 생성되지 않는 경우가 있다. 그렇다면 [직접 생성해야 한다](https://github.com/flannel-io/flannel/blob/master/dist/sample_subnet.env).
- 특정 커널 환경에서 VXLAN 패킷의 체크섬이 변질되는 문제가 있다. [`tx-checksum-ip-generic` 옵션을 해제](https://github.com/flannel-io/flannel/blob/master/Documentation/troubleshooting.md#nat)하여 해결할 수 있다.

### Worker node 등록
이제 K8s 컨테이너를 실행하는 worker node 컴퓨터에 접속한다.

이전에 Control plane 만들고 나서 복사했던 명령어를 실행하면 된다. 아래는 예시.

```
kubeadm join 192.168.1.11:6443 --token ssakq.03psdfgs4eq31omogi \
        --discovery-token-ca-cert-hash sha256:13se53sxd8dnh3k08834ssdsw4e488fbbbf2eebaaf037b2
```

등록이 끝났다면 마스터 노드에 접속해서 worker node 가 잘 인식되는지 확인해보자.

```shell
kubectl get nodes
```

# Troubleshoot

## context deadline exceeded
```shell
$ kubectl apply -f manifest.yaml

Error from server (InternalError): error when creating "manifest.yaml": Internal error occurred:
failed calling webhook "ipaddresspoolvalidationwebhook.metallb.io": failed to call webhook:
Post "https://metallb-webhook-service.metallb-system.svc:443/validate-metallb-io-v1beta1-ipaddresspool?timeout=10s": context deadline exceeded 
```

마니페스트를 적용할 때 웹훅 연결이 안되서 리소스 생성에 실패하는 문제를 겪었다.

클러스터 네트워크를 구현하는 `flannel` 이 패킷을 전송하는 특정 포트가 있는데, 이게 방화벽에서 차단된 것이었다.

[공식 문서에서 방화벽 관련 권고사항](https://github.com/flannel-io/flannel/blob/master/Documentation/troubleshooting.md#firewalls)을 확인하고 방화벽을 수정하여 해결했다.
