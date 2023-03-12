## 5.3 젠킨스 설치 및 설정하기



### 5.3.1 헬름으로 젠킨스 설치하기

- 젠킨스로 지속적 통합을 진행하는 과정에서 컨테이너 이미지를 레지스트리에 푸시하는 과정을 진행해야 한다.

- 헬름으로 설치되는 젠킨스는 파드에서 동작하는 어플리케이션이기 때문에, PV를 마운트하지 않으면 파드가 다시 시작될 때 내부 볼륨에 저장하는 모든데이터가 삭제되므로, 이를 방지하기 위해서 어플리케이션의 PV가 NFS를 통해 프로비저닝 될 수 있게 해야한다.
- 만들어진 디렉토리에 사용자 ID와 그룹 ID번호를 -n 옵션으로 확인할 수 있다.
  - `ls -n /nfs_shard`
- 젠킨스를 헬름 차트로 설치해 어플리케이션을 사용하게 되면 젠킨스의 여러 설정 파일과 구성 파일들이 PVC를 통해 PV에 파일로 저장된다.
  - 이때, PV에 적절한 접근ID를 부여하지 않으면 PVC를 사용해 파일을 읽고 쓰는데 문제가 발생할 수 있다.
  - 그러므로, 젠킨스 PV가 사용할 NFS 디렉토리에 대한 접근 ID(사용자ID, 그룹ID)를 설정 해줘야한다.
- 젠킨스는 사용자가 배포를 위해 생성한 내용과 사용자 계정 정보, 사용하는 플러그인과 같은 데이터를 저장하기 위해서 PV와 PVC의 구성을 필요로 한다.

#### 테인트와 톨러레이션

> 일반적으로 테인트와 톨러레이션은 혼합해서 사용한다.

테인트는 손에 잡기 싫은 것, 피하고 싶은 것, 가지 말았으면 하는 것을 의미한다. 톨러레이션은 참아내는 인내로 비유할 수 있다.

즉, 테인트는 **쉽게 접근하지 못하는 소중한 것으로 설정** 하고 톨러레이션은 **이를 접근하기 위한 특별한 키** 라고 생각할 수 있다.

| 값         | -                                                            |
| ---------- | ------------------------------------------------------------ |
| 테인트     | 키와 값 그리고 키와 값에 따른 효과의 조합을 통해 설정함으로서 노드 파드 배치 기준을 설정함. |
| 톨러레이션 | 키와 값, 효과에 더해져 연산자를 추가로 가지고 있다.          |

Effect(효과)는 테인트와 톨러레이션의 요소인 키 또는 값이 일치하지 파드가 노드에 스케줄 되려고 하는 경우 어떤 동작을 할 것인지를 나타낸다.

| 효과             | 테인트가 설정된 노드에 파드 신규 배치                  | 파드가 배치된 노드에 테인트 설정 |
| ---------------- | ------------------------------------------------------ | -------------------------------- |
| NoSchedule       | 노드에 파드 배치를 거부                                | 노드에 존재하는 파드 유지        |
| PreferNoSchedule | 다른 노드에 파드 배치가 불가능할 때는 노드에 파드 배치 | 노드에 존재하는 파드 유치        |
| NoExecute        | 노드에 파드 배치를 거부                                | 파드를 노드에서 제거             |

```sh
# Jenkins-install.sh

# 세션 유효시간 1440분
jkopt1="--sessionTimeout=1440"
# 세션 정리시간 86400초
jkopt2="--settionEviction=86400"
# 서울 시간대로 변경
jvopt1="-Duser.timezone=Asia/Seoul"
# 설정 값을 미리 입력해 둔 야플 파일을 깃허브 저장소에서 받아오기
jvopt2="-Dcas.jenkins.config=https://raw.githubusercontent.com/sysnet4admin/_Book_k8sInfra/main/ch5/5.3.1/jenkins-config.yaml"
# edu 차트 저장소의 jenkins 차트를 사용해 jenkins 릴리즈를 설치
helm install jenkins edu/jenkins \
# jenkins라는 이름의 PVC 사용
--set persistence.existingclaim=jenkins \
# 젠킨스 접속 시 사용할 관리자 비밀번호를 admin으로 설정
--set master.adminPassword=admin \
# 젠킨스의 컨트롤러 파드를 쿠버네티스 마스터 노드 m-k8s에 배치하도록 선택
--set master.nodeSelector."kubernetes\.io/hostname"=m-k8s \
# 톨러레이션 설정
--set master.tolerations[0].key=node-role.kubernetes.io/master \
--set master.tolerations[0].effect=NoSchedule \
--set master.tolerations[0].operator=Exists \
# 젠킨스를 구동하는 파드가 실행될 때 가질 유저ID와 그룹ID 설정
--set master.runAsUser=1000 \
--set master.runAsGroup=1000 \
# 젠킨스 버전 설정
--set master.tag=2.249.3-lts-centos7 \
# 차트로 생성되는 서비스의 타입을 로드밸런서로 설정(외부IP 받아오기)
--set master.serviceType=LoadBalancer \
# 젠킨스가 http 상에서 구동되도록 포트를 80으로 지정
--set master.servicePort=80 \
# 젠킨스에 필요한 설정을 지정 (위의 세션 시간들)
--set master.jenkinsOpts="$jkopt1 $jkopt2" \
# 젠킨스 실행환경에 적용되는 변수들
--set master.javaOpts="$jvopt1 $jvopt2"
```



### 5.3.2 젠킨스 살펴보기

젠킨스 컨트롤러가 단독으로 설치할 경우에는 컨트롤러가 설치된 서버에서 젠킨스 자체 시스템관리, CI/CD 설정, 빌드 등의 작업을 모두 젠킨스 컨트롤러 단일 노드에서 수행한다.

그러나, 컨트롤러-에이전트 구조로 설치할 경우 컨트롤러는 젠킨스 자체의 관리 및 CI/CD와 관련된 설정만을 담당하고 실제 빌드 작업은 에어전트로 설정된 노드에서 이루어진다.

즉, **컨트롤러 단독 설치는 일반적으로 간단한 테스트에서** 그리고 **컨트롤러-에이전트 구조를 주로 사용** 한다.



