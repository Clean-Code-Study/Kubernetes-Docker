## 4가지 방법으로 컨테이너 이미지 만들기

> 컨테이너 인프라 환경을 구성할 때, 이미 제공된 이미지를 사용하는 경우도 있지만, 직접 만든 어플리케이션으로 컨테이너를 만들 수도 있다.
>
> 기본적인 빌드 / 용량 줄이기 / 컨테이너 내부 빌드 / 멀티 스테이지



### 4.3.1 기본 방법으로 빌드하기

> 스프링 부트를 이용해 만든 자바 소스 코드로 이미지를 빌드해보는 과정은 아래와 같다.
>
> 자바 소스 빌드 → 도커파일 작성 → 도커파일 빌드 → 빌드 완료

```dockerfile
FROM openjdk:9
LABEL description="Exho IP Java Application"
EXPOSE 60431
COPY ./target/app-in-host.jar /opt/app-in-image.jar
WORKDIR /opt
ENTRYPOINT ["java", "-jar", "app-in-iamge.jar"
```

###### - 도커 파일 내용

```bash
import openjdk:8
Label_desc="Echo IP Java Application"	# 컨테이너 이미지 설명
EXPOSE=60431	# 60431포트를 사용해 오픈하도록 설명을 넣음
scp <HOST>/target/app-in-host.jar <Image>/opt/app-in-image.jar
cd /opt
./java -jar app-in-image.jar
```

###### - 배시 명령으로 유사 해석

1. `FROM <이미지 이름>:[태그]`
   - 이미지를 가져온다.
   - 가져온 이미지 내부에서 컨테이너 이미지를 빌드한다. (누군가 만들어 놓은 이미지에 필요한 부분을 추가가는 것이라 생각하면 된다)
   - 기초 이미지로 어떤 것을 선택하냐에 따라 다양한 환경의 컨테이너를 빌드할 수 있다.
1. `LABEL <레이블 이름>=<값>`
   - 이미지에 부가적인 설명을 위한 레이블을 추가할 때 사용
1. `EXPOSE <숫자>`
   - 생성된 이미지로 컨테이너를 구동할 때 어떤 포트를 사용하는지 알려준다
   - 이 명령을 사용한다고, 컨테이너를 구동할 때 자동으로 해당 포트를 호스트 포트로 연결하지는 않는다.
   - 외부와 연결하려면 지정한 포트를 호스트 포트와 연결해야 한다는 정보만 제공할 뿐이다
   - 실제로 외부에서 접속하여면 docker run 명령으로 이미지를 컨테이너로 빌드할 때, 반드시 -p 옵션을 넣어 포트를 연결해야 한다.
1. `COPY <호스트 경로> <컨테이너 경로>`
   - 호스트에서 새로 생성하는 컨테이너 이미지로 필요한 파일을 복사한다.
1. `WORKDIR <경로>`
   - 이미지의 현재 작업 위치를 opt로 변경한다.
1. `ENTRYPOINT [명령어, 옵션 ... 옵션]`
   - 컨테이너 구동 시, ENTRYPOINT 뒤에 나오는 대괄호 안의 든 명령을 실행한다.
   - 콤마로 구분된 문자열 중, 첫 번째 나오는 문자열은 실행할 명령어고 이후 두 번째 부터는 명령어를 실행할 때 추가하는 옵션이다.



### 4.3.2 컨테이너 용량 줄이기

> 불필요한 공간을 점유하는 비용 낭비이기도 하지만, 성능에 영향을 미칠 수 있다.

기초 이미지가 GCR(google container registry)에서 제공하는 distroless로 변경된다. (기존엔, openjdk)

> distroless 이미지란?
>
> 패키지 매니저나 쉘 등 보통 리눅스 배포판에서 사용하는 소프트웨어들이 포함되지 않은 베이스 이미지를 뜻함.
>
> 이 이미지는 실제 어플리케이션과 의존성 패키지들을 조금 더 가볍게 패키징 할 수 있다.

```dockerfile
FROM gcr.io/distroless/java:8
LABEL description="Exho IP Java Application"
EXPOSE 60431
COPY ./target/app-in-host.jar /opt/app-in-image.jar
WORKDIR /opt
ENTRYPOINT ["java", "-jar", "app-in-iamge.jar"
```

###### - 변경한 도커 파일

기존 dockerfile과는 FROM부에서 차이가 난다.



### 4.3.3 컨테이너 내부에서 컨테이너 빌드하기

```dockerfile
FROM openjdk:8
LABEL description="Exho IP Java Application"
EXPOSE 60431
RUN git.clone https://github.com/iac-source/inbuilder.git	# RUN 으로 이미지 내부에서 소스 코드 실행
WORKDIR inbuilder # git clone으로 내려받은 디렉터리를 현재 작업 공간으로 설정
RUN chmod 700 mvnw	# mvnw에 실행 권한 설정
RUN ./mvnw clean package	# 메이븐 래퍼로 JAR 빌드
RUN mv target/app-in-host.jar /opt/app-in-image.jar # 빌드된 JAR을 /opt/app-in-image.jar로 옮김
WORKDIR /opt
ENTRYPOINT ["java", "-jar", "app-in-iamge.jar"
```

###### - 모든 작업 명령을 dockerfile로 옮긴 모습

이렇게 될 경우, 가장 큰 컨테이너 이미지를 얻는다. 컨테이너 이미지는 커지면 커질수록 비효율적으로 동작할 수 밖에 없다.

즉, 이 방법은 좋지 않은 방법이지만 Dockerfile 하나로 빌드하면 컨테이너가 바로 생성되는 편리함은 놓치지 쉽지 않다.



### 4.3.4 최적화해 컨테이너 빌드하기

> 멀티 스페이지 빌드(Multi-stage build)는 최종 이미지의 용량을 줄일 수 있고, 호스트에 어떠한 빌드 도구도 설치할 필요없다.
>
> 그러나 멀티 스테이지는 docker-cd 17.06 버전부터 지원한다.

멀티 스테이지의 핵심은, 빌드하는 위치와 최종 이미지를 분리하는 것이다. 이렇게 하면, 최종 이미지는 빌드된 JAR를 가지고 있지만, 용량은 줄일 수 있다.



---

