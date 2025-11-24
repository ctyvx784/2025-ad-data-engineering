# knock Mongo DB 동기화

담당자: 강영선
상태: 닫힘
우선순위: 보통
서비스: Model Light
ID: JOB-325
시작 예정일: 2025년 9월 22일
작업 구분: Data, backend, server
작업 유형: 기능 개발

# 상세정보

- [노크 몽고디비 로컬화 정리 문서](https://www.notion.so/2719fb013f3580b68ab3da5c8b20cde2?pvs=21) 를 기반으로 knock Mongo DB의 1일전 데이터를 매일 다운로드 하여 NKS의 몽고 디비로 삽입

# 작업 계획

- [x]  NKS 몽고 디비 서버 생성
- [x]  몽고 디비 다운로드 job 생성
    - [x]  외부 몽고 디비 연결
    - [x]  NKS 몽고 디비 연결
    - [x]  외부 몽고 디비 다운로드 - 아마 배치로 끊어서 가져와야함
    - [x]  다운로드 데이터 upsert
- [x]  백업 스토리지 생성

# 작업 결과

[NKS MongoDB 서버 구성](https://www.notion.so/NKS-MongoDB-2739fb013f35802388cac9107a62d02d?pvs=21) 

[서비스 배포](https://www.notion.so/2779fb013f358014a5d3e6a9ec7bb965?pvs=21)

# 관련 파일

[https://github.com/addddev/k8s-data-api/pull/165](https://github.com/addddev/k8s-data-api/pull/165)

[https://www.notion.so](https://www.notion.so)

# NKS MongoDB 서버 구성

## 1. 목적

- 외부 MongoDB 서버에서 매일 하루치 데이터**(약 15GB)**를 다운로드하여 NKS 내 MongoDB 서버에 적재
- 같은 VPC 내 서비스가 목적 DB를 조회하여 사용
- 배치 기반 데이터 적재 및 내부 조회 전용 목적

---

## 2. MongoDB 서버 구성 옵션

### Single Replica Set vs Sharding

| 항목 | Single Replica Set | Sharding |
| --- | --- | --- |
| 구성 | Primary 1 + Secondary 2 (3노드) | 여러 Replica Set을 샤드로 묶음, Config Server + Mongos 필요 |
| 장점 | 단순 구조, HA 확보, 읽기 분산 가능, 운영/관리 용이 | 데이터 수평 확장 가능, 쓰기/읽기 트래픽 분산 |
| 단점 | 데이터 용량 증가 시 단일 노드 부담, 쓰기 집중 | 아키텍처 복잡, 샤딩 키 설계 중요, 운영 난이도 증가 |
| 적합한 경우 | 하루 단위 배치, read-heavy workload, 데이터 수 TB 이하 | 데이터 수십 TB 이상, 쓰기 트래픽 폭발, 고도의 수평 확장 필요 |

**결론:** 하루치 배치 + read-heavy workload 목적이므로 **Single Replica Set** 권장

---

## 3. 데이터 적재 및 동기화 방식

### 선택한 방식: **증분 Upsert**

- **원리**: 외부 MongoDB에서 전일 데이터(`createdAt`/`updatedAt`)를 추출 → NKS MongoDB에 **bulk upsert**
- **장점**:
    - 멱등성 보장(재실행해도 동일 결과)
    - 서비스 무중단
    - 단순/안정적 운영

### 구현 도구

- **Node.js 스크립트** + **K8s CronJob**
- 일일 01:00 실행, 전일 데이터만 업서트

---

## 4. 서버 스펙 산정

### 하루 적재량: 15GB

- **디스크**: 30일 보존 기준 → 데이터 450GB + 인덱스 30~50% + 여유분 = **1~1.5TB NVMe SSD**

---

## 5. 백업 및 아카이빙 전략

- **핫 데이터 보존**: 30일
- **콜드 아카이브**: 30일 초과 데이터는 Object Storage로 일일 덤프 후 삭제
- **백업 주기**:
    - 주 1회 전체 백업 (full dump)
    - 매일 증분/스냅샷 백업
- **복구 전략**: 최근 full + 이후 일일 증분을 조합

---

## 6. 결론

- NKS 내 3노드 Replica Set 구축
- 증분 Upsert 방식으로 외부 MongoDB 데이터를 일일 적재
- 16GB RAM + 2 CPU로 시작, 부하에 따라 확장
- 30일 핫데이터 + 일일 아카이브/백업으로 용량 관리