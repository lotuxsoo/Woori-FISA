# 🌱 Spring Boot 애플리케이션 Kubernetes 배포 가이드

![Linux](https://img.shields.io/badge/Linux-Operating%20System-blue) ![Docker](https://img.shields.io/badge/Docker-Containerization-2496ED) ![Kubernetes](https://img.shields.io/badge/Kubernetes-Container%20Orchestration-326CE5) ![SpringBoot](https://img.shields.io/badge/SpringBoot-Framework-green)

Spring Boot 프로젝트에서 `.jar` 파일을 생성하고, Docker 이미지로 만들어서 Kubernetes(k8s) 환경에서 다중 인스턴스로 배포하는 과정을 정리했습니다.

<br>

## 1. Spring Boot .jar 파일 생성

Spring Boot 프로젝트에서 실행 가능한 `.jar` 파일을 만들기 위해서는 Gradle 빌드를 사용합니다.

### 1.1 Gradle 설정 파일 (build.gradle)

프로젝트 루트 디렉터리에 있는 `build.gradle` 파일에서 Spring Boot 플러그인과 빌드 작업을 설정합니다.

```gradle
plugins {
    id 'org.springframework.boot' version '3.0.0'
    id 'io.spring.dependency-management' version '1.0.15.RELEASE'
    id 'java'
}

group = 'com.example'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '17'

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

tasks.named('test') {
    useJUnitPlatform()
}

bootJar {
    archiveFileName = 'SpringApp.jar'  # 생성될 .jar 파일 이름 설정
}
```

### 1.2 .jar 파일 빌드

Gradle 프로젝트에서 `.jar` 파일을 빌드하려면 아래 명령어를 실행합니다.

```bash
./gradlew clean bootJar
```

이 명령어를 실행하면 `build/libs/` 디렉터리에 `SpringApp.jar` 파일이 생성됩니다.

## 2. Docker 이미지 생성

`.jar` 파일이 준비되면, 이를 Docker 이미지로 변환하여 컨테이너에서 실행할 수 있습니다.

### 2.1 Dockerfile 작성

`Dockerfile`을 프로젝트 루트 디렉터리에 생성하여 Docker 이미지를 정의합니다.

```Dockerfile
# Dockerfile
FROM openjdk:17-jdk-alpine
VOLUME /tmp
COPY SpringApp.jar app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

### 2.2 Docker 이미지 빌드

이제 `SpringApp.jar` 파일을 포함하는 Docker 이미지를 빌드합니다.

```bash
docker build -t myusername/bootapp:1.0 .
```

`myusername` 부분을 Docker Hub 사용자명으로 변경하세요. 빌드된 Docker 이미지는 `bootapp:1.0`으로 태그됩니다.

### 2.3 Docker Hub에 이미지 푸시

Docker Hub에 이미지를 푸시하여 Kubernetes에서 사용할 수 있도록 합니다.

```bash
docker login
docker tag bootapp:1.0 myusername/bootapp:1.0
docker push myusername/bootapp:1.0
```

이미지를 푸시한 후, Docker Hub에서 이미지를 확인할 수 있습니다.
![image](https://github.com/user-attachments/assets/bb24fab8-60b2-4a98-b471-a03682069a46)

## 3. Kubernetes 배포 설정

Docker 이미지를 Kubernetes 클러스터에 배포하기 위한 설정을 진행합니다.

### 3.1 Deployment 설정 파일 작성

`deployment.yaml` 파일을 생성하여 애플리케이션을 배포하는 Deployment를 정의합니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-app-deployment
spec:
  replicas: 3 # 파드 개수
  selector:
    matchLabels:
      app: spring-app
  template:
    metadata:
      labels:
        app: spring-app
    spec:
      containers:
        - name: spring-app-container
          image: myusername/bootapp:1.0 # Docker Hub에 푸시한 이미지
          ports:
            - containerPort: 8080 # 컨테이너 포트
```

### 3.2 Service 설정 파일 작성

외부 통신을 위한 LoadBalancer 타입의 서비스를 생성합니다. `service.yaml` 파일을 작성합니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: spring-app-service
spec:
  selector:
    app: spring-app
  ports:
    - protocol: TCP
      port: 80 # 외부에서 접근할 포트
      targetPort: 8080 # 파드의 컨테이너 포트
  type: LoadBalancer
```

## 4. Kubernetes에 애플리케이션 배포

### 4.1 Deployment 및 Service 적용

배포와 서비스 설정을 Kubernetes에 적용합니다.

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

### 4.2 배포 확인

파드가 정상적으로 실행 중인지 확인합니다.

```bash
kubectl get pods
```

그리고 서비스가 외부 IP를 할당받았는지 확인합니다.

```bash
kubectl get svc
```

![image](https://github.com/user-attachments/assets/c12d75e2-44f1-4484-8c21-30c3713ec7e0)

### 🤔 개념 외우기

#### **Minikube IP**

    - Minikube 클러스터의 외부 접근을 위한 IP 주소

#### **Cluster IP**

    - Kubernetes 클러스터 내에서 서비스 간의 통신을 위한 내부 IP 주소

#### 포트 포워딩 설정

우분투 환경에서 Spring Boot 애플리케이션을 윈도우에서 접근할 수 있도록 **포트 포워딩**을 설정했습니다.<br/>
spring-app-service의 Cluster IP는 10.105.120.146:80이며, 이를 localhost:80으로 포워딩하여 외부에서 접근할 수 있도록 했습니다.
![image](https://github.com/user-attachments/assets/61981b68-82b6-4976-be7c-d4c4f830fea1)

#### Spring Boot 애플리케이션 접속 확인

브라우저에서 localhost:80으로 접속하여 Spring Boot 애플리케이션이 정상적으로 작동하는 것을 확인했습니다.
![image](https://github.com/user-attachments/assets/abd06577-7004-4027-b4b6-5e18741bca64)

Minikube를 사용하는 경우, 다음 명령어로 서비스 URL을 확인할 수 있습니다.

```bash
minikube service spring-app-service
```

![image](https://github.com/user-attachments/assets/151cf600-8cdb-48ea-9f2a-25b5df4471ae)

## 5. 스케일링 및 모니터링

### 5.1 애플리케이션 스케일링

애플리케이션의 파드 수를 늘리려면 다음 명령어로 스케일링을 수행합니다.

```bash
kubectl scale deployment spring-app-deployment --replicas=5
```

### 5.2 모니터링

Kubernetes 리소스 상태를 모니터링하려면 다음 명령어를 사용할 수 있습니다.

```bash
kubectl top pods
kubectl top nodes
```

![image](https://github.com/user-attachments/assets/0513276e-20b4-47c8-93db-8204d2a87ae5)

## 6. Kubernetes 대시보드 확인

Kubernetes 대시보드를 사용해 클러스터 상태를 시각적으로 모니터링할 수 있습니다. Minikube에서 대시보드를 실행하려면:

```bash
minikube dashboard
```

대시보드가 브라우저에서 열리며, 현재 배포된 애플리케이션, 파드 상태, 리소스 사용량 등을 시각적으로 확인할 수 있습니다.
![image](https://github.com/user-attachments/assets/d5924f95-2c9b-4baf-a9b2-52d2fe30b68d)

## 7. 리소스 정리

애플리케이션과 관련된 리소스를 삭제하려면 다음 명령어를 사용합니다.

```bash
kubectl delete -f deployment.yaml
kubectl delete -f service.yaml
```

## 💡 요약 정리

1. **Spring Boot 프로젝트에서 .jar 파일 생성**: Gradle을 사용해 Spring Boot 애플리케이션의 `.jar` 파일을 생성합니다.
2. **Docker 이미지 생성 및 푸시**: `.jar` 파일을 Docker 이미지로 빌드하고 Docker Hub에 푸시합니다.
3. **Kubernetes 배포 설정**: Deployment와 Service 설정을 작성하여 Kubernetes 클러스터에 배포합니다.
4. **애플리케이션 배포 확인 및 외부 통신 설정**: 배포된 파드와 서비스를 확인하고, 외부 통신이 가능하도록 설정합니다.
5. **스케일링 및 모니터링**: 애플리케이션을 스케일링하고, Kubernetes 대시보드에서 리소스를 모니터링합니다.
6. **리소스 정리**: 더 이상 필요 없는 리소스는 `kubectl delete` 명령어로 정리합니다.
