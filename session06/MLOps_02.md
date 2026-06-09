# MLOps 2차 스터디: SLURM 클러스터 구축 사례 연구

> **발표자**: 문태훈  
> **작성 일시**: 2026-06-04
> **대상**:MLOps 2차 스터디

---

## 📋 목차

1. [MLOps와 SLURM](#1-mlops와-slurm)
2. [초기 아키텍처](#2-초기-아키텍처)
3. [하드웨어 및 네트워크 명세](#3-하드웨어-및-네트워크-명세)
4. [Master 노드 설계](#4-master-노드-설계)
5. [GPU Worker 노드 설계](#5-gpu-worker-노드-설계)
6. [실제 구축 방식 (Docker & 호스트 하이브리드)](#6-실제-구축-방식)
7. [모니터링 및 관찰성](#7-모니터링-및-관찰성)
8. [확장 전략](#8-확장-전략)
9. [배운 점 & 권장사항](#9-배운-점--권장사항)

---

## 1) MLOps와 SLURM

### MLOps란?

**MLOps**는 Machine Learning Operations의 약자로, 머신러닝 모델을 **개발 → 배포 → 모니터링 → 재생산**하는 전체 생명주기를 체계적으로 관리하는 분야입니다.

```
Data → Model Training → Deployment → Monitoring → Feedback
         ↓
      (GPU Job Scheduling)
```

### SLURM의 역할

**SLURM** (Simple Linux Utility for Resource Management)은:
- **고성능 컴퓨팅(HPC)** 및 **AI 연산 클러스터**용 작업 스케줄러
- 여러 GPU/CPU 리소스를 여러 사용자/작업에 공정하게 배분
- 작업 큐, 우선순위, 자원 격리 관리

### MLOps에서 SLURM이 필요한 이유

1. **리소스 공유**: 여러 ML 팀이 제한된 GPU 리소스를 경쟁적으로 사용
2. **작업 자동화**: 하이퍼파라미터 튜닝, 분산 훈련 작업 자동 제출
3. **추적성(Traceability)**: 어느 사용자가 언제 어떤 리소스를 얼마나 썼는지 기록
4. **격리(Isolation)**: 한 프로젝트의 작업이 다른 프로젝트 방해 차단

---

## 2) 초기 아키텍처

### 2.1 목표 구성

초기 **1 Master + 1 GPU Worker** 구성으로 시작해 최대 **20대 GPU Worker**까지 확장 예정.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Internal Network (172.16.1.0/24)            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────┐    ┌──────────────────────────┐   │
│  │   Master                 │    │   GPU Worker             │   │
│  │  172.16.1.183            │    │  172.16.1.205            │   │
│  ├──────────────────────────┤    ├──────────────────────────┤   │
│  │ - slurmctld (6817)       │    │ - slurmd (6818)          │   │
│  │ - slurmdbd (6819)        │    │ - munge                  │   │
│  │ - mariadb (3306)         │    │ - NVIDIA Driver/CUDA     │   │
│  │ - munge                  │    │ - 6x A100 or RTX         │   │
│  │ (Docker: Core/Monitor)   │    │ (Host Service)           │   │
│  └──────────────────────────┘    └──────────────────────────┘   │
│          │                               │                      │
│          └───────────────┬───────────────┘                      │
│                    (munge 동기화)                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

향후 확장 계획:
- GPU Worker: 172.16.1.206 ~ 172.16.1.225 (20대까지)
- 필요시 Login node, Backup Master 추가
```

### 2.2 컴포넌트 분담

| 컴포넌트 | 역할 | 위치 | 기술 |
|---------|------|------|------|
| **slurmctld** | 작업 스케줄링 및 리소스 관리 | Master | Docker (권장) |
| **slurmdbd** | 작업 이력/회계 DB 접근 | Master | Docker (권장) |
| **MariaDB** | 회계 데이터 저장 | Master | Docker |
| **slurmd** | Worker 측 에이전트, 작업 실행 | GPU Worker | Host Service (권장) |
| **munge** | 노드 간 인증 | Master/Worker | Host Service |
| **prometheus-slurm-exporter** | 메트릭 수집 | Master | Docker (내장) |
| **Grafana** | 대시보드 | Master | Docker (선택) |
| **slurm-web** | 웹 UI | Master | Docker (선택) |

### 2.3 데모
- slurm-web
- ![img_4.png](img_4.png)
- ![img_5.png](img_5.png)
- ![img_8.png](img_8.png)
- ![img_7.png](img_7.png)
- jupyterlab
- ![img_2.png](img_2.png)
- slurm-submit-ui
- ![img_1.png](img_1.png)
- ![img_6.png](img_6.png)
- OpenOndemand
- ![img_3.png](img_3.png)
---

## 3) 하드웨어 및 네트워크 명세

### 3.1 Master 서버 사양

초기 구성(1-5개 노드 연결):

| 항목 | 최소 | 권장 |
|------|------|------|
| **CPU** | 4 vCPU | 8 vCPU |
| **메모리** | 8 GB | 16 GB |
| **스토리지** | 100 GB SSD | 200 GB SSD |
| **네트워크** | 1 Gbps | 1 Gbps |
| **OS** | Ubuntu 22.04 LTS | Ubuntu 22.04 LTS |

**역할**:
- 모든 작업 스케줄링 결정
- 재계 DB 관리
- 사용자 웹 포털
- 모니터링 시스템

### 3.2 GPU Worker 서버 사양

실제 ML 훈련/추론 작업 실행:

| 항목 | 최소 | 권장 |
|------|------|------|
| **CPU** | 16 vCPU | 32 vCPU |
| **메모리** | 64 GB | 128 GB+ |
| **GPU** | 1장 (VRAM 16GB) | 6x A100 or RTX (VRAM 48GB) |
| **스토리지** | 500 GB NVMe | 1 TB NVMe |
| **네트워크** | 1 Gbps | 10 Gbps (데이터셋/모델 전송) |
| **OS** | Ubuntu 22.04 LTS | Ubuntu 22.04 LTS |

**역할**:
- 실제 GPU 작업 실행
- 로컬 캐시/모델 저장
- 데이터 전처리

### 3.3 네트워크 정책

#### 3.3.1 포트 정책

```
Master ←→ Worker 통신:

  6817/tcp  ← slurmctld (Job submission)
  6818/tcp  ← slurmd (Hearthbeat, job launches)
  6819/tcp  ← slurmdbd (Accounting DB queries)
  3306/tcp  ← MariaDB (Internal only, NOT exposed)
  6820/tcp  ← slurmrestd (REST API, optional)
  8080/tcp  ← slurm-web UI (optional)
  9090/tcp  ← Prometheus (monitoring, optional)
  3000/tcp  ← Grafana (monitoring, optional)
```

#### 3.3.2 보안 정책

1. **내부망 격리**: 모든 통신은 `172.16.1.0/24` 내부
2. **DB 격리**: MariaDB(3306)은 Master 내부만, 외부 접근 차단
3. **인증**: munge 토큰 기반 노드 간 상호 인증
4. **방화벽**: Master ↔ Worker 필수 포트만 허용

#### 3.3.3 호스트명 매핑

내부 DNS가 없는 경우 `/etc/hosts` 고정:

```
172.16.1.183    master
172.16.1.205    ai-6gpu-05
# 향후 추가
172.16.1.206    ai-6gpu-06
...
```

---

## 4) Master 노드 설계

### 4.1 아키텍처 결정: Docker vs Host

#### 권장안: 부분 Docker화 (Hybrid)

**Master의 핵심 서비스는 Docker에서 실행**:
- `slurmctld` (Controller)
- `slurmdbd` (Accounting daemon)
- `mariadb` (DB)
- Monitoring 스택 (Prometheus, Grafana, exporter)

**장점**:
- 배포 자동화 (`docker-compose`)
- 환경 일관성 (OS 버전 독립)
- 스케일링 간편 (replicas 추가 등)
- 로그 중앙집중화

**단점**:
- 네트워킹 추가 복잡도
- 초기 학습곡선

#### 비권장안: 완전 Host 서비스

초기에는 상대적으로 단순하지만:
- OS별로 다른 패키지 버전 관리 필요
- 배포 자동화 어려움
- 확장 시 재설정 필요

### 4.2 Docker 구축 세부사항

#### 4.2.1 docker-compose.yml 구성

```yaml
services:
  slurm-mariadb:
    image: mariadb:11.4
    environment:
      MARIADB_DATABASE: slurm_acct_db
      MARIADB_USER: slurm
      MARIADB_PASSWORD: <strong-password>
    volumes:
      - mariadb_data:/var/lib/mysql
      
  slurmdbd:
    build: ./docker/slurm
    depends_on:
      slurm-mariadb:
        condition: service_healthy
    environment:
      SLURM_DB_HOST: slurm-mariadb
      SLURM_DB_USER: slurm
      SLURM_DB_PASSWORD: <strong-password>
      
  slurmctld:
    build: ./docker/slurm
    depends_on:
      - slurmdbd
    volumes:
      - ./slurm.conf.example:/etc/slurm/slurm.conf:ro
      - ./nodes.custom.conf.example:/etc/slurm/nodes.custom.conf:ro
      - ./secrets/munge.key:/run/secrets/munge.key:ro
```

#### 4.2.2 Munge 키 관리

**Munge**: 클러스터 내 노드 간 안전 통신 보장

```bash
# Master에서 1회 생성 (또는 기존 키 복사)
cd slurm/master
chmod +x init-munge-key.sh
./init-munge-key.sh

# 생성된 키는 secrets/munge.key
# 모든 worker에 동일한 키 배포 필수!!
ls -la secrets/munge.key
# -rw------- 1 root root 1024 ... munge.key
```

#### 4.2.3 환경 설정 (.env)

```bash
# slurm/master/.env
MARIADB_ROOT_PASSWORD=root-change-me-strong
SLURM_DB_NAME=slurm_acct_db
SLURM_DB_USER=slurm
SLURM_DB_PASSWORD=slurm-change-me-strong

GRAFANA_ADMIN_PASSWORD=grafana-change-me
```

#### 4.2.4 기동 절차

```bash
cd slurm/master

# 1. .env 파일 생성 및 비밀번호 수정
cp .env.example .env
nano .env  # 비밀번호 변경

# 2. munge.key 생성/배치
./init-munge-key.sh

# 3. Core 스택 (slurmctld/slurmdbd/mariadb) 기동
docker compose -f docker-compose.yml up -d --build

# 4. (선택) 모니터링 스택 기동
docker compose -f docker-compose.monitoring.yml up -d

# 5. (선택) Slurm Web UI 기동
docker compose -f docker-compose.yml --profile slurm-web up -d

# 6. 상태 확인
docker compose logs -f slurmctld slurmdbd
```

#### 4.2.5 검증

```bash
# Master에서 포트 리스닝 여부
netstat -tlnp | grep 6817

# 컨테이너 상태
docker compose ps

# 로그 확인
docker compose logs slurmdbd | tail -20
```

### 4.3 SLURM 설정 파일 수정 (slurm.conf)

Master의 핵심 설정:

```bash
# slurm/master/slurm.conf.example

# Controller & Accounting
SlurmctldHost=master(172.16.1.183)
DbdAddr=172.16.1.183

# 리소스 선택 방식 (GPU/CPU 분할 지원)
SelectType=select/cons_tres

# GPU를 GRES로 관리
GresTypes=gpu

# Accounting
AccountingStorageType=accounting_storage/slurmdbd
AccountingStorageHost=172.16.1.183
AccountingStoragePort=6819

# Partition (작업 대기열)
PartitionName=gpu Nodes=ai-6gpu-05 Default=YES

# Node 정의
Include /etc/slurm/nodes.custom.conf
```

### 4.4 Worker 노드 정의 (nodes.custom.conf)

```bash
# slurm/master/nodes.custom.conf.example

NodeName=ai-6gpu-05 NodeHostname=ai-6gpu-05 NodeAddr=172.16.1.205 \
  CPUs=32 RealMemory=250000 \
  Gres=gpu:6 \
  State=UNKNOWN
```

**필드 의미**:
- `NodeName`: SLURM 내 논리적 이름 (팀 별칭 가능)
- `NodeHostname`: 실제 서버 hostname
- `NodeAddr`: 실제 서버 IP
- `CPUs`: CPU 코어 수
- `RealMemory`: 실제 메모리 (MB)
- `Gres=gpu:6`: 6개 GPU 선언

---

## 5) GPU Worker 노드 설계

### 5.1 아키텍처 결정: Host Service (Docker 아님)

**GPU Worker는 Host Service로 실행** (Docker 아님):

```
직접 Host OS
├─ slurmd (systemd service)
├─ munge (systemd service)
├─ NVIDIA Driver (kernel module)
└─ CUDA Runtime
```

**이유**:
1. **GPU 직접 제어**: cgroup v2 없이 GPU 리소스 직접 감지
2. **성능**: 컨테이너 오버헤드 없음
3. **장애 진단**: 드라이버 이슈 직접 접근
4. **안정성**: GPU 할당/반환 복잡도 최소화

### 5.2 사전 준비

#### 5.2.1 OS 기본 설정

```bash
# hostname 설정
sudo hostnamectl set-hostname ai-6gpu-05

# /etc/hosts 업데이트
sudo tee -a /etc/hosts > /dev/null <<EOF
172.16.1.183    master
172.16.1.205    ai-6gpu-05
EOF

# 시간 동기화 (Master와 일치 필수!)
sudo systemctl enable --now chrony
chronyc sources
```

#### 5.2.2 NVIDIA 드라이버 설치

```bash
# 드라이버 설치 (환경에 맞춤)
sudo apt update
sudo apt install -y nvidia-driver-550 nvidia-utils

# 검증
nvidia-smi
# GPU list, 온도, 점유율 확인
```

#### 5.2.3 SLURM 패키지 설치

```bash
sudo apt install -y slurmd slurm-client munge
```

### 5.3 Worker 설정 배포

Master에서 준비한 템플릿을 Worker로 복사:

```bash
# Master: slurm/worker/ 아래 템플릿 확인
ls -la slurm/worker/
# - slurm.conf.example
# - nodes.custom.conf.example
# - gres.conf.example
# - deploy-worker.sh
```

```bash
# Worker에서 수행
scp -r /path/to/slurm/worker/* worker-user@172.16.1.205:/tmp/

# 또는 git clone
cd /tmp && git clone <repo> && cd path/to/slurm
```

### 5.4 gres.conf (GPU 선언)

Worker의 GPU 정보를 SLURM에 알림:

```bash
# slurm/worker/gres.conf.example

Name=gpu File=/dev/nvidia0
Name=gpu File=/dev/nvidia1
# ... 
Name=gpu File=/dev/nvidia5
```

자동 생성 스크립트 제공:

```bash
cd slurm/worker
chmod +x deploy-worker.sh
./deploy-worker.sh  # 자동으로 gres.conf 생성
```

### 5.5 최종 배포 및 검증

```bash
# Worker에서
sudo systemctl daemon-reload

# Munge 활성화 (Master의 키와 동일해야 함!)
sudo systemctl enable --now munge

# SLURMD 활성화
sudo systemctl enable --now slurmd

# 상태 확인
systemctl status munge
systemctl status slurmd

# Munge 통신 테스트
munge -n | unmunge
```

```bash
# Master에서 Worker 인식 확인
sinfo
# PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
# gpu*         up   infinite      1   idle ai-6gpu-05

scontrol show node ai-6gpu-05
# NodeName=ai-6gpu-05 Arch=x86_64 CoresPerSocket=16 CPUAlloc=0
# Gres=gpu:6(IDX:0-5)

# GPU 할당 여부 확인
sinfo -o "NodeName Nodes CPUs RealMemory Gres State"
```

---

## 6) 실제 구축 방식: Docker & 호스트 하이브리드

### 6.1 전체 배포 흐름

```
Step 1: Master 준비 (호스트)
├─ hostname 설정: master
├─ /etc/hosts 매핑
├─ Docker/docker-compose 설치
└─ munge.key 준비

Step 2: Master Docker 기동
├─ docker-compose.yml (Core)
│  ├─ slurm-mariadb
│  ├─ slurmdbd
│  └─ slurmctld
├─ docker-compose.monitoring.yml (선택)
│  ├─ prometheus
│  └─ grafana
└─ (선택) slurm-web docker build

Step 3: Worker 준비 (호스트)
├─ hostname 설정: ai-6gpu-05
├─ /etc/hosts 매핑
├─ NVIDIA driver 설치
├─ SLURM 패키지 설치
└─ Master의 munge.key 복사

Step 4: Worker 설정 배포
├─ /etc/slurm/slurm.conf ← worker/slurm.conf.example
├─ /etc/slurm/nodes.custom.conf ← worker/nodes.custom.conf.example 또는 자동생성
└─ /etc/slurm/gres.conf ← worker/gres.conf.example 또는 자동생성

Step 5: Worker 서비스 활성화
├─ munge 시작/활성화
└─ slurmd 시작/활성화

Step 6: 통합 검증
├─ Master에서 sinfo 확인
├─ Worker GPU 인식 확인
└─ 테스트 작업 제출
```

### 6.2 Master Docker 설정 상세

#### 포트 매핑

```yaml
# docker-compose.yml 포트 정리

services:
  slurmctld:
    ports:
      - "6817:6817"   # Job submission
      - "6820:6820"   # REST API (optional)
      - "9341:9341"   # Prometheus exporter
      
  slurmdbd:
    ports:
      - "6819:6819"   # Accounting
      
  slurm-mariadb:
    # 3306은 expose하지 않음 (내부 only)

  prometheus:
    ports:
      - "9090:9090"   # Monitoring (optional)
      
  grafana:
    ports:
      - "3000:3000"   # Dashboard (optional)
      
  slurm-web:
    ports:
      - "10080:80"    # Web UI (optional, --profile slurm-web)
```

#### Volume 전략

```yaml
volumes:
  mariadb_data:        # DB 영속성
  slurm_ctld_state:    # Controller 상태 저장
  slurm_dbd_state:     # DBD 상태 저장
  slurm_logs:          # 로그 중앙집중화
```

---

## 7) 모니터링 및 관찰성

### 7.1 모니터링 아키텍처

```
┌─────────────────────────────────────────────┐
│     Master Docker                           │
├─────────────────────────────────────────────┤
│                                             │
│  slurmctld + prometheus-slurm-exporter      │
│      ↓                                      │
│   Metrics (9341:9341)                       │
│      ↓                                      │
│   Prometheus (9090:9090)                    │
│      ↓                                      │
│   Grafana (3000:3000)                       │
│                                             │
│  [대시보드]                                 │
│  ├─ Node status (idle/allocated/down)      │
│  ├─ GPU utilization per node               │
│  ├─ Job queue depth                        │
│  ├─ CPU/Memory per partition               │
│  └─ Accounting (user/account/time)         │
│                                             │
└─────────────────────────────────────────────┘
```

### 7.2 메트릭 수집

#### prometheus-slurm-exporter

Master의 `slurmctld` 컨테이너 내에 내장:

```bash
# Dockerfile에 포함
RUN apt-get install -y prometheus-slurm-exporter

# entrypoint-slurmctld.sh에서 백그라운드 실행
/usr/bin/prometheus-slurm-exporter --listen-address=0.0.0.0:9341
```

수집 메트릭:
- `slurm_node_count` (상태별)
- `slurm_nodes_total` (전체)
- `slurm_node_cpu_{allocated,idle,total}`
- `slurm_node_memory_{real,allocated,idle,shared}`
- GPU 관련 (GRES) 메트릭

#### Prometheus 스크래핑

```yaml
# docker/prometheus/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'slurm'
    static_configs:
      - targets: ['slurmctld:9341']  # Docker 내부 hostname
```

### 7.3 Grafana 대시보드

자동 프로비저닝:

```yaml
# docker/grafana/provisioning/dashboards/
├── SLURM_Cluster.json
├── SLURM_Nodes.json
├── SLURM_Jobs.json
└── SLURM_Accounting.json
```

**대시보드 예시 내용**:

1. **Cluster Overview**
   - 전체 노드 상태 pie chart
   - 파티션별 리소스 현황

2. **Node Status**
   - 개별 노드 CPU/메모리 usage
   - GPU utilization timeline
   - 온도/전력 소비

3. **Job Queue**
   - 대기 중인 작업 수
   - 평균 대기 시간
   - 우선순위 분포

4. **Accounting**
   - 사용자별 GPU-hours
   - 계정별 비용 추이
   - CPU-hours vs Wall-clock time

---

## 8) 확장 전략

### 8.1 초기(1) → 5대 노드

```
Timeline: 1-2개월

단계:
1. Master 안정화 (1개월)
   └─ 로그 레벨, 타임아웃 튜닝
   └─ 모니터링 대시보드 검증

2. Worker 추가 (병렬 진행)
   └─ 172.16.1.206, 172.16.1.207, ...
   └─ 선형 배치 가능 (munge.key만 동일)

전제:
- Master의 slurmctld CPU < 50%
- DB 트랜잭션 응답 < 100ms
- 네트워크 bandwidth 충분
```

### 8.2 중기(5) → 20대 노드

```
Timeline: 3-6개월

병목 지점:
1. slurmctld 단일 장애점
   └─ Backup controller 도입 검토 (HA)
   
2. MariaDB 성능
   └─ 슬로우 쿼리 최적화
   └─ 복제(Replication) 검토
   
3. 네트워크 토폴로지
   └─ 스위치 업링크 대역폭 점검
   └─ NVMe SSD 스토리지 공유 (NFS/BeeGFS)

아키텍처 변화:
Master (1)
  ├─ slurmctld + backup controller (HA)
  ├─ slurmdbd + replica (Replication)
  └─ MariaDB Primary + Secondary
  
Workers (20)
  ├─ 172.16.1.206 ~ 172.16.1.225
  ├─ 각각 6-8x GPU
  └─ 공유 스토리지 NFS 마운트
```

### 8.3 노드명 명명 규칙

확장을 고려한 SLURM 노드 정의:

```bash
# slurm/master/nodes.custom.conf

# 범위 정의 방식 (권장)
NodeName=ai-6gpu-[05-24] NodeHostname=ai-6gpu-[05-24] \
  NodeAddr=172.16.1.[205-224] \
  CPUs=32 RealMemory=250000 Gres=gpu:6

# OR
# 개별 정의 (조직에 따라)
NodeName=vision-a100-01 NodeHostname=ai-6gpu-05 NodeAddr=172.16.1.205 ...
NodeName=vision-a100-02 NodeHostname=ai-6gpu-06 NodeAddr=172.16.1.206 ...
```

### 8.4 신규 Worker 자동 배포 스크립트

```bash
#!/bin/bash
# deploy-worker.sh

# 현재 서버에서 자동으로 설정 생성
HOSTNAME=$(hostname)
IP=$(hostname -I | awk '{print $1}')
CPUS=$(nproc)
MEMORY=$(free -m | awk '/^Mem:/{print $2}')
GPU_COUNT=$(nvidia-smi --list-gpus | wc -l)

# /etc/slurm/nodes.custom.conf 자동 생성
sudo tee /etc/slurm/nodes.custom.conf > /dev/null <<EOF
NodeName=$HOSTNAME NodeHostname=$HOSTNAME NodeAddr=$IP \
  CPUs=$CPUS RealMemory=$MEMORY Gres=gpu:$GPU_COUNT
EOF

# /etc/slurm/gres.conf 자동 생성
for i in $(seq 0 $((GPU_COUNT-1))); do
  echo "Name=gpu File=/dev/nvidia$i"
done | sudo tee /etc/slurm/gres.conf > /dev/null

# 서비스 재시작
sudo systemctl restart munge slurmd
```

---

## 9) 배운 점 & 권장사항

### 9.1 성공적인 구축을 위한 체크리스트

#### 네트워크 사전 검증

```bash
# 1. 시간 동기화 확인 (가장 중요!)
timediff_master=$(date +%s)
ssh worker "date +%s" | tee /tmp/worker_time
echo "Time diff: $((timediff_master - $(cat /tmp/worker_time)))"
# 결과: 1초 이내여야 함!

# 2. 호스트명 해석
host master
host ai-6gpu-05

# 3. 포트 연결 테스트
nc -zv 172.16.1.183 6817  # Master에서
nc -zv 172.16.1.205 6818  # Worker에서 (역방향)

# 4. Munge 키 동일성 확인
sha256sum /etc/munge/munge.key  # 모든 노드에서 동일해야 함
```

#### SLURM 배포 검증

```bash
# 순서대로 검증
1. $ sinfo
   # 모든 노드가 'idle' 또는 'alloc'인지 확인
   # 'down' 상태가 아닌지 확인

2. $ scontrol show node
   # 각 노드의 상세 정보 (CPU, 메모리, GRES)

3. $ sacct
   # 작업 이력 조회 (DB 연동 확인)

4. Test job:
   $ sbatch --partition=gpu --gres=gpu:1 --cpus-per-task=4 --mem=16G \
       --wrap="nvidia-smi && hostname"
```

### 9.2 주의할 점

#### 1. Munge 키 불일치 (가장 흔한 오류)

```
증상: Worker가 sinfo에 보이지 않음
로그: "Invalid credential from ..." / "auth/munge: ..."

해결:
$ ssh master cat /etc/munge/munge.key | \
  ssh worker 'sudo tee /etc/munge/munge.key' && \
  ssh worker 'sudo systemctl restart munge'
```

#### 2. 시간 불동기

```
증상: 간헐적 "connection timeout"
원인: Master와 Worker의 system time 차이

검증:
$ ntpdate -q 172.16.1.183  # NTP pool 기준
$ chronyc sources          # Chrony 시간소스
```

#### 3. slurm.conf 구문 오류

```
증상: slurmctld가 계속 재시작
로그: "Unable to read config file"

검증:
$ slurmd -C  # 현재 노드의 구성 출력
$ slurm-config  # 빌드된 설정 검증
```

### 9.3 MLOps 관점의 권장 운영

#### 1. 사용자 격리

```bash
# 사용자별 계정정책 수립
sacctmgr add account ml-vision Description="Vision Team"
sacctmgr add account ml-nlp Description="NLP Team"

# 각 계정에 QoS(Quality of Service) 설정
scontrol create QoS Name=express MaxWall=4:00:00 MaxJobs=10
saccmgr create QoS Name=batch MaxWall=24:00:00 MaxJobs=100
```

#### 2. 모니터링 알림 설정

```bash
# Grafana Alert 예시
- Alert: "GPU_Node_Memory_High"
  Condition: GPU 메모리 > 90% for 10 min
  Action: Slack 알림 → On-call engineer

- Alert: "Job_Queue_Deep"
  Condition: 대기 작업 > 50개
  Action: 자동 스케일-아웃 (aws/k8s 연동)
```

#### 3. 작업 추적(Traceability)

```bash
# 모든 ML 작업은 sbatch + --job-name으로 제출
sbatch --job-name="bert-finetune-epoch5" \
       --account=ml-nlp \
       --partition=gpu \
       --gres=gpu:1 \
       train.sh

# 이후 회계 추적
sacct --accounts=ml-nlp --format=JobID,JobName,User,Start,End,GPU,TimeLimitPer
```

### 9.4 향후 고려 아이템


#### 작업 자동화/CI-CD 통합

```
MLflow + SLURM 연동
1. Data Scientist가 mlflow run으로 실험 시작
2. MLflow가 SLURM으로 sbatch 자동 제출
3. 작업 완료 후 결과 자동 기록

장점:
- 작업 제출 자동화
- 하이퍼파라미터 튜닝, 그리드 서치 자동화
- 결과 추적 표준화
```

---

## 📚 추가 학습 자료

### SLURM 공식 문서
- https://slurm.schedmd.com/ (메인)
- https://slurm.schedmd.com/slurm.conf.html (설정 레퍼런스)
- https://slurm.schedmd.com/sbatch.html (작업 제출)

### 관련 기술
- **Prometheus**: 시계열 메트릭 DB (`https://prometheus.io`)
- **Grafana**: 대시보드 플랫폼 (`https://grafana.com`)
- **Munge**: 노드 인증 (`https://dun.github.io/munge`)

### 운영 확장
- **Kubernetes + Kubeflow**: 컨테이너 기반 ML 워크플로우
- **MLflow**: ML 모델 수명주기 관리
- **Ray Cluster**: 분산 컴퓨팅 (hyperparameter tuning)

---

## 🎯 핵심 요약

### "왜 SLURM을 선택했는가?"
0. 현재 AI연구소에서 사용하는 주먹구구식 방식 탈피
![img.png](img.png)

1. **MLOps 필수 요소**
   - 다중 사용자/프로젝트 리소스 공유
   - 작업 자동화 (하이퍼파라미터 튜닝, 분산 훈련)
   - 회계 추적 (GPU-hours, 비용 분석)

2. **검증된 기술**
   - 200+ 대학/기업에서 사용 중
   - 확장성 (노드 1개 → 10,000+개)
   - 커뮤니티 활발

3. **대안 대비**
   - **Kubernetes + Kubeflow**: 초기 진입 난이도 높음, 멀티테넌시 복잡
   - **하드코딩 배쉬 스크립트**: 확장 불가능, 회계 불가능
   - **HTCondor**: 금융/과학용 (ML엔 과도하게 복잡)

### "초기 1 Master + 1 Worker로 시작하는 이유"

- 빠른 DevOps 학습
- 선택적 Docker 도입 (필수 아님)
- 작은 팀/프로젝트 초기화
- 20대까지 선형 확장 가능한 아키텍처

### "다음 단계"

- ✅ 기본 SLURM 구축 (본 문서)
- 📋 Workers 5대까지 배포 
- 🚀 모니터링 대시보드 완성 
- 🔧 NFS 공유 스토리지 도입 
- 🤖 MLflow 연동
- 📊 Kubernetes 마이그레이션 검토

---

