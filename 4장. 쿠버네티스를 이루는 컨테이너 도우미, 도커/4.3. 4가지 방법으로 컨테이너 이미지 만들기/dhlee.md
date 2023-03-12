# 4.3 4가지 방법으로 컨테이너 이미지 만들기

- 기본적인 빌드
- 용량 줄이기
- 컨테이너 내부 빌드
- 멀티스테이지

# 4.3.1 기본 방법으로 빌드하기

- openjdk 설치
    
    ```bash
    yum install java-1.8.0-openjdk-devel -y
    ```
    
- 메이븐 빌드
    
    ```bash
    chmod 700 mvnw
    ./mvnw clean package
    ```
    
- 도커 빌드
    
    ```bash
    docker build -t basic-img .
    ```
    
    ```docker
    FROM openjdk:8
    LABEL description="Echo IP Java Application"
    EXPOSE 60431
    COPY ./target/app-in-host.jar /opt/app-in-image.jar
    WORKDIR /opt
    ENTRYPOINT [ "java", "-jar", "app-in-image.jar" ]
    ```
    
    ```bash
    docker images basic-img
    
    REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
    basic-img           latest              0945ce68cc79        About a minute ago   544 MB
    ```
    
    ```bash
    docker build -t basic-img:1.0 -t basic-img2.0 .
    
    Sending build context to Docker daemon  17.7 MB
    Step 1/6 : FROM openjdk:8
     ---> b273004037cc
    Step 2/6 : LABEL description "Echo IP Java Application"
     ---> Using cache
     ---> 55763fd614a9
    Step 3/6 : EXPOSE 60431
     ---> Using cache
     ---> 288aa74cefc8
    Step 4/6 : COPY ./target/app-in-host.jar /opt/app-in-image.jar
     ---> Using cache
     ---> 146726821a13
    Step 5/6 : WORKDIR /opt
     ---> Using cache
     ---> 1fbae7ec4afa
    Step 6/6 : ENTRYPOINT java -jar app-in-image.jar
     ---> Using cache
     ---> 0945ce68cc79
    ```
    
    캐시를 활용하여 빠르게 빌드됨
    
    ```bash
    sed -i 's/Application/Development/' Dockerfile
    docker build -t basic-img:3.0 .
    ```
    
    `Application ****`부분을 `Development` 로 변경하고 다시 빌드
    
    ```bash
    Sending build context to Docker daemon  17.7 MB
    Step 1/6 : FROM openjdk:8
     ---> b273004037cc
    Step 2/6 : LABEL description "Echo IP Java Development"
     ---> Running in 46da5340115d
     ---> 52913db623c3
    Removing intermediate container 46da5340115d
    Step 3/6 : EXPOSE 60431
     ---> Running in b920615da769
     ---> becbf3cf9c6d
    Removing intermediate container b920615da769
    Step 4/6 : COPY ./target/app-in-host.jar /opt/app-in-image.jar
     ---> 287c4f908d77
    Removing intermediate container feb993c2df7a
    Step 5/6 : WORKDIR /opt
     ---> 2196d22543d8
    Removing intermediate container 8c113c242da2
    Step 6/6 : ENTRYPOINT java -jar app-in-image.jar
     ---> Running in 58c1c19892e0
     ---> 99605a5cdf77
    Removing intermediate container 58c1c19892e0
    Successfully built 99605a5cdf77
    ```
    
    ```bash
    docker images basic-img
    
    REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
    basic-img           3.0                 99605a5cdf77        About a minute ago   544 MB
    basic-img           1.0                 0945ce68cc79        6 minutes ago        544 MB
    basic-img           latest              0945ce68cc79        6 minutes ago        544 MB
    ```
    

# 4.3.2 컨테이너 용량 줄이기

컨테이너 이미지의 용량을 줄여 빌드할 수 있다.

```bash
FROM gcr.io/distroless/java:8
LABEL description="Echo IP Java Application"
EXPOSE 60432
COPY ./target/app-in-host.jar /opt/app-in-image.jar
WORKDIR /opt
ENTRYPOINT [ "java", "-jar", "app-in-image.jar" ]
```

- [gcr.io/distroless/java:8](http://gcr.io/distroless/java:8) Gcr에서 제공하는 경량
    
    ```bash
    docker images | head -n 3
    
    REPOSITORY                            TAG                 IMAGE ID            CREATED             SIZE
    optimal-img                           latest              022f0e680ebd        32 seconds ago      148 MB
    basic-img                             3.0                 99605a5cdf77        7 minutes ago       544 MB
    ```
    
- 300MB 정도 용량이 작음

# 4.3.3 컨테이너 내부에서 컨테이너 빌드하기

```docker
FROM openjdk:8
LABEL description="Echo IP Java Application"
EXPOSE 60433
RUN git clone https://github.com/iac-source/inbuilder.git
WORKDIR inbuilder
RUN chmod 700 mvnw
RUN ./mvnw clean package
RUN mv target/app-in-host.jar /opt/app-in-image.jar
WORKDIR /opt
ENTRYPOINT [ "java", "-jar", "app-in-image.jar" ]
```

```bash
docker build -t nohost-img .
```

```bash
docker images | head -n 4

REPOSITORY                            TAG                 IMAGE ID            CREATED             SIZE
nohost-img                            latest              0eb3f8bc866e        26 seconds ago      633 MB
optimal-img                           latest              022f0e680ebd        5 minutes ago       148 MB
basic-img                             3.0                 99605a5cdf77        12 minutes ago      544 MB
```

새로 생성된 nohost-img가 용량이 가장 큰데, 컨테이너 내부에서 빌드를 진하기 때문에 빌드 중간에 생성한 파일들과 내려받은 라이브러리 캐시들이 최종 이미지인 nohost-img에 그대로 남기 때문이다.

# 4.3.4 최적화해 컨테이너 빌드하기

멀티 스테이지 빌드 방법으로 최종 이미지의 용량을 줄일 수 있고 호스트에 어떠한 빌드 도구도 설치할 필요가 없다.

멀티 스테이지는 docker-ce 17.06 버전 부터 지원한다.

```docker
FROM openjdk:8 AS int-build
LABEL description="Java Application builder"
RUN git clone https://github.com/iac-source/inbuilder.git
WORKDIR inbuilder
RUN chmod 700 mvnw
RUN ./mvnw clean package

FROM gcr.io/distroless/java:8
LABEL description="Echo IP Java Application"
EXPOSE 60434
COPY --from=int-build inbuilder/target/app-in-host.jar /opt/app-in-image.jar
WORKDIR /opt
ENTRYPOINT [ "java", "-jar", "app-in-image.jar" ]
```

```bash
docker build -t multistage-img .
```

```bash
docker images | head -n 3
REPOSITORY                           TAG                 IMAGE ID            CREATED              SIZE
multistage-img                       latest              5dcef45eff3c        About a minute ago   148MB
<none>                               <none>              3b3df8d15a3a        About a minute ago   615MB
```

<none> 으로 표시된 이미지는 멀티 스테이지 과정에서 자바 소스를 빌드하는 과정에서 생성된 댕글링 이미지이다.

공간을 적게 사용하는 이미지를 만드는 것이 목적이르모 댕글링 이미지를 삭제한다.

```bash
docker rmi $(docker images -f dangling=true -q)
```