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


## 아키텍처 구조
![image](https://github.com/user-attachments/assets/25d05527-ef0f-47cc-8dda-49a0a8f39a83)

## 1. Jenkins와 GitHub 간 웹훅 연결
![image](https://github.com/user-attachments/assets/4d76e7e6-05ab-4ac2-8375-8dc28ce51c39)
도커에 jenkins를 설치 후, jenkins를 8888포트를 사용하여 접속이 가능하도록 설정합니다.

