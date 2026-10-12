# Jenkins 기반 Docker CI/CD 파이프라인 구축

GitHub, Jenkins, Docker, Docker Hub를 연동하여 소스 코드 변경 감지부터 Docker 이미지 빌드, 이미지 업로드, 테스트 서버 자동 배포 및 헬스 체크까지 구현한 개인 DevOps 프로젝트입니다.

## 1. 프로젝트 개요

### 프로젝트 목표

- Jenkins Pipeline 기반 CI/CD 환경 구축
- GitHub 저장소와 Jenkins 연동
- Docker 이미지 빌드 및 Docker Hub Push 자동화
- 테스트 컨테이너 자동 배포 구현
- 배포 후 서비스 정상 여부를 확인하는 헬스 체크 구성

### 프로젝트 환경

| 항목 | 구성 |
|---|---|
| OS | Rocky Linux 8.10 |
| CI/CD | Jenkins |
| Container | Docker |
| Source Control | GitHub |
| Image Registry | Docker Hub |
| Web Server | Nginx |
| Pipeline | Jenkinsfile (Declarative Pipeline) |

## 2. 아키텍처

```text
Developer
    |
    | Git Commit / Push
    v
GitHub Repository
    |
    | Poll SCM
    v
Jenkins Pipeline
    |
    +-- Checkout
    |
    +-- Docker Image Build
    |
    +-- Docker Hub Push
    |
    +-- Deploy to Test Server
    |
    +-- HTTP Health Check
    |
    v
Docker Container
    |
    v
Nginx Web Application
```

### 서비스 구성

| 서비스 | 포트 | 역할 |
|---|---:|---|
| Jenkins | 8081 | CI/CD 파이프라인 실행 및 관리 |
| 테스트 웹 서비스 | 8082 | Docker 컨테이너 기반 웹 서비스 |
| 기존 웹 서비스 | 8080 | 기존 서비스 유지 |

Jenkins 컨테이너에 Docker CLI를 설치하고 호스트의 Docker 소켓을 연결하여, Jenkins Pipeline에서 서버의 Docker Engine을 사용하도록 구성했습니다.

## 3. CI/CD 파이프라인

### 1) Source Checkout

Jenkins에서 GitHub 저장소의 소스 코드와 Jenkinsfile을 가져옵니다.

### 2) Docker Image Build
<img width="1219" height="743" alt="image" src="https://github.com/user-attachments/assets/fefece19-6873-4a50-80b5-616718bfa609" />


Dockerfile을 기반으로 Nginx 웹 애플리케이션 이미지를 빌드합니다.

- 빌드 번호를 활용한 버전 태그 생성
- `latest` 태그 생성

### 3) Docker Hub Push

Jenkins Credentials에 등록한 Docker Hub 인증 정보를 사용해 이미지를 업로드합니다.

이미지 저장소:

`dydcjsrjaror/jenkins-cicd-demo`

### 4) Automated Deployment

Docker Hub에 업로드한 이미지를 Pull한 후 테스트 컨테이너를 교체합니다.

- 테스트 컨테이너 이름: `jenkins-cicd-demo-test`
- 호스트 포트: `8082`
- 컨테이너 포트: `80`

### 5) Health Check

컨테이너 내부에서 Nginx HTTP 응답을 확인합니다. 정상 응답이 확인되면 배포 성공으로 처리하며, 헬스 체크 실패 시 기존 이미지로 복구를 시도하도록 구성했습니다.

## 4. 주요 구현 내용

### Jenkins Pipeline

Jenkinsfile에서 다음 단계를 정의했습니다.

1. Checkout
2. Build Docker Image
3. Docker Hub Push
4. Verify Image
5. Deploy to Test Server
6. HTTP Health Check

### Poll SCM 자동 빌드

Jenkins의 Poll SCM 기능을 이용해 GitHub 저장소의 변경 사항을 주기적으로 확인하도록 구성했습니다.

설정:

```text
H/2 * * * *
```

GitHub에 변경 사항을 Push하면 Jenkins가 다음 확인 주기에 변경 사항을 감지하고 Pipeline을 실행합니다.

### Docker Credentials 관리

Docker Hub 인증 정보를 Jenkins Credentials로 등록하고 Pipeline에서 참조하도록 구성했습니다. 인증 토큰을 Jenkinsfile에 직접 작성하지 않도록 했습니다.
<img width="1132" height="422" alt="image" src="https://github.com/user-attachments/assets/c8e0adad-3674-4925-bfba-25417ac7e4f4" />

## 5. 실행 결과

- [x] Jenkins와 GitHub 저장소 연동
- [x] Poll SCM 변경 감지 및 자동 빌드
- [x] Docker 이미지 빌드
- [x] Docker Hub 이미지 Push
- [x] Docker 이미지 Pull
- [x] 테스트 컨테이너 자동 배포
- [x] HTTP 헬스 체크 성공
- [x] 배포 실패 시 이전 이미지 복구 로직 구성

테스트 웹 서비스:
<img width="472" height="172" alt="image" src="https://github.com/user-attachments/assets/d266e2ee-a55a-4fe8-9192-5711bd3d4cbe" />

`http://<SERVER_PUBLIC_IP>:8082`

## 6. Troubleshooting

### 문제 1. Jenkins 컨테이너에서 Docker 명령 실행 불가

**원인**

Jenkins 컨테이너에 Docker CLI가 설치되어 있지 않아 Docker 명령을 실행할 수 없었습니다.

**해결**

Jenkins 기반 커스텀 이미지를 생성하고 Docker CLI를 설치했습니다. 또한 호스트 Docker 소켓을 마운트하고 적절한 그룹 권한을 부여했습니다.

### 문제 2. Docker Hub Push 인증 실패

**원인**

Docker Hub 인증 정보 및 GitHub 저장소 접근 토큰의 권한 설정에 문제가 있었습니다.

**해결**

Docker Hub 인증 정보는 Jenkins Credentials에 등록하고, GitHub 토큰은 대상 저장소에 필요한 권한을 부여하여 해결했습니다.

### 문제 3. 배포 후 헬스 체크 실패

**원인**

호스트의 `127.0.0.1:8082`를 통한 연결 확인이 실패했지만, 공인 IP와 컨테이너 내부에서는 정상적으로 HTTP 응답이 반환됐습니다.

**해결**

호스트 루프백 주소 대신 컨테이너 내부의 Nginx HTTP 응답을 확인하도록 헬스 체크 방식을 변경했습니다.

## 7. 개선 과제

- GitHub Webhook을 이용한 이벤트 기반 빌드 구성
- 테스트 결과에 따른 배포 승인 및 배포 단계 개선
- 배포 실패 및 롤백 시나리오 검증
- Jenkins 접근 제어 및 Docker 소켓 권한 최소화
- Prometheus 및 Grafana를 활용한 배포 후 모니터링 연계

## 8. 프로젝트를 통해 학습한 내용

- Jenkins Declarative Pipeline 작성 및 실행
- GitHub 기반 소스 코드 변경 관리
- Docker 이미지 빌드 및 레지스트리 Push/Pull
- Jenkins Credentials를 활용한 인증 정보 관리
- 컨테이너 교체 기반 배포 자동화
- HTTP 헬스 체크 및 배포 실패 처리
- Linux 서버의 포트 및 Docker 네트워크 트러블슈팅

## Repository

GitHub: https://github.com/dpskst/Jenkins-CI-CD
