## 3.4 알아두면 쓸모 있는 쿠버네티스 오브젝트

> 디플로이먼트 외에도 용도에 따라 사용할 수 있는 다양한 오브젝트가 있다. 예를 들면 데몬셋, 컨피드맵, PV, PVC, 스테이트풀셋 등이 있다.



### 3.4.1 데몬셋

데몬셋은 디플로이먼트의 replicas가 노드 수만큼 정해져 있는 형태라 할 수 있는데, **노드 하나당 파드 한 개만을 생성** 한다.

데몬셋은 Calico 네트워크 플러그인, kube-proxy, MetalLB 스피커 등 기존에도 많이 사용했다. 이들의 공통점은 **노드의 단일 접속 지점으로 노드 외부와 통신하는 것이다.** 즉 이는 파드가 1개 이상 필요하지 않으며 노드를 관리하는 파드라면 데몬셋으로 만드는게 효율적이다.

> ```sh
> kubectl get pods -n metallb-system -o wide -w
> ```
>
> -n: namespace 옵션으로, 위의 명령은 metallb-system 네임스페이스의 파드를 조회하겠다는 의미.
>
> -w: watch 옵션으로, 오브젝트 상태를 감시하여 변화를 감지하면 그 변화를 출력한다. 리눅스의 tail -f 옵션과 비슷하다.



### 3.4.2 컨피그맵

컨피그맵(config map)은 이름 그대로 **설정을 목적으로** 사용하는 오브젝트이다.

디플로이먼트를 생성한 후, 컨피그맵을 설정(`kubectl expose deployment cfgmap --type=LoadBalancer --name=cfgmap-xvc --port=80`)한다.

![image-20230206224117666](/Users/user/Library/Application Support/typora-user-images/image-20230206224117666.png)

이후, 그와 관련한 모든 파드를 삭제하면, kubelet에서 해당 파드를 자동으로 생성한다.

![image-20230206224255590](/Users/user/Library/Application Support/typora-user-images/image-20230206224255590.png)

###### - 삭제하기전 파드 상태

![image-20230206224320959](/Users/user/Library/Application Support/typora-user-images/image-20230206224320959.png)

###### - 삭제하고 다시 생성된 파드 상태



### 3.4.3 PV와 PVC

> 파드는 언제라도 생성되고 지워지지만, 때때로 파드에서 생성한 내용을 기록하고 보관하거나 모든 파드가 동일한 설정값을 유지하고 관리하기 위해 공유된 볼륨으로부터 공통된 설정을 가지고 올라올 수 있도록 설계해야 할 때도 있다.
>
> 쿠버네티스는 이런 경우를 위해 다음과 같은 목적으로 다양한 형태의 볼륨을 제공한다.
>
> - 임시: `emptyDir`
> - 로컬: `host Path`, `local`
> - 원격: `persistentVolumeClaim` , `cephfs` , `cinder` , `csi` , `fc(fibre channel)` , `flexVolume` , `flocker` , `glusterfs` , `iscsi` , `nfs` , `portwworxVolume` , `quobyte` , `rbd` , `scaleIO` , `stoageos` , `vsphereVolume`
> - 특수목적: `downwardAPI` , `configMap` , `secret` `azureFile` , `projected`
> - 클라우드: `awsElasticBlock` , `azureDisk` , `gcdPersitentDisk`

쿠버네티스는 필요할 때 `PVC(PersistnetVolumeClaim, 지속적으로 사용 가능한 볼륨 요청)` 를 요청해 사용한다. 이 PVC를 사용하려면, `PV(PersistentVolume, 지속적으로 사용 가능한 볼륨)`  로 볼륨을 선언해야 한다.

> 즉, PV는 볼륨을 사용할 수 있게 준비하는 단계이고(요리사가 피자를 굽는 것이고) 그리고 PVC는 준비된 볼륨에서 일정 공간을 할당받는 것(손님이 원하는 만큼 피자를 접시에 담아오는 것)이다.

PVC와 PV의 구성은 거의 동일하지만, **PV는 사용자가 요청할 볼륨 공간을 관리자가 만들고** , **PVC는 사용자(개발자)간 볼륨을 요청하는 데 사용하다는 점** 에서 차이가 있다.



### 3.4.4 스테이프풀셋

> 지금까지는 파드가 replicas에 선언된 만큼 무작위로 생성될 뿐이었다. 하지만 파드가 만들어지는 이름과 순서를 예측해야 할 때가 있는데 주로 레디스(redis), 주키퍼(zookeeper), 카산드라(cassandra), 몽고DB(mongoDB), 등의 마스터-슬레이브 구조 시스템에서 필요하다.

스테이트풀셋(statefule set)은 volumeClaimTemplates 기능을 사용해 PVC를 자동으로 생성할 수 있고, 각 파드가 순서대로 생성되기 때문에 *고정된 이름* , *볼륨* , *설정* 등을 가질 수 있다.

다만, 효율성 면에서 좋은 구조가 아니므로 요구 사항에 맞게 적절히 사용하는 것이 좋다.

>  참고로, 스테이트풀셋은 디플로이먼트와 형제나 다름없는 구조라 디플로이먼트에서 오브젝트 종류를 변경하면 바로 사용가능하다.
>
> 또한, 스테이트풀셋에서 expose 명령어는 지원하지 않는다. (expose 명령어는 디플로이먼트, 파드, 레플리카셋, 레플리케이션 컨트롤러에서 지원한다.)







