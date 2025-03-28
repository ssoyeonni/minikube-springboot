# ☸️ Kubernetes(Minikube) 기반 Spring Boot 배포
<br>

## 🧾 프로젝트 소개
이 프로젝트는 Spring Boot 애플리케이션을 컨테이너화하고, Minikube 기반의 Kubernetes 클러스터에서 NodePort 및 LoadBalancer 서비스를 통해 배포하는 실습 프로젝트입니다. <br>

개발자가 직접 만든 Docker 이미지를 Docker Hub에 업로드한 후, Kubernetes에 배포하고 클러스터 외부에서 접근하는 과정을 실습하며,
CI/CD 초기 흐름과 클라우드 네이티브 애플리케이션 배포 구조에 대한 이해를 높이는 것을 목표로 합니다.

<br>

### ⚙️ 개발 환경
| 항목               | 내용 |
|--------------------|------|
| 운영체제 (OS)       | Ubuntu 24.04.2 LTS |
| 가상화 소프트웨어   | VMware |
| 컨테이너 및 오케스트레이션 | Kubernetes (Minikube), docker |
| 언어 및 프레임워크  | Java 17, Spring Boot |
| 빌드 도구           | Gradle |
| 배포 방식           | NodePort, LoadBalancer |

<br>

## 📦 1. springboot 애플리케이션 이미지 생성

### 1) jar 파일 생성
> Run As -> Run Configuration -> Gradle Task -> New_configuration

<img src="https://github.com/user-attachments/assets/86129f42-d24f-4733-a9f1-3dac221ed2c2" width="600" />

<img src="https://github.com/user-attachments/assets/08b35375-1a59-4e40-9d22-2e4586d08caa" width="600" />


### 2) Dockerfile 생성 & 빌드
> 이미지를 빌드하는 코드를 Dockerfile에 넣어 실행합니다. 

- Dockerfile
  ```
  FROM eclipse-temurin:17-jre-alpine

  WORKDIR /app

  COPY step07_CICD-0.0.1-SNAPSHOT.jar app.jar

  ENTRYPOINT ["java", "-jar", "app.jar"]
  ```

- Dockerfile 빌드
  ```
  # docker build <이미지 이름> <위치>
  
  docker build springappimg .
  ```

<br>

## 🐳 2. docker hub에 이미지 업로드
- 태그 붙인 이미지를 업로드
```
docker push ssoyeonni/springappimg:1.1
```


### 1) docker 로그인
```
docker login
```
<img src="https://github.com/user-attachments/assets/5266589b-7236-47fe-b48c-537a49cf9e68" width="500" />


### 2) docker tag
현재 springappimg 라는 이름의 이미지 파일이 생성되어 있습니다. <br>
docker hub에 이미지를 업로드 할 때 ```docker push``` 명령을 사용하는데, 이미지에 정확한 저장소 경로가 붙어 있어야 작동이 되므로, tag를 붙여줘야 합니다.

```
# docker tag <이미지명> <docker username>/<이미지명>:태그

docker tag springappimg ssoyeonni/springappimg:1.1
```
<img src="https://github.com/user-attachments/assets/719a47aa-a606-4e5a-944f-4a6ea1f2801a" width="600" />


### 3) 이미지 업로드

- docker hub에 업로드
  ```
  docker push ssoyeonni/springappimg:1.1
  ```

- 업로드 확인
  
  <img src="https://github.com/user-attachments/assets/b14add21-91b2-4d71-8246-bdbf93d93516" width="600" />

<br>

## 🔗 3. minikube - NodePort type
**외부에서 클러스터 내부의 애플리케이션(pod)로 접근하는 방법**에는 **NodePort**와 **LoadBalancer**가 있습니다.<br>
그 중 **minikube**는 Kubernetes 클러스터의 각 노드의 포트를 열어 외부에서 직접 접근할 수 있도록 하는 방식입니다.

### 1) spring-nodeport.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-deployment-n
spec:
  replicas: 2
  selector:
    matchLabels:
      app: springapp
  template:
    metadata:
      labels:
        app: springapp
    spec:
      containers:
      - name: springapp
        image: ssoyeonni/springappimg:1.1
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: spring-service-n
spec:
  selector:
    app: springapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080     # 30000 ~ 32767 가능
  type: NodePort

```

### 2) spring-nodeport.yaml을 minikube에서 배포
```
kubectl apply -f spring-nodeport.yaml

kubectl get all
```

### 3) NodePort 정상 실행 확인

```
kubectl get all
```

<img src="https://github.com/user-attachments/assets/df7a8e17-9c56-4593-8014-002aa9ae5333" width="600" />

> minikube ip: 192.168.49.2 <br>
> port: 30080

- curl 명령
  ```
  curl 10.105.90.96/fisa1

  curl 192.168.49.2:30080/fisa1
  ```
  
  <img src="https://github.com/user-attachments/assets/fb5a31f8-1a36-4bd5-b7ef-c7fafd1a1f42" width="600" />
  
- 브라우저 접속
  > ubuntu에 있는 minikube의 실행중인 애플리케이션을 브라우저에서 접속하려면 포트포워딩 필요
  
  <img src="https://github.com/user-attachments/assets/8987f221-0473-4829-af2a-767acaa1f0af" width="400" />

  ```
  localhost:30080/fisa1
  ```

  <img src="https://github.com/user-attachments/assets/cbf85bf0-0663-493e-bf51-a517c24b7de8" width="600" />

<br>

## 🪢 4. minikube - LoadBalancer type
**LoadBalancer**는 클라우드 제공자의 로드밸런서를 자동으로 생성해서 클러스터 외부에서 Pod로 접근할 수 있게 해주는 Service 타입입니다.

### 1) spring-loadbalancer.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-deployment-l
spec:
  replicas: 2
  selector:
    matchLabels:
      app: springapp
  template:
    metadata:
      labels:
        app: springapp
    spec:
      containers:
      - name: springapp
        image: ssoyeonni/springappimg:1.1
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: spring-service-l
spec:
  type: LoadBalancer
  selector:
    app: springapp
  ports:
    - protocol: TCP
      port: 8088       # External-ip 사용하는 서비스 포트
      targetPort: 80 # 컨테이너 포트

```

### 2) spring-loadbalancer.yaml을 minikube에서 배포
```
kubectl apply -f spring-loadbalancer.yaml

kubectl get all

# 다른 터미널 창에서
minikube tunnel

```


### 3) LoadBalancer 정상 실행 확인

```
kubectl get all
```

<img src="https://github.com/user-attachments/assets/e5b990fb-44f8-4e0f-a3c9-e3e48f8f1423" width="600" />

> minikube ip: 192.168.49.2 <br>
> port: 31892

- curl 명령
  ```
  curl 192.168.49.2:31892/fisa1
  ```
  
  
- 브라우저 접속
  > 포트포워딩
  
  <img src="https://github.com/user-attachments/assets/ab96ceae-6391-4877-91ac-c0e810c22f04" width="400" />


  ```
  localhost:31892/fisa1
  ```

  <img src="https://github.com/user-attachments/assets/92b41efa-2e59-467a-a65c-ee0f44e3aff4" width="600" />


## 💣 트러블슈팅
### ❌ 1) pod가 CrashLoopBackOff 상태인 문제
   <img src="https://github.com/user-attachments/assets/c238e95b-c358-47ae-8f51-c77577fb6675" width="600" />

  - 원인: Dockerfile에서 복사한 app.jar로 실행해야하는데 springapp.jar로 실행시켰어서 에러
    ```
    FROM eclipse-temurin:17-jre-alpine

    WORKDIR /app

    COPY step07_CICD-0.0.1-SNAPSHOT.jar app.jar

    ENTRYPOINT ["java", "-jar", "springapp.jar"]
    ```
  - 해결법: springapp.jar -> app.jar로 변경 후 이미지 재빌드
    ```
    ubuntu@myserver03:~/springboot$ kubectl logs spring-deployment-n-64469fd884-cqv8z
    Error: Unable to access jarfile springapp.jar
    ```
    - 에러
      
### ❌ 2) Dockerfile을 수정하고 재빌드해도 여전히 springapp.jar로 실행하려함
  - 해결법: docker hub에 이미지 업로드 시 버전을 다르게 줌 (1.0 -> 1.1)
    <img src="https://github.com/user-attachments/assets/bd4f1af0-48c5-439a-8bbe-5758197cbe10" width="600" />
      - status가 Running으로 됨. 성공!!
