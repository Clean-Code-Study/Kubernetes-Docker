# 4.4 쿠버네티스에서 직접 만든 컨테이너 사용하기

컨테이너 이미지를 빌드하는 방법을 알아야 하는 이유는 결과적으로 직접 만든 컨테이너 이비지를 쿠버네티스에서 활용하기 위해서이다.

쿠버네티스에서 이미지를 사용하려면 쿠버네티스가 이미지를 불러올 수 있는 공간에 이미지를 넣어 두어야 한다.

# 4.4.1 쿠버네티스에서 도커 이미지 구동하기

쿠버네티스는 컨테이너를 효과적으로 다루기 위해 만들어졌고, 컨테이너인 파드도 쉽게 부를 수 있다.

직접 만든 컨테이너 이미지도 kubectl 명령으로 쿠버네티스 클러스터에서 바로 구동할 수 있다.

```bash
kubectl create deployment failure1 --image=multistage-img
```

```bash
kubectl get pods -w

NAME                        READY   STATUS         RESTARTS   AGE
failure1-6dc55db9d4-jq4cg   0/1     ErrImagePull   0          17s
failure1-6dc55db9d4-jq4cg   0/1     ImagePullBackOff   0          19s
```

파드의 상태 및 변화를 확인하면, 이미지를 내려받는데 문제가 발생하여 `ErrImagePull`, `ImagePullBackOff` 오류가 번갈아 발생한다.

이는 이미지가 호스트에 존재함에도 기본 설정에 따라 이미지를 외부(도커 허브)에서 받으려고 시도하기 때문이다.

  

```bash
kubectl create deployment failure2 --dry-run=client -o yaml \
--image=multistage-img > failure2.yaml
```

내부에 존재하는 컨테이너 이미지를 사용하도록 설정하여 디플로이먼트를 생성한다.

사용자가 원하는 형태의 디플로이먼트를 만드는 가장 좋은 방법은 현재 수행되는 구문을 야믈 형태로 뽑아내는것이다.

해당 명령으로 야믈 파일을 만들어 바꿀 수 있다..

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: failure2
  name: failure2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: failure2
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: failure2
    spec:
      containers:
      - image: multistage-img
        imagePullPolicy: Never # 추가된 옵션. 호스트에 존재하는 이미지를 사용함
        name: multistage-img
        resources: {}
status: {}
```

```bash
kubectl apply -f failure2.yaml 
# deployment.apps/failure2 created

kubectl get pods

NAME                        READY   STATUS              RESTARTS   AGE
failure1-6dc55db9d4-jq4cg   0/1     ImagePullBackOff    0          8m1s
failure2-59bfb8b764-v6kd5   0/1     ErrImageNeverPull   0          6s
```

`ErrImageNeverPull` 가 발생하였다..

기존에 생성한 deployment를 지운다.

```bash
curl -O https://raw.githubusercontent.com/sysnet4admin/_Book_k8sInfra/main/ch4/4.3.4/Dockerfile
```

w3-k8s 워커 노드에서 아래 명령을 실행한다.

```bash
curl -O https://raw.githubusercontent.com/sysnet4admin/_Book_k8sInfra/main/ch4/4.3.4/Dockerfile
docker build -t multistage-img .
```

마스터 노드에서 failure2.yaml을 success1.yaml로 복사후 replicas를 3으로 failure2의 일므도 success1으로 바꾼다..

```bash
cp failure2.yaml success1.yaml
sed -i 's/replicas: 1/replicas: 3/' success1.yaml
sed -i 's/failure2/success1/' success1.yaml
```

```bash
kubectl apply -f success1.yaml

kubectl get pods -o wide

NAME                        READY   STATUS              RESTARTS   AGE   IP             NODE     NOMINATED NODE   READINESS GATES
success1-6fc588fdf4-5cgjp   0/1     ErrImageNeverPull   0          11s   172.16.77.3    k8s      <none>           <none>
success1-6fc588fdf4-85xq8   1/1     Running             0          11s   172.16.132.2   w3-k8s   <none>           <none>
success1-6fc588fdf4-sxl5f   1/1     Running             0          11s   172.16.132.3   w3-k8s   <none>           <none>
```

워커노드 3번만 성공한 이유는, 컨테이너 이미지가 워커노드 3번에만 존재하기 때문이다.

워커노드 1, 2 번에는 multistage-img가 없어 파드를 생성할 수 없다.

해결방법은 크게 2가지로,

1. 기본으로 사용하는 도커 허브에 multistage-img를 올려 다시 내려받기
2. 쿠버네티스 클러스터가 접근할 수 있는 곳에 이미지 레지스트리를 만들고 그곳에서 받아오도록 설정하는 방법이다.

# 4.4.2 레지스트리 구성하기

호스트에서 생성한 이미지를 쿠버네티스에서 사용하려면 모든 노드에서 공통으로 접근하는 레지스트리(저장소)가 필요하다.

도커나 쿠버네티스는 도커 허브라는 레지스트리에서 이미지르 ㄹ내려 받을 수 있어 인터넷이 연결되어 있다면 활용할 수 있다.

직접 만든 이미지가 외부에 공개되지 않기를 원하는 경우, 레지스트리를 직접 구축하는 방법이 있다. 이 경우에는 인터넷을 연결할 필요가 없어 보안이 중요한 내부 전상망에서도 구현이 가능하다.

- Quay(키)
- Harbor(하버)
- Nexus Repository
- Docker Registry

인증서를 만들어 배포한 뒤 레지스트리를 구동해야 한다. 인증서를 생성하려면 서명 요청서를 작성해야 하며, 서명 요청서에는 인증서를 생성하는 개인이나 기관의 정보와 인증서를 생성하는 데 필요한 몇가지 추가 정보를 기록한다.웹 서버에서 사용하는 인증서는 명령줄에서 직접 인증서를 생성하지만, 도커는 이미지를 올리거나 내려받기 위해 레지스트리에 접속하는 과정에서 주체 대체 이름(SAN, Subject Alternative Name)이라는 추가 정보를 검증하기 때문에 요청서에 추가 정보를 기입해 인증서를 생성하는 과정이 필요하다.

```bash
[req]
distinguished_name = private_registry_cert_req
x509_extensions = v3_req
prompt = no

[private_registry_cert_req]
C = KR
ST = SEOUL
L = SEOUL
O = gilbut
OU = Book_k8sInfra
CN = 192.168.1.10

[v3_req]
keyUsage = keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.0 = m-k8s
IP.0 = 192.168.1.10
```

```bash
[req]
distinguished_name = private_registry_cert_req
x509_extensions = v3_req
prompt = no

[private_registry_cert_req]
C = KR
ST = SEOUL
L = SEOUL
O = gilbut
OU = Book_k8sInfra
CN = 192.168.1.10

[v3_req]
keyUsage = keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.0 = m-k8s
IP.0 = 192.168.1.10
[root@m-k8s 4.4.2]# cat create-registry.sh 
#!/usr/bin/env bash
certs=/etc/docker/certs.d/192.168.1.10:8443
mkdir /registry-image
mkdir /etc/docker/certs
mkdir -p $certs
openssl req -x509 -config $(dirname "$0")/tls.csr -nodes -newkey rsa:4096 \
-keyout tls.key -out tls.crt -days 365 -extensions v3_req

yum install sshpass -y
for i in {1..3}
  do
    sshpass -p vagrant ssh -o StrictHostKeyChecking=no root@192.168.1.10$i mkdir -p $certs
    sshpass -p vagrant scp tls.crt 192.168.1.10$i:$certs
  done
  
cp tls.crt $certs
mv tls.* /etc/docker/certs

docker run -d \
  --restart=always \
  --name registry \
  -v /etc/docker/certs:/docker-in-certs:ro \
  -v /registry-image:/var/lib/registry \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/docker-in-certs/tls.crt \
  -e REGISTRY_HTTP_TLS_KEY=/docker-in-certs/tls.key \
  -p 8443:443 \
  registry:2
```

```bash
./create-registry.sh
```

```bash
docker ps -f name=registry

CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                             NAMES
0e44aa82678a        registry:2          "/entrypoint.sh /etc…"   30 seconds ago      Up 29 seconds       5000/tcp, 0.0.0.0:8443->443/tcp   registry
```

사설 도커 레지스트리에 등록할 수 있게 컨테이너 이미지의 이름을 변경한다.

multistage:latest 이미지를 레지스트리에서 읽으려면 레지스트리가 서비스되는 주소와 제공되는 이미지 이름을 레지스트리에 등록될 이름으로 지정해야 한다.

```bash
docker tag multistage-img 192.169.1.10:8443/multistage-img
docker images  192.169.1.10:8443/multistage-img

REPOSITORY                         TAG                 IMAGE ID            CREATED             SIZE
192.169.1.10:8443/multistage-img   latest              5dcef45eff3c        36 minutes ago      148MB
```

```bash
docker push 192.169.1.10:8443/multistage-img

The push refers to repository [192.169.1.10:8443/multistage-img]
```

```bash
curl https://192.169.1.10:8443/v2/_catalog -k
```

# 4.4.3 직접 만든 이미지로 컨테이너 구동하기

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: success2
  name: success2
spec:
  replicas: 3
  selector:
    matchLabels:
      app: success2
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: success2
    spec:
      containers:
      - image: 192.168.1.10:8443/multistage-img
        name: multistage-img
        resources: {}
status: {}
```

```yaml
kubectl apply -f success2.yaml
```