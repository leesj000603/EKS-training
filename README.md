# EKS-training
AWS EKS 학습을 통해 Kubernetes 클러스터를 생성하고, Spring Boot 애플리케이션을 Docker 컨테이너로 배포, LoadBalancer 서비스를 구현하여 레플리카로 생성된 Springboot pod들로 트래픽이 분산됨을 확인.

## 참고
- [AWS EKS 사용자 가이드 - kubectl 설치](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/install-kubectl.html)
- [AWS EKS Workshop](https://catalog.us-east-1.prod.workshops.aws/workshops/46236689-b414-4db8-b5fc-8d2954f2d94a/ko-KR/eks/10-install)


## AWS CLI 설치
```
$sudo apt-get update
$sudo apt-get install unzip


# curl명령어 사용,
# -o옵션은 다운로드한 패키지가 기록되는 파일 이름을 지정한다.
# 현재 디렉토리의 awscliv2.zip 으로 설치됨
$curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# 설치 프로그램을 압축 해제
# 패키지를 압축 해제하고 aws현재 디렉토리 아래에 이름이 지정된 디렉토리를 만든다.
$unzip awscliv2.zip

# install설치 프로그램을 실행,
# 설치 명령은 새로 압축 해제된 디렉토리에 있는 파일을 사용한다. 
# 기본적으로 모든 파일은 aws. 에 설치되고 심볼릭 링크가 /usr/local/aws-cli 에 생성
$sudo ./aws/install

# 설치 확인
$aws —version

# AWS CLI 설정
$aws configure
AWS Access Key ID [None]: access 키 아이디 입력
AWS Secret Access Key [None]: 시크릿 accesskey 입력
Default region name [None]: ap-northeast-2 #서울
Default output format [None]: json

```

## kubectl 설치
```
# 1.28 버전 설치
$curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.28.11/2024-07-12/bin/linux/amd64/kubectl

# 디바이스의 하드웨어 플랫폼용 명령을 사용하여 Amazon S3에서 클러스터의 Kubernetes 버전에 대한 SHA-256 체크섬을 다운로드
$curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.28.11/2024-07-12/bin/linux/amd64/kubectl.sha256

# 다운받은 SHA-256 체크섬을 사용하여 다운로드한 바이너리를 확인
$sha256sum -c kubectl.sha256
kubectl: OK

# 바이너리에 실행 권한 적용
$chmod +x $HOME/bin/kubectl


# 바이너리를 PATH의 폴더에 복사한다. kubectl 버전이 이미 설치된 경우 $HOME/bin/kubectl을 생성하고 $HOME/bin이 $PATH로 시작하도록 해야 한다.
$mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH


# kubectl 설치 확인
$kubectl version --client
Client Version: v1.28.11-eks-1552ad0
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3

# eksctl을 다운로드하고 압축해제
$curl --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

# 이동한 후 eksctl 버전 확인
$sudo mv -v /tmp/eksctl /usr/local/bin
$eksctl version


# AWS_REGION 환경변수 설정
$export AWS_REGION=ap-northeast-2

# eks 클러스터 배포
#  --region에 클러스터가 있는 AWS 리전. --name에 를 클러스터 이름. kubectl과 버전 맞추기.
# 15분 정도 소요
$eksctl create cluster --name [클러스터명] --version 1.28 --region ${AWS_REGION}


# 노드 확인
$kubectl get nodes
```

### 기본적으로 두개의 노드 생성
![image](https://github.com/user-attachments/assets/a02a1058-662d-4244-a2ef-2f61708b7799)

### ekctl vpc가 region에 생성됨
![image](https://github.com/user-attachments/assets/34b05d20-569f-4db2-8227-c4529db4387f)

EKS 클러스터의 노드는 ec2로 생성되며 각 노드는 자동 생성 또는 지정한 vpc에 위치하게 된다.


## Spring boot 어플리케이션
### 기본 url로 접속시 greeting 메세지와 /message로 접속시 pod 정보를 확인할 수 있도록 구현
![image](https://github.com/user-attachments/assets/a95c3726-73f6-49f6-9828-f3df868bc319)

![image](https://github.com/user-attachments/assets/34d2cd4f-da03-4bcc-9b96-c4f432a73c27)
![image](https://github.com/user-attachments/assets/64e23965-d688-4b3b-96e3-1a879094c4bd)


## docker 이미지 생성 및 dockerhub에 저장
```
# JRE가 포함된 이미지 사용
FROM eclipse-temurin:17-jre

# 작업 디렉토리 설정
WORKDIR /app

# 로컬에서 JAR 파일을 컨테이너로 복사
COPY app.jar app.jar

# JAR 파일 실행
ENTRYPOINT ["java", "-jar", "app.jar"]
```
```
docker build -t eks-example .
docker tag eks-example leesj000603/eks-example:latest
docker push leesj000603/eks-example:latest
```

![image](https://github.com/user-attachments/assets/a2d22bd8-b40a-48bb-9806-ad72e7c4cd57)

## Pod 생성
```
kubectl run eks-example-a --image=leesj000603/eks-example

kubectl get pods
NAME          READY   STATUS   RESTARTS      AGE
nginx-apple   0/1     Error    2 (23s ago)   42s

#error가 난 것은 PODNAME 환경변수를 설정해주지 않았기 때문
```

```
#PODNAME 환경변수를 --env옵션으로 설정
kubectl run eks-example-a --image=leesj000603/eks-example --env PODNAME=eks-example-a

kubectl get pods
NAME            READY   STATUS    RESTARTS   AGE
eks-example-a   1/1     Running   0          4s
```

### 정상적으로 Springboot 컨테이너를 올린 pod가 Running status로 된 것을 확인하였으니 삭제
```
kubectl delete pod eks-example-a
```


## Deployment

### Deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: eks-example-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: eks-example
  template:
    metadata:
      labels:
        app: eks-example
    spec:
      containers:
        - name: eks-example-container
          image: leesj000603/eks-example
          env:
            - name: PODNAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name  # metadata에서 가져온 파드 이름을 환경변수로 설정
          ports:
            - containerPort: 8080  # 파드에서 사용할 포트
```

### Deployment 생성및 확인
```
kubectl apply -f deployment.yaml

kubectl get deployment,replicaset,pods
NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/eks-example-deployment   3/3     3            3           15s

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/eks-example-deployment-747c87f49b   3         3         3       15s

NAME                                          READY   STATUS    RESTARTS   AGE
pod/eks-example-deployment-747c87f49b-9chfp   1/1     Running   0          15s
pod/eks-example-deployment-747c87f49b-b2n6g   1/1     Running   0          15s
pod/eks-example-deployment-747c87f49b-hgxrp   1/1     Running   0          14s
```

## Service
### Service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: eks-example-service
spec:
  type: LoadBalancer  # 로드 밸런서 타입
  selector:
    app: eks-example  # Deployment와 연결되는 레이블
  ports:
    - protocol: TCP
      port: 80        # 외부에서 접근할 포트
      targetPort: 8080 # 파드의 포트

```

### Service 생성 및 확인
```
kubectl apply -f service.yaml

kubectl get service,deployment,replicaset,pods
NAME                          TYPE           CLUSTER-IP     EXTERNAL-IP                                                                    PORT(S)        AGE
service/eks-example-service   LoadBalancer   10.100.97.60   ac65545001bc14715986d2bc35e2b9e5-1148121724.ap-northeast-2.elb.amazonaws.com   80:32174/TCP   32s
service/kubernetes            ClusterIP      10.100.0.1     <none>                                                                         443/TCP        62m

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/eks-example-deployment   3/3     3            3           2m30s

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/eks-example-deployment-747c87f49b   3         3         3       2m30s

NAME                                          READY   STATUS    RESTARTS   AGE
pod/eks-example-deployment-747c87f49b-9chfp   1/1     Running   0          2m30s
pod/eks-example-deployment-747c87f49b-b2n6g   1/1     Running   0          2m30s
pod/eks-example-deployment-747c87f49b-hgxrp   1/1     Running   0          2m29s
```

## 정상동작 확인

### 서비스 External IP로 접속 확인
![image](https://github.com/user-attachments/assets/9d2ace26-0501-4505-b86b-ae7181c6c161)


기존 브라우저 세션에서 요청을 보내는 경우, 클라이언트의 요청이 이전에 연결된 파드로 다시 전송되기 때문에. 이로 인해 새로 시작한 세션에서 요청이 다른 파드로 전송되도록 

각기 다른 크롬 창을 열어 각각의 브라우저 세션을 생성하였다.
### 로드밸런싱 확인
![image](https://github.com/user-attachments/assets/583ec174-956e-4adb-b108-f11899c78a20)
![image](https://github.com/user-attachments/assets/590fc991-10cf-4000-95bc-83a6f0efea39)
![image](https://github.com/user-attachments/assets/267f558f-92c5-440a-b73d-8ae64675fe7d)


## EKS 클러스터 삭제

```
eksctl delete cluster --name [클러스터명] --region ${AWS_REGION}
```
### 꼭 삭제! 클러스터 한개를 사용하는 경우, 꽤 비싼 요금인 시간당 0.1 USD가 부과된다..
