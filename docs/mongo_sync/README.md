# Knock Mongo → NKS MongoDB Daily Synchronization

## 📌 Overview
외부에 위치한 Knock MongoDB 인스턴스에 의존하던 구조를 개선하기 위해,  
NKS(Kubernetes) 클러스터 내부에 **전용 MongoDB Replica Set**을 구축하고  
매일 하루치 데이터를 동기화하는 배치 파이프라인을 설계·구현했습니다.

## 🎯 Goals
- 외부 MongoDB 장애/네트워크 이슈로부터 서비스 보호
- 내부 서비스들이 같은 VPC 내 MongoDB에 빠르게 접근할 수 있도록 개선
- 일일 배치 기반으로 안정적인 데이터 레이크 구성

## 🧱 Responsibilities
- NKS 내 MongoDB Replica Set(Primary 1 + Secondary 2) 구성
- 외부 MongoDB ↔ NKS MongoDB 연결 설정
- 하루치 데이터(예: ~15GB 수준)를 Batch로 내려받아 **Bulk Upsert**하는 Job 구현
- Kubernetes CronJob 스케줄링 및 모니터링
- Object Storage 기반 백업 버킷 생성 및 운영 전략 수립

## 🛠 Implementation

### 1) MongoDB Cluster Design
- Single Replica Set 구조 선택 (Primary 1 + Secondary 2)
- 하루치 배치 적재 + read-heavy 워크로드에 최적화
- 30일 핫데이터 보존을 기준으로 디스크 용량 산정 (데이터 + 인덱스 + 여유분)

### 2) Data Sync Job
- Node.js 스크립트로 외부 MongoDB에서 전일 데이터 조회
- Cursor 기반 Batch 읽기 후 내부 MongoDB에 Bulk Upsert
- 멱등성을 보장하여 재실행 시에도 동일 결과 유지
- K8s CronJob으로 매일 새벽 실행

### 3) Backup & Archiving
- 30일 초과 데이터는 Object Storage에 덤프로 백업 후 삭제
- 주 1회 Full Backup + 일일 증분/스냅샷 전략 수립
- 장애 시 최근 Full + 이후 증분으로 복구 가능하도록 구조 설계

## 📊 Impact / KPI
- 🔐 외부 MongoDB 장애/지연에 대한 의존도 제거
- ⚡ 내부 서비스에서 MongoDB에 대한 접근 지연 감소
- 🧩 30일 핫데이터 + 아카이브 전략으로 스토리지 안정성 확보
