# ☸️ Kubernetes 기반 N-tier 서비스 배포 실습

> Spring Boot + MySQL 웹 서비스를, 장애에도 안 죽고 외부에서 접속 가능한 분산 시스템으로 구축한 실습 프로젝트

---

## 📌 프로젝트 개요

단일 서버에서 돌아가던 Spring Boot 앱을 쿠버네티스 클러스터 위에 올려,  
**고가용성(HA)**, **외부 접속**, **설정 분리**를 갖춘 미니 클라우드 서비스 수준의 구조로 만든 실습입니다.

---

## 🖥️ 클러스터 구성

| 노드 | 역할 | 비고 |
|------|------|------|
| server01 | Master Node | 클러스터 제어 |
| server02 | Worker Node | 워크로드 실행 |
| server03 | Worker Node | 워크로드 실행 |

---

## 📁 파일 구성

```
k8s-app/
├── 01-secret.yaml        # DB 비밀번호 등 민감 정보
├── 02-configmap.yaml     # DB 주소 등 일반 설정
├── 03-mysql.yaml         # MySQL Deployment & Service
├── 04-spring-app.yaml    # Spring Boot Deployment & Service
└── 05-ingress.yaml       # 외부 접속 라우팅
```

---

## 🏗️ 아키텍처

```
브라우저
  │
  ▼
myapp.local (hosts 파일 설정)
  │
  ▼
NodePort (30225)
  │
  ▼
Ingress (Nginx Controller)
  │
  ▼
spring-service (ClusterIP)
  │
  ├── spring-app Pod 1  ─┐
  └── spring-app Pod 2  ─┤ (Load Balancing)
                         │
                         ▼
                    db:3306 (ClusterIP)
                         │
                         ▼
                    mysql Pod
```

---

## ⚙️ 주요 구성 요소 설명

### 1. Secret & ConfigMap — 설정 분리

```yaml
# 01-secret.yaml
# DB 비밀번호 등 민감 정보를 코드와 분리하여 관리
kind: Secret

# 02-configmap.yaml
# DB 주소, 포트 등 일반 설정 관리
kind: ConfigMap
```

> 코드와 설정을 분리하는 것이 실무의 핵심 (IaC 개념)

---

### 2. MySQL — DB 컨테이너

```yaml
# 03-mysql.yaml
kind: Deployment
replicas: 1

---
kind: Service
# 고정된 이름 'db'로 Pod에 접근 가능
# → Spring에서 jdbc:mysql://db:3306/fisa 로 접속
```

---

### 3. Spring Boot App — 앱 컨테이너

```yaml
# 04-spring-app.yaml
kind: Deployment
replicas: 2    # 고가용성(HA)을 위한 2개 복제

---
kind: Service
```

> Pod가 2개이므로 하나가 죽어도 서비스 유지 → **고가용성(HA)**

---

### 4. Ingress — 외부 입구 + 라우터

```yaml
# 05-ingress.yaml
# myapp.local → spring-service 로 라우팅
kind: Ingress
```

> Nginx Ingress Controller가 트래픽을 받아 Pod로 부하 분산

---

## 🔑 핵심 개념 정리

| 개념 | 역할 |
|------|------|
| **Pod** | 실제 실행되는 앱 단위 |
| **Deployment** | Pod를 몇 개 실행할지 관리 |
| **Service** | Pod를 찾는 고정 이름 제공 (IP 대신 이름으로 접근) |
| **Ingress** | 외부 → 내부로 들어오는 단일 입구 |
| **ConfigMap** | 일반 설정 분리 관리 |
| **Secret** | 민감 정보(비밀번호 등) 분리 관리 |

---

## 🌐 외부 접속 방법

1. **hosts 파일 설정** (도메인 흉내)
   ```
   127.0.0.1  myapp.local
   ```

2. **포트포워딩** (로컬 PC → VM 연결)

3. **브라우저 접속**
   ```
   http://myapp.local:30225
   ```

---

## ✅ 실습 완료 체크리스트

- [x] Spring Boot Docker 이미지 빌드 → Docker Hub 업로드
- [x] MySQL 컨테이너화
- [x] Secret / ConfigMap으로 설정 분리
- [x] MySQL Deployment & Service 배포
- [x] Spring Boot Deployment (replicas: 2) & Service 배포
- [x] Ingress 설정으로 외부 접속 구성
- [x] DB 연결 및 테이블 생성 확인
- [x] API 호출 및 데이터 조회 성공

---

## 🏆 구현된 실무 수준 기능

| 기능 | 설명 |
|------|------|
| **N-tier 구조** | App / DB 계층 분리 |
| **고가용성 (HA)** | replicas: 2로 Pod 이중화 |
| **부하 분산** | Nginx Ingress Controller가 자동 분산 |
| **외부 접속** | Ingress + NodePort 구성 |
| **설정 분리** | ConfigMap & Secret 사용 |
| **장애 대응** | Pod 재시작 및 복제본 유지 |

---

## 💡 배운 점

> "단일 서버에서 돌아가던 앱을, 장애에도 안 죽고 외부에서 접속 가능한 분산 시스템으로 만든 것"

이 실습을 통해 단순히 앱을 실행하는 것을 넘어,  
**인프라를 코드로 관리(IaC)** 하고 **클라우드 네이티브 서비스를 설계**하는 경험을 쌓았습니다.
