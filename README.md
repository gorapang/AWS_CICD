# Jenkins & AWS CI/CD Pipeline
## 팀원
| <img src="https://github.com/gorapang.png" width="80"> | <img src="https://github.com/Ungbbi.png" width="80"> | <img src="https://github.com/castlhoo.png" width="80"> | <img src="https://github.com/Been980804.png" width="80"> |
|:---:|:---:|:---:|:---:|
| [박정주](https://github.com/gorapang) | [박웅빈](https://github.com/Ungbbi) | [김성호](https://github.com/castlhoo) | [이현빈](https://github.com/Been980804) |

---
## 개요
**Jenkins**와 **AWS S3 & Lambda**를 이용하여 **CI/CD** 파이프라인을 통해 배포를 자동화 하고, **EC2**를 통해 서비스를 구동합니다. 전체 파이프라인은 다음과 같은 단계로 이루어집니다:
1. [**Jenkins와 GitHub 간 웹훅 연결**](#️-jenkins-docker-설치)
2. [**ngrok으로 로컬 Jenkins를 외부에서 접근 가능하게 설정**](#-ngrok-연동)
3. [**Spring Boot 애플리케이션과 AWS RDS 연동**](#-SpringApp-RDS-연동 )
4. [**AWS CLI를 이용한 Jenkins의 S3 연동**](#-aws-cli-연동)
5. [**빌드된 JAR 파일 S3에 업로드**](#-aws-cli-연동)
6. [**AWS IAM 설정**](#-aws-iam-설정)
7. [**S3에 이벤트 트리거 설정 및 람다 함수 생성**](#-s3-ET-Lambda)
8. [**Trouble Shooting**](#-Trouble-Shooting)


## 🏗️ **아키텍처 구조**
![image](https://github.com/user-attachments/assets/25d05527-ef0f-47cc-8dda-49a0a8f39a83)

이 아키텍처는 Jenkins에서 Spring Boot 애플리케이션을 빌드한 후, S3에 업로드하고, Lambda와 EC2를 통해 AWS에서 실행하는 과정입니다.

---

## 1. 🌐 **Jenkins와 GitHub 간 웹훅 연결**
![image](https://github.com/user-attachments/assets/4d76e7e6-05ab-4ac2-8375-8dc28ce51c39)
Jenkins는 Docker 컨테이너 내에 설치되어 있으며 8888 포트를 통해 접근 가능합니다.

- **웹훅 설정**: Jenkins가 GitHub과 연결되어 코드가 변경될 때마다 자동으로 빌드가 트리거됩니다.

![image](https://github.com/user-attachments/assets/56c34ba1-41a8-4f11-8baa-110b545e0c32)

Jenkins 설정 후, GitHub 웹훅을 연결하여 자동 빌드 설정을 완료합니다.

---

## 2. 🛠️ **ngrok으로 Jenkins를 외부에서 접근 가능하게 설정**
![image](https://github.com/user-attachments/assets/935b551e-ceaa-48e1-8a57-14637d1e95b0)
ngrok을 사용하여 로컬에서 실행 중인 Jenkins를 외부에서 접근할 수 있도록 HTTPS 주소를 제공합니다.

- **ngrok 설정**: ngrok URL을 GitHub 웹훅에 입력하여 GitHub에서 Jenkins로 요청을 보낼 수 있도록 설정합니다.
  
  예시: `https://078e-118-131-63-236.ngrok-free.app/webhook/`

---

## 3. ⚙️ **Spring Boot 애플리케이션과 AWS RDS 연동**
Spring Boot 애플리케이션은 8080 포트에서 실행되며, Gradle을 사용해 JAR 파일로 패키징됩니다.

- **RDS 연동**: Jenkins에서 프로젝트를 빌드한 후, 빌드된 JAR 파일을 S3에 업로드합니다.
- **AWS 설정**: VPC, 서브넷, 라우팅 테이블, 게이트웨이를 사용해 EC2 인스턴스를 설정하고, Jenkins를 통해 S3 버킷에 빌드된 JAR 파일을 저장합니다.


---

## 4. 📂 **Jenkins와 AWS S3 연결 (AWS CLI 사용)**
Jenkins에서 AWS S3로 JAR 파일을 업로드하기 위해 아래 단계를 따릅니다.

### 🔐 **Step 1: Jenkins 컨테이너에 AWS CLI 설치**
```bash
# Jenkins 컨테이너에 root 권한으로 접속
docker exec -it --user root myjenkins /bin/bash

# AWS CLI 설치
apt-get update
apt-get install awscli -y

# AWS CLI 자격증명 설정
aws configure
```

### 📂 **Step 2: Jenkins가 AWS 자격 증명을 읽을 수 있도록 설정**
```bash
# AWS credentials 파일 위치 확인
find / -name credentials 2>/dev/null

# Jenkins에서 credentials 파일을 읽을 수 있게 설정
mkdir -p /var/jenkins_home/.aws
cp ~/.aws/credentials /var/jenkins_home/.aws/credentials

# 권한 설정
chmod 600 /var/jenkins_home/.aws/credentials
chown jenkins:jenkins /var/jenkins_home/.aws/credentials
```

---

## 5. 📦 **JAR 파일을 S3에 업로드하는 Jenkins 파이프라인**

다음 Jenkins 파이프라인 스크립트는 GitHub에서 코드를 클론한 후, Gradle을 사용해 빌드를 실행하고 결과 JAR 파일을 AWS S3 버킷에 업로드하는 작업을 수행합니다.

### Jenkins 파이프라인 스크립트:
```groovy
pipeline {
    agent any

    stages {
        # 1. GitHub에서 레포지토리 클론하는 단계
        stage('Clone Repository') {
            steps {
                # 'main' 브랜치에서 GitHub 레포지토리를 클론
                git branch: 'main', url: 'https://github.com/gorapang/AWS_CICD'
            }
        }

        # 2. Gradle 빌드 실행
        stage('Build') {
            steps {
                # 'step18_empApp' 디렉토리로 이동하여 그 안에서 빌드를 수행
                dir('./step18_empApp') {
                    # 1. gradlew 파일에 실행 권한을 부여합니다.
                    sh 'chmod +x gradlew'
                    
                    # 2. Gradle을 이용해 클린 빌드를 수행하고, 테스트 단계를 생략
                    sh './gradlew clean build -x test'
                    
                    # 3. 현재 작업 공간 경로를 출력
                    sh 'echo $WORKSPACE'
                }
            }
        }

        # 3. 빌드된 JAR 파일을 AWS S3 버킷에 업로드
        stage('Upload to S3') {
            steps {
                script {
                    # 업로드할 JAR 파일 경로와 S3 버킷, 업로드 경로를 설정
                    def jarFile = '/var/jenkins_home/workspace/AWS-CLI/step18_empApp/build/libs/step18_empApp-0.0.1-SNAPSHOT.jar'
                    def s3Bucket = 'ce05-bucket-02'
                    def s3Key = 'ce05-Oct11Project/step18_empApp-0.0.1-SNAPSHOT.jar'
                    
                    # AWS CLI 명령어를 사용하여 JAR 파일을 S3에 업로드
                    sh "/usr/bin/aws s3 cp ${jarFile} s3://${s3Bucket}/${s3Key}"
                }
            }
        }
    }
}
```

### 각 단계 설명

1. **Clone Repository 단계**:
   - GitHub의 'main' 브랜치에서 레포지토리를 클론하여 Jenkins 작업 공간으로 복사
   - `git branch: 'main', url: 'https://github.com/gorapang/AWS_CICD'`: 'main' 브랜치에서 지정된 URL의 레포지토리를 가져옵니다.

2. **Build 단계**:
   - 클론한 코드 중, `step18_empApp` 디렉토리 안에서 Gradle 빌드를 수행합니다.
   - `chmod +x gradlew`: Gradle Wrapper (`gradlew`) 파일에 실행 권한을 부여합니다.
   - `./gradlew clean build -x test`: Gradle을 사용해 클린 빌드를 수행하되, 테스트는 건너뜁니다.
   - `sh 'echo $WORKSPACE'`: 현재 Jenkins 작업 공간의 경로를 출력합니다.

3. **Upload to S3 단계**:
   - 빌드가 완료된 후, 결과물인 JAR 파일을 AWS S3 버킷에 업로드합니다.
   - `def jarFile`: 업로드할 JAR 파일의 경로를 지정합니다.
   - `def s3Bucket`: 업로드할 AWS S3 버킷의 이름을 설정합니다.
   - `def s3Key`: S3 버킷 내에서 JAR 파일이 저장될 경로를 설정합니다.
   - `sh "/usr/bin/aws s3 cp ${jarFile} s3://${s3Bucket}/${s3Key}"`: AWS CLI를 사용해 JAR 파일을 S3에 업로드합니다.
   
---

### 📄 주요 파일 및 경로
- **JAR 파일**: 빌드된 파일의 경로는 `/var/jenkins_home/workspace/AWS-CLI/step18_empApp/build/libs/step18_empApp-0.0.1-SNAPSHOT.jar`입니다.
- **S3 버킷**: 업로드할 S3 버킷은 `ce05-bucket-02`입니다.
- **S3 경로**: S3 버킷 내에서 JAR 파일은 `ce05-Oct11Project/step18_empApp-0.0.1-SNAPSHOT.jar`로 저장됩니다.

이 Jenkins 파이프라인은 코드의 빌드 및 배포 프로세스를 자동화하여 S3에 업로드함으로써, AWS 인프라에서의 애플리케이션 배포를 간소화합니다.

---

### 실제 결과화면
![image](https://github.com/user-attachments/assets/752cec3e-1428-4490-916f-99190d7308dc)

![image](https://github.com/user-attachments/assets/dbf2671b-1751-4f76-9484-b1e4f2bdb8fb)

![image](https://github.com/user-attachments/assets/04b4b174-fe9f-4354-9ef0-5f4486bad359)

---

## 6. 🔒 **AWS IAM 설정 및 EC2 구성**

### 1. **IAM 역할 생성**
![image](https://github.com/user-attachments/assets/aa495f35-14e6-4dd0-951e-a02c1237c973)

- **ce05-lambda**: Lambda가 EC2와 상호작용할 수 있는 권한을 부여합니다.
- **ce05-EC2**: EC2가 AWS 서비스를 사용할 수 있는 권한을 부여합니다.

![image](https://github.com/user-attachments/assets/ae04f6c0-54de-4711-a3f6-1106b88b8a21)

### 2. **IAM 역할을 EC2에 연결**
- 생성한 **ce05-EC2** 역할을 EC2 인스턴스에 연결하여 필요한 권한을 부여합니다.

---

##7. S3에 이벤트 트리거 설정 및 람다 함수 생성
### 1. **S3 버킷 이벤트 트리거 연결 설정하기 🔗**
> 1. S3 콘솔 접속
- AWS 콘솔에서 **S3**로 이동한 후, S3 버킷 **ce05-bucket-02**를 선택합니다.

> 2. 이벤트 알림 설정 ⚙️
![image](https://github.com/user-attachments/assets/baa0c433-258b-49a4-a7fb-a6cce078d09a)
- 버킷 페이지에서 상단의 **Properties(속성)** 탭으로 이동합니다.
- **Event notifications(이벤트 알림)** 섹션을 찾고 **Create event notification(이벤트 알림 생성)** 버튼을 클릭합니다.

> 3. 이벤트 알림 세부 설정 🛠️
- **Event name(이벤트 이름)**: 이벤트에 대한 이름을 지정합니다. 예: `lambda-trigger`.
- **Prefix**: `ce05-Oct11Project/` (특정 폴더에서 이벤트만 감지하려면 설정).
- **Event types(이벤트 유형)**: **All object create events(모든 객체 생성 이벤트)**를 선택하여 모든 객체 생성 시 이벤트를 트리거합니다.
- **Destination(대상)**: **Lambda function(람다 함수)**를 선택한 후, 기존에 생성한 Lambda 함수를 지정합니다.
- **Save changes(변경 사항 저장)** 버튼을 클릭하여 이벤트 트리거 설정을 완료합니다.

![image](https://github.com/user-attachments/assets/f42201c3-00d0-43a8-813b-57e78144fa3e)

### 2. **Lamda 함수 적용**
```python
import json
import boto3

# Boto3 클라이언트 생성
ssm_client = boto3.client('ssm')

def lambda_handler(event, context):
    # S3 이벤트에서 버킷 이름과 객체 키(JAR 파일 이름)를 가져옴
    bucket_name = event['Records'][0]['s3']['bucket']['name']
    jar_file_key = event['Records'][0]['s3']['object']['key']
    
    # EC2 인스턴스 ID 설정
    ec2_instance_id = 'ec2_instance_id'

    # S3에서 EC2로 JAR 파일 복사 및 Spring Boot 애플리케이션 재시작 명령어
    commands = [
        # S3에서 JAR 파일을 EC2로 복사
        f'aws s3 cp s3://{bucket_name}/{jar_file_key} /home/ubuntu/app/app.jar',
        # 기존 애플리케이션 종료 (java 프로세스 종료)
        'pkill -f "java -jar" || true',
        # 새로운 JAR 파일 실행 (nohup으로 백그라운드 실행 및 로그 저장)
        'nohup java -jar /home/ubuntu/app/app.jar > /home/ubuntu/app/app.log 2>&1 &'
    ]
    
    try:
        # SSM을 통해 EC2에 명령 전달
        response = ssm_client.send_command(
            InstanceIds=[ec2_instance_id],
            DocumentName='AWS-RunShellScript',
            Parameters={'commands': commands},
        )
        
        # SSM 명령 ID 출력
        command_id = response['Command']['CommandId']
        print(f"SSM 명령 전송 완료. 명령 ID: {command_id}")
    
    except Exception as e:
        print(f"오류 발생: {str(e)}")
    
    return {
        'statusCode': 200,
        'body': json.dumps('Lambda 함수가 성공적으로 실행되었습니다!')
    }
```

### 설명:
- **Lambda**: S3 버킷에 새로운 JAR 파일이 업로드될 때 자동으로 실행됩니다.
- **SSM**: EC2 인스턴스에 연결되어 명령어를 실행합니다.
- **S3 -> EC2 파일 복사**: 업로드된 JAR 파일을 EC2로 복사하고 애플리케이션을 재시작합니다.

### 3. EC2에서 AWS CLI 설치하기 ⚙️

EC2 인스턴스에서 AWS CLI를 설치하는 방법입니다.

```bash
# unzip 패키지 설치
sudo apt install unzip

# AWS CLI 다운로드
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# 다운로드한 파일 압축 해제
unzip awscliv2.zip

# AWS CLI 설치
sudo ./aws/install

# 설치 확인
aws --version
```

### 설명:
1. **unzip 설치**: AWS CLI 설치 파일을 압축 해제하기 위해 unzip을 설치합니다.
2. **AWS CLI 다운로드**: AWS 공식 사이트에서 CLI를 다운로드합니다.
3. **압축 해제 및 설치**: 다운로드한 파일을 압축 해제하고, 설치합니다.
4. **설치 확인**: `aws --version` 명령어로 AWS CLI 설치가 완료되었는지 확인합니다.

### 4. 테스트 및 확인 🧪

S3 버킷에 파일을 업로드한 후 Lambda 함수가 정상적으로 트리거되는지 확인합니다.

### 1. S3 버킷에 파일 업로드하기 ⬆️
- S3 버킷 **ce05-bucket-02**의 **ce05-Oct11Project** 폴더에 새로운 파일을 업로드합니다.

### 2. Lambda 트리거 확인하기 🔍
- AWS Lambda 콘솔에서 **Monitoring(모니터링)** 탭으로 이동하여 최근 Lambda 실행 로그를 확인합니다.
- **CloudWatch Logs**를 통해 Lambda 함수가 실행되었는지 확인하고, 로그를 검토하여 성공적으로 동작했는지 확인합니다.
![image](https://github.com/user-attachments/assets/59b56d25-bfbe-4f88-ad9f-978ae21c3c69)
![image](https://github.com/user-attachments/assets/b11b99b1-5f4b-43de-a109-7bf35b29a273)

### 결과화면
![image](https://github.com/user-attachments/assets/153c0b5d-f4e6-455d-8f2d-e0af0f943f67)
![image](https://github.com/user-attachments/assets/f2717cca-7899-4cbb-8840-4dab2ed15ebf)

## 8. Trouble Shooting
> 1. Jenkins 컨테이너 내부에 AWS CLI가 설치되어 있지 않아서 발생
  ```bash
  docker exec -it <Jenkins 컨테이너 이름> /bin/bash
  
  apt-get update
  
  apt-get install awscli -y
  ```

> 2. Jenkins가  aws credentials를 읽지 못해서 발생
  ```bash
  mkdir -p /var/jenkins_home/.aws
  
  cp ~/.aws/credentials /var/jenkins_home/.aws/credentials
  
  chmod 600 /var/jenkins_home/.aws/credentials
  chown jenkins:jenkins /var/jenkins_home/.aws/credentials
  ```

