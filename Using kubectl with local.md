# 로컬 환경에서 kubectl 사용하기

본 포스팅은 [NHN의 k8s 교육](https://www.nhncloud.com/kr/edu/attend)을 수강하고 나서 작성된 글입니다.

수업 내용는 아래의 포스팅과 직접적인 관계ㅡㄴ 없지만, k8s를 처음 접하는 사용자의 입장에서 기초를 잡는것에 많은 도움이 되었습니다.

`kubectl`이란 k8s 클러스터의 control plane과 통신하기 위한 명령줄 인터페이스입니다. 즉, 생성된 클러스터에 명령어를 사용하기 위해 필요한 도구입니다. 이번 글에서는 `kubectl`을 로컬 환경에 설치하여 클라우드 장비에 구동된 클러스터에 명령어를 보내는 방법을 알아보겠습니다.

## kubeConfig 파일이 주어진 상황

해당 챕터에서는 Cloud 서비스 중 NHN의 k8s 서비스인 NKS의 클러스터에 로컬 환경(intel mac)에서 접속하는 방법에 대해 다뤄봅니다. 다른 클라우드 서비스는 사용해보지 않았지만 다른 서비스의 경우에도 가이드를 통해 kubeConfig 파일을 얻어낼 수 있는것으로 보입니다. aws의 EKS의 경우 [다음](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/create-kubeconfig.html)과 같은 가이드를 제공하여 사용자로 하여금 kubeConfig 파일을 이용할 수 있도록 해줍니다.
NHN의 NKS의 경우 아래 이미지에서 볼 수 있듯이 GUI를 통해 간편하게 kubeConfig 파일을 사용자에게 제공하고 있습니다.
![Alt text](<스크린샷 2023-12-08 오후 2.11.12.png>)
해당 파일을 다운로드 한 뒤 작업을 시작하면 됩니다.

먼저 local에 kubectl을 설치해야합니다. [문서](https://kubernetes.io/docs/tasks/tools/)를 참고해도 됩니다. 아래는 Intel Mac을 기준 명령어입니다.

```shell
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
sudo chown root: /usr/local/bin/kubectl
```

설치를 완료했다면 `kubectl version --client`를 통해 확인합니다. 아래와 같은 결과가 나옵니다.

```
Client Version: v1.28.4
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
```

이제부터는 클라우드 서비스에 띄워져있는 클러스터의 control plane과 통신하기 위한 설정을 해줘야합니다. `kubectl`은 기본적으로 `$HOME/.kube` 디렉토리에서 config라는 이름의 파일을 찾아 이를 설정값으로 하여 cluster와 통신을 합니다.해당 config 파일에는 통신할 cluster의 정보가 담겨 있습니다.
아래의 명령어를 통해 저희가 가지고 있는 kubeconfig 파일을 로컬의 `$HOME/.kube/config`로 복사하는 작업을 실행합니다.

```shell
mkdir ~/.kube # 해당하는 Dir이 없을 경우 생성
cp xxx.yaml ~/.kube/config
```

config 파일을 클라우드가 제공하는 kubeconfig 파일의 내용으로 변경했다면 `kubectl get nodes`를 통해 해당 클러스터가 가지고 있는 worker node 리스트를 받아볼 수 있습니다.

지금까지 단일 클러스터에 대해서 로컬에서 kubectl을 사용하는 법을 알아보았습니다. 지금부터는 클러스터가 여러개 있을 경우 해당 config을 어떻게 구성해야 kubectl을 사용할 수 있는지에 대해 알아보겠습니다.

앞서 진행한 절차와 동일하게 cluster를 하나 더 생성하고 kubeconfig file을 다운로드 받습니다.
현재 `~/.kube/config` 파일을 살펴보면 아래와 같이 하나의 클러스터에 대한 정보만 담겨져 있습니다.
여기서 저희는 새롭게 생성한 클러스터의 정보를 추가하여 여러개의 클러스터에 접근하도록 설정 가능합니다.

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: ...
    server: https://xxxxxxxx-nks-kr1.container.nhncloud.com:6443
  name: "toast-my-cluster-1"
contexts:
- context:
    cluster: "toast-my-cluster-1"
    user: admin
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: admin
  user:
    client-certificate-data: ...
    client-key-data: ...
```
위와 같이 설정된 정보를 아래와 같이 변경해주면 됩니다.
- clusters 프로퍼티에 새롭게 생성된 정보 추가
- contexts 프로퍼티에 새롭게 생성된 정보 추가 (name과 user 값을 클러스터 마다 달리 설정)
- users 프로터에 새롭게 추가된 user 추가

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: ...
    server: https://xxxxxxx-nks-kr1.container.nhncloud.com:6443
  name: "toast-my-cluster-1"
  - cluster:
    certificate-authority-data: ...
    server: https://xxxxxxx-nks-kr1.container.nhncloud.com:6443
  name: "toast-my-cluster-2"
contexts:
- context:
    cluster: "toast-my-cluster-1"
    user: cluster1-admin
  name: cluster1
- context:
    cluster: "toast-my-cluster-1"
    user: cluster2-admin
  name: cluster2
current-context: cluster1 # 설정 변경
kind: Config
preferences: {}
users:
- name: cluster1-admin
  user:
    client-certificate-data: ...
    client-key-data: ...
- name: cluster2-admin
  user:
    client-certificate-data: ...
    client-key-data: ...
```

만약 kubectl 명령어를 보낼 클러스터를 변경하고 싶다면, `current-context` 값을 변경하면 됩니다.

## Reference

- [kubectl](https://kubernetes.io/docs/reference/kubectl/)
- [configure-access-multiple-cluster](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)
- [organize-cluster-access-kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
