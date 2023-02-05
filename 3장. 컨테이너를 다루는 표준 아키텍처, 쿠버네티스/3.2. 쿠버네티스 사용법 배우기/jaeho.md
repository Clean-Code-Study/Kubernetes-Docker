## 쿠버네티스 기본 사용법 배우기



### 3.2.1 파드를 생성하는 방법

> 쿠버네티스를 사용한다는 것 = 사용자에게 효과적으로 파드를 제공한다는 것



`kubectl run` vs `kubectl create`

- kubectl run
  - run 명령을 이용해서 파드를 생성하면 단일 파드 1개만 생성되고 관리됨
- kubectl create
  - create는 deployment를 추가해서 실행해야 한다. (`kubectl create deployment`)
  - create deployment를 사용해서 파드를 생성하면 **디플로이먼트** 라는 관리 그룹내에서 파드가 생성됨

즉, run으로 파드를 생성하면 초코파이 1개 / create deployment로 파드를 생성하면 초코파이 상자내의 초코파이 1개와 같음.

> 파드를 생성하고, kubectl get pod 혹은 kubectl get pods를 하면 생성된 파드 조회가 가능.
>
> 거기에, wide 옵션을 주면 파드의 IP 정보까지 출력해서 보여준다.
>
> ![image-20230202223302300](/Users/user/Library/Application Support/typora-user-images/image-20230202223302300.png)

최근에는 create deployment를 이용해 파드를 생성하며, run을 사용할 경우 deprecated를 표시한다. 

실무에서는 create deployment를 사용하는 것이 권고되며 간단한 테스트일 때, run을 사용할 수 있다.



### 3.2.2 오브젝트란

> 파드와 디플로이먼트는 **스펙** 과 **상태** 등의 값을 가지고 있다.
>
> 이러한 값을 가지고 있는 파드와 디플로이먼트를 개별 속성에 포함해 부르는 단위를 **오브젝트** 라고 한다.



#### 기본 오브젝트

1. 파드
   - 쿠버네티스에서 실행되는 최소 단위. (= 즉, 웹 서비스를 구동하는데 필요한 최소 단위)
   - 독립적인 공간과 사용 가능한 IP를 가지고 있다.
   - 하나의 파드는 1개 이상의 컨테이너를 가지고 있어, 컨테이너(기능)을 묶어 하나의 목적처럼 사용 가능(MSA 구조)
   - 범용으로 사용할 때는 개부분 1개의 파드에 1개의 컨테이너를 적용한다.
1. 네임스페이스
   - 쿠버네티스 클러스터에서 사용되는 리소스들을 구분해 관리하는 그룹.
   - 특별히 지정하지 않으면 default, kube-system, metallb-system 가 있다.
     - default: 기본으로 할당.
     - kube-system: 쿠버네티스 시스템에서 사용.
     - metallb-system: 온프레미스에서 쿠버네티스를 사용할 경우 외부에서 쿠버네티스 클러스터 내부로 접속하게 도와주는 컨테이너 속해있음.
1. 볼륨
   - 파드가 생성될 때, 파드에서 사용할 수 있는 디렉토리 제공.
   - 기본적으로 파드는 영속되는 개념이 아니라서 제공되는 디렉터리도 임시로 사용함.
   - 하지만 파드가 사라지더라도 저장과 보존이 가능한 디렉터리 볼륨 오브젝트를 통해 생성하고 사용가능함.
1. 서비스
   - 파드는 클러스터 내에서 유동적이기 때문에 접속 정보가 고정일 수 없다. (따라서, 파드 접속을 안정적으로 유지하도록 서비스를 통해 내/외부로 연결된다)
   - 그래서, 서비스는 새로 파드가 생성될 때 부여되는 새로운 IP를 기존에 제공하던 기능과 연결해 준다.
   - 기존 인프라에서 로드밸런서, 게이트웨이와 비슷한 역할을 한다.



#### 디플로이먼트

기본 오브젝트만으로도 쿠버네티스를 사용할 수 있으나, 좀 더 효율적으로 작동하도록 기능들을 조합하고 추가해 구현한 것이 **디플로이먼트** 이다.

> 그 외에도, 데몬셋, 컨피그맵, 레플리카셋, PV(persistent volume), PVC(persistent volume claim), 스테이스풀셋이 있다.

디플로이먼트 오브젝트는 파드에 기반을 두고 있으며, 레플리카셋 오브젝트를 합쳐놓은 형태이다.

> 참고로, 레플리케이션 컨트롤러가 발전한 형태는 레플리카셋이며 현재는 레플리카셋만 알면 된다.

> 디플로이먼트로 생성한 파드를 삭제하려면, `kubectl delete deployment ${NAME}` 을 사용하면 된다.



### 3.2.3 레플리카셋으로 파드 수 관리하기

> 레플리카셋 오브젝트는 쿠버네티스 안에서 다수의 파드를 만들게 도와준다.

`scale` 와 `replicas` 옵션을 추가하여 기존 디플로이먼트로 생성한 파드의 수를 증가시킬 수 있다.

```sh
kubectl scale deployment ${name} --replicas=3
```

![image-20230202234734400](/Users/user/Library/Application Support/typora-user-images/image-20230202234734400.png)

>만약 디플로이먼트로 생성되지 않은 파드(run으로 생성된)를 레플리카셋으로 증가시키면 오류를 뱉는다.
>
>![image-20230202234832152](/Users/user/Library/Application Support/typora-user-images/image-20230202234832152.png)



### 3.2.4 스펙을 지정해 오브젝트 생성하기

디플로이먼트를 생성하면서 한꺼번에 여러개의 파드를 생성하려면 파일에서 설정해주어야 한다. 이러한 파일을 **오브젝트 스펙** 이라고 한다.

> create deployment에서는 replicas옵션을 사용할 수 없으며, scale은 이미 만들어진 디플로이먼트에서만 사용할 수 있다.

오브젝트 스펙은 일반적으로 YAML 문법으로 작성한다.

```yaml
apiVersion: apps/v1	# API 버전
kind: Deployment	# 오브젝트 종류
metadata:
  name: echo-hname
  labels:
    app: nginx
spec:
  replicas: 3	# 몇 개의 파드를 생성할지 결정
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: echo-hname
        image: sysnet4admin/echo-hname	# 사용되는 이미지
```

위 예제 파일에서 spec.replicas 에서 몇 개의 파드를 생성할 지 결정한다.

> apiVersion이라는 설정값이 있는데, 쿠버네티스에서 사용가능한 API 버전을 명시하는 것이다.
>
> `kubectl api-versions` 명령을 통해 확인 할 수 있다.
>
> ![image-20230202235821900](/Users/user/Library/Application Support/typora-user-images/image-20230202235821900.png)

추가적으로 해당 예제 파일을 분석해보면,

- METADATA
  - name: 디플로이먼트의 이름
  - labels: 디플로이먼트의 레이블
- SPEC
  - replicas: 레플리카셋을 몇 개 생성할지를 결정
  - selector 셀렉터의 레이블 지정
  - template.metadata: 템플릿의 레이블 지정
  - template.spec: 템플릿에서 사용할 컨테이너 이미지 지정

설정한 yaml 파일을 이용해 디플로이먼트를 생성하려면, `kubectl create -f ${yaml 경로}` 명령을 입력하면 된다.



### 3.2.5 apply로 오브젝트 생성하고 관리하기

create로 디플로이먼트를 생성하면 파일의 변경 사항을 바로 적용할 수 없다는 단점이 있다.

이런 경우를 위해 쿠버네티스는 apply라는 명령어를 제공한다.

```sh
kubectl apply -f ${yaml 경로}
```

![image-20230203003826009](/Users/user/Library/Application Support/typora-user-images/image-20230203003826009.png)

warning이 발생한 이유는, 오브젝트를 처음부터 apply로 생성한 것이 아니기에 발생한 것이다.

> 경고가 떠도 작동에는 문제가 없지만, 일관성에 문제가 생길 수 있다. 그렇기에, 변경사항이 발생할 가능성이 있는 오브젝트는 처음부터 apply로 생성하는 것이 좋다.



###### 오브젝트 생성 명령어 비교

| 구분        | Run       | Create    | Apply         |
| ----------- | --------- | --------- | ------------- |
| 명령 실행   | 제한적    | 가능함    | 안 됨         |
| 파일 실행   | 안 됨     | 가능함    | 가능함        |
| 변경 가능   | 안 됨     | 안 됨     | 가능함        |
| 실행 편의성 | 매우 좋음 | 매우 좋음 | 좋음          |
| 기능 유지   | 제한적    | 지원됨    | 다양하게 지원 |



### 3.2.6 파드의 컨테이너 자동 복구 방법

쿠버네티스는 거의 모든 부분이 **자동 복구** 되도록 설계되었고, 이러한 파드의 자동 복구 기술을 **셀프 힐링** 이라고 한다.

이는 제대로 작동하지 않는 컨테이너를 다시 시작하거나 교체해 파드가 정상적으로 작동하게 한다.

> kubectl exec 명령을 통해 파드 컨테이너의 셀(shell)에 접속이 가능하다.
>
> -it 옵션을 주게되면, stdin(standard in) + t (teletypewriter)를 이용하여 표준 입력을 명령줄 인터페이스로 작성할 수 있다.
>
> `kubectl exec -it nginx-pod -- /bin/bash`



### 3.2.7 파드의 동작 보증 기능

> 쿠버네티스는 파드 자체에 문제가 발생하면 파드를 자동 복구해서 파드가 항상 동작하도록 보장하는 기능도 있다.

- 쿠버네티스는 디플로이먼트에 속한 파드가 아닌 경우, 어떠한 컨트롤러도 이 파드를 관리하지 않는다. 즉, 삭제된다면 다시 생성되지도 않는 것이다.

- 반면, 디플로이먼트에 속한 파드인 경우, 해당 디플로이먼트에 선언한 레플리카 수를 유지하려고 하기 때문에 파드의 수를 하상 확인하고 부족하면 새로운 파드를 만들어 낸다.
  - 감시 → 차이 발견 → 상태 변경 → 변경 완료 후 감시



### 3.2.8 노드 자원 보호하기

> 노드는 쿠버네티스 스케줄러에서 파드를 할당받고 처리하는 역할을 한다.

> 각 파드의 내용값을 선택적으로 보려면 custom-columns 옵션을 이용할 수 있다.
>
> `kubectl get pods -o=custom-columns=NAME:.metadata.name,IP:.status.podID, STATUS:.status.phase, NODE:.spec.nodeName`
>
> ![image-20230204210403579](/Users/user/Library/Application Support/typora-user-images/image-20230204210403579.png)

어떠한 노드에 문제가 생겼을 때, `cordon` 명령을 통해 해당 노드의 현재 상태를 보존하고, 더 이상 스케줄되지 않는 상태로 만든다.

![image-20230204211113074](/Users/user/Library/Application Support/typora-user-images/image-20230204211113074.png)

###### - 여기서는 w1-k8s 노드에 cordon 명령을 적용

파드가 할당되지 않도록 설정했던 것을 해제하려면 `uncordon` 명령을 사용하면 된다.

```sh
kubelctl uncordon w1-k8s
```



### 3.2.9 노드 유지보수하기

쿠버네티스를 운영하다 보면, 정기 또는 비정기적인 유지보수를 위해 노드를 꺼야하는 상황이 발생하기도 하는데, 이럴때 `drain` 기능을 사용할 수 있다.

**drain은 지정된 노드의 파드를 전부 다른 곳으로 이동시켜** 해당 노드를 유지보수할 수 있게 한다.

> `kubectl drain ${노드명}` 명령을 통해 파드를 옮기면 아래와 같은 이슈가 발생할 수 있다.
>
> ![image-20230204211943424](/Users/user/Library/Application Support/typora-user-images/image-20230204211943424.png)
>
> 이는 DeamonSet이 각 노드에 1개만 존재하는 파드라서 drain으로는 삭제할 수 없다고 나오는 것이다.
>
> 이 에러를 무시하고 drain 하려면, ignore-daemonsets 옵션을 추가해서 사용하면 된다.
>
> ```sh
> kubectl drain w1-k8s --ignore-darmonsets
> ```

drain은 실제로 파드를 옮기는 것이 아닌, 노드에서 파드를 삭제하고 다른 곳에 다시 생성하는 방식이다. 이는 언제라도 파드를 삭제할수 있는 쿠버네티스의 특징으며, 이 쿠버네티스에서 대부분 이동은 파드를 지우고 다시 만드는 과정을 의미한다.

이 drain을 완료하면, 해당 노드는 SchedulingDisabled 상태가 추가 된다. (이는 uncordon 명령을 통해 스케줄 될 수 있는 상태로 변경가능.)



### 3.2.10 파드 업데이트하고 복구하기



#### 파드 업데이트

>  파드를 배포할 때 --record 옵션을 사용할 수 있다. 이는 배포한 정보의 히스토리를 기록하는 옵션이다. (매우 중요한 옵션❗️)
>
> ```sh
> kubectl apply -f ${yaml 파일} --record
> ```
>
> record 옵션으로 기록된 히스토리는 rollout history  명령을 실행해 확인할 수 있다.
>
> ```sh
> kubectl rollout history deployment rollout-nginx
> ```
>
> ![image-20230204213415250](/Users/user/Library/Application Support/typora-user-images/image-20230204213415250.png)

>파드 컨테이너의 이미지 업데이트를 하고 싶다면 `set image` 명령으로 특정 이미지 버전으로 변경 가능하다.
>
>```sh 
>kubectl set image deployment ${디플로이먼트 이름} ${컨테이너 이름}: ${원하는 버전}
>```

파드를 업데이트하면, 파드의 이름과 IP가 변경됨을 알 수 있다. 여러 번 언급되었듯이 파드는 언제라도 지우고 다시 만들수 있는 특성이 있기 때문이다.

**즉, 컨테이너를 업데이트하는 가장 쉬운 방법은 파드를 관리하는 레플리카의 수를 줄이고 늘려 파드를 새로 생성하는 것이다.**

이때 중요한 것은, 파드를 모두 한 번에 지우느 것이 아니라 파드를 하나씩 순차적으로 지우고 생성한다. (파드수가 많으면 다수의 파드가 업데이트 된다. 기본 업데이트 전체의 25% 이며 최솟값은 1이다)

- 시작 → 새로 생성 중 → 파드 1개 삭제 → 새로 생성 중 → 파드 1개 삭제 → 새로 생성 중 → 파드 1개 삭제 → 업데이트 완료

![image-20230204215247434](/Users/user/Library/Application Support/typora-user-images/image-20230204215247434.png)

###### - history 명령을 통해 업데이트 확인



#### 업데이트 실패 시 파드 복구하기

파드를 업데이트하는 도중, 의도치 않게 잘못된 이미지로 업데이트하는 경우 `ImagePullBackOff` 상태가 나타날 수 있다.

> 어떤 문제인지 확인하는 방법으로, rollout status 명령어를 활용할 수 있다.
>
> ![image-20230205170212265](/Users/user/Library/Application Support/typora-user-images/image-20230205170212265.png)
>
> 계속해서 시도하지만, 결국 생성되지 않은 메세지를 출력한다.
>
> ![image-20230205170307510](/Users/user/Library/Application Support/typora-user-images/image-20230205170307510.png)
>
> describe 명령을 이용하면 문제점을 좀 더 정확히 파악가능하다.
>
> ![image-20230205170409970](/Users/user/Library/Application Support/typora-user-images/image-20230205170409970.png)
>
> 마지막 이벤트로 replica set이 scaled up 되던 상태에서 멈춰 있다.
>
> 이와 같은 이슈는 충분히 실수로 발생할 수 있다. 그렇기에 이를 방지하고자 업데이트 할 때 rollout을 사용하고 --record로 기록하는 것이다.

이러한 이슈로 인해 정상적인 상태로 다시 복구해야 하는 상황으로 이어진다. 먼저, 업데이트할 때 사용했던 명령들을 rollout history로 확인한다.

```sh
kubectl rollout history deployment ${파드 이름}
```

![image-20230205170756682](/Users/user/Library/Application Support/typora-user-images/image-20230205170756682.png)

REVISION 상태를 보고 우리는 `kubectl rollout undo deployment rollout-nginx` 명령을 통해 바로 이전 상태로 돌아간다.

![image-20230205170953104](/Users/user/Library/Application Support/typora-user-images/image-20230205170953104.png)

rollout undo 명령도 함께 history에 남기 때문에, 아래와 같이 REVISION 이 변경된다.

![image-20230205171107017](/Users/user/Library/Application Support/typora-user-images/image-20230205171107017.png)

![image-20230205171204367](/Users/user/Library/Application Support/typora-user-images/image-20230205171204367.png)

###### - rollout status 가 정상으로 나오는 모습

만약, 바로 전 상태가 아닌 특정 시점으로 돌아가고싶다면, --to-revision 옵션을 사용하면 된다.

```sh
kubectl rollout undo deployment rollout-nginx --to-revision=1
```

