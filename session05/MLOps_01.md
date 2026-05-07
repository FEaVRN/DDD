---
page-title: MLOps 1차 스터디
aliases: MLOps 1차 스터디 - 문태훈
url: http://172.16.1.212:2026
tags:
  - MLOps
  - 모니터링
date: 2026-04-30 13:39
---
# MLOps 스터디
## kubernetes 기반
- kubeflow
	- K8s 위에서 돌아가는 가장 유명한 MLOps 플랫폼
	- 주피터 노트북, 파이프라인 설계, 모델 서빙까지 통합 관리하며 GPU 할당을 자동화
- volcano
	- 여러 GPU를 동시에 사용하는 **Gang Scheduling**이 가능해져 대규모 분산 학습에 유리
- run:ai
	- 하나의 GPU를 여러 명에게 쪼개주는 **Fractional GPU** 기능과 우선순위 기반의 자원 회수(Preemption) 기능이 매우 강력
## HPC
- slurm
	- 스케줄링 능력이 탁월합니다. 컨테이너 없이도 동작하며, 연구실 환경에서 선호
- ray
	- LLM 학습으로 가장 각광받는 프레임워크
	- Python 코드 몇 줄로 여러 서버의 GPU를 마치 하나의 메모리처럼 다룰 수 있게 해줌
	- **K8s** 위에서도, 단독 서버들 위에서도 잘 돌아감
## 경량화 및 편의성
- HashiCorp Nomad
	- k8s 보다 훨씬 가볍고 설치가 간단
	- Docker, 일반 프로세스도 GPU 스케줄링 관리 편함
- ClearML / WandB
	- 실험 기록(Experiment Tracking) 툴
	- 자체적인 Orchestration 기능 제공
	- 에이전트 하나만 설치하면 중앙 서버에서 각 서버로 학습 태스크를 던지고 GPU 점유 현황을 볼 수 있음

## 추천 솔루션
| 요구 사항                                                   | 추천 솔루션                      |
| ----------------------------------------------------------- | -------------------------------- |
| "정석대로, 가장 확장성 있게 구축하고 싶다"                  | Kubernetes + NVIDIA GPU Operator |
| "데이터 과학자들이 GUI에서 클릭만으로 GPU를 쓰게 하고 싶다" | Kubeflow 또는 Run:ai             |
| "컨테이너 학습보다는 분산 학습 코드 실행이 우선이다"        | Ray                              |
| "K8s는 너무 어렵다. 가볍게 중앙에서 작업만 던지고 싶다"     | ClearML 또는 Nomad               |
### 추천 상세
| 비교 항목     | Ray (추천)                  | Kubeflow                   | ClearML                 |
| ------------- | --------------------------- | -------------------------- | ----------------------- |
| 핵심 타겟     | 분산 학습 & 고성능 서빙     | 워크플로우 파이프라인 관리 | 실험 기록 & 간단한 배치 |
| Python 친화도 | 매우 높음 (코드 중심)       | 보통 (YAML/설정 중심)      | 높음 (SDK 중심)         |
| LMM 대응력    | 최상 (vLLM, DeepSpeed 연동) | 보통                       | 낮음                    |
| 난이도        | 중 (Python만 알면 쉬움)     | 상 (K8s 지식 필수)         | 하 (도입이 매우 빠름)   |



## Monitoring
### Ray Dashboard(추천)
- Ray를 설치하면 기본으로 포함되는 웹 UI
- ray start 시 자동 활성화
- job 상태 확인, 로그 확인, 노드별 자원 사용량 실시간 모니터링
- 확인 가능 지표
	- 노드별 CPU/GPU/RAM 사용률  
	- 개별 Worker 프로세스의 상태 및 에러 로그  
	- Ray Serve 대시보드 (API 서비스 상태 및 지연 시간)
### Prometheus + Grafana(추천)
- Ray Metrics Export
- Ray 공식 Grafana 템플릿 제공
- 여러 대의 서버 지표 통합해서 볼 수 있음
- Alert 서비스 제공
- 확인 가능 지표
	- 서버 여러 대의 GPU/CPU/Temp 등
### Zabbix(비추천)
- 레거시 서버 관리에는 추천하나, AI/ML 환경에서는 비추천
- short-lived, auto-scaling 자원 추적의 취약
- 실시간 클러스터 프로세스 추적하기에는 설정이 너무 무겁고 느림

## MLOps 마무리
MLOps 관점에서는 다음 3가지가 구현 되어야 함
- Quotas(할당량)
	- 수리응용 1팀은 4개, 수리응용 2팀은 3개, 자동화팀은 3개
- Fractional GPU
	- 벤치마킹이나 가벼운 테스트 시 GPU 1개를 0.1개씩 10명이 나눠 쓰게 하여 낭비 줄이기
- Monitoring
	- 단순히 살아있는지가 아니라, GPU 온도와 Memory Leak 여부를 실시간 대시보드로 모니터링
# AI 연구소 GPU 서버
total 15대
- a100
- b200번대
	- b200-1
	- b200-2
- gpu200번대
	- 201
	- 202
	- 203
	- 204
	- 205
	- 206
	- 207
	- 208
- md 100번대
	- master
	- 101
	- 102
	- 103
	- 104
	- 105
	- 106
	- 107
	- 108



# 부록
- https://docs.google.com/document/d/1GB9Sa6hsjpyXCapIx2tkmR93f_CVMMeXofiXUYRIeo8/edit?tab=t.p0n1fh87wx3x#heading=h.d4e4ehvlsm6a
- https://github.com/huns-kim/gpu-monitor
- http://172.16.1.212:2026/
- http://localhost:2026/