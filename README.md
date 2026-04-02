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


 
## ⚙️ YAML 파일 상세
 
### 01-secret.yaml — 민감 정보 관리
 
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  MYSQL_ROOT_PASSWORD: "root1234"
  MYSQL_DATABASE: "fisa"
  MYSQL_USER: "user01"
  MYSQL_PASSWORD: "user01"
  SPRING_DATASOURCE_PASSWORD: "user01"
```
 
> **핵심:** 비밀번호 등 민감 정보를 코드와 분리하여 관리.  
> `stringData`로 평문 작성 → 쿠버네티스가 자동으로 Base64 인코딩하여 저장.
 
---
 
### 02-configmap.yaml — 일반 설정 관리
 
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  SPRING_DATASOURCE_URL: "jdbc:mysql://db:3306/fisa?serverTimezone=Asia/Seoul&useSSL=false&allowPublicKeyRetrieval=true"
  SPRING_DATASOURCE_USERNAME: "user01"
  SPRING_JPA_HIBERNATE_DDL_AUTO: "update"
```
 
> **핵심:** DB 접속 URL에서 `db`는 MySQL Service 이름. IP 대신 이름으로 접근하므로  
> Pod가 재시작되어 IP가 바뀌어도 영향 없음.
 
---
 
### 03-mysql.yaml — MySQL Deployment & Service
 
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:      # Secret에서 값을 가져옴
                  name: app-secret
                  key: MYSQL_ROOT_PASSWORD
            - name: MYSQL_DATABASE
              valueFrom:
                secretKeyRef:
                  name: app-secret
                  key: MYSQL_DATABASE
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: app-secret
                  key: MYSQL_USER
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secret
                  key: MYSQL_PASSWORD
---
apiVersion: v1
kind: Service
metadata:
  name: db              # Spring이 'db'라는 이름으로 접근
spec:
  selector:
    app: mysql
  ports:
    - port: 3306
      targetPort: 3306
  type: ClusterIP       # 클러스터 내부에서만 접근 가능
```
 
> **핵심:** Service 이름 `db`가 DNS 역할을 함.  
> `jdbc:mysql://db:3306/fisa` 처럼 IP 없이 이름으로 DB 접근 가능.
 
---
 
### 04-spring-app.yaml — Spring Boot Deployment & Service
 
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-app
spec:
  replicas: 2           # Pod 2개 → 고가용성(HA) 확보
  selector:
    matchLabels:
      app: spring-app
  template:
    metadata:
      labels:
        app: spring-app
    spec:
      containers:
        - name: spring-app
          image: yewon0106/step04-empapp:1.0
          imagePullPolicy: Always
          ports:
            - containerPort: 8084
          env:
            - name: SPRING_DATASOURCE_URL
              valueFrom:
                configMapKeyRef:   # ConfigMap에서 값을 가져옴
                  name: app-config
                  key: SPRING_DATASOURCE_URL
            - name: SPRING_DATASOURCE_USERNAME
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: SPRING_DATASOURCE_USERNAME
            - name: SPRING_DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef:      # Secret에서 값을 가져옴
                  name: app-secret
                  key: SPRING_DATASOURCE_PASSWORD
            - name: SPRING_JPA_HIBERNATE_DDL_AUTO
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: SPRING_JPA_HIBERNATE_DDL_AUTO
---
apiVersion: v1
kind: Service
metadata:
  name: spring-service
spec:
  selector:
    app: spring-app
  ports:
    - port: 80          # 외부(Ingress)에서 80으로 접근
      targetPort: 8084  # Pod 내부 포트 8084로 포워딩
  type: ClusterIP
```
 
> **핵심:** `replicas: 2`로 Pod가 2개 실행됨.  
> 하나가 죽어도 나머지 하나가 요청을 처리 → **고가용성(HA)**.  
> 환경변수를 ConfigMap과 Secret에서 주입하여 이미지에 설정값이 포함되지 않음.
 
---
 
### 05-ingress.yaml — 외부 입구 & 라우터
 
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: spring-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: spring-service
                port:
                  number: 80
```
 
> **핵심:** `myapp.local`로 들어오는 모든 요청(`/`)을 `spring-service:80`으로 라우팅.  
> Nginx Ingress Controller가 Pod 2개로 자동 부하 분산.
 
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
