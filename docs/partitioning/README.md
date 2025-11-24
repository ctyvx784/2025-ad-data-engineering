# MySQL Log Partitioning & Automation

## 📌 Overview
광고 효과 로그(ad_effect_log)가 하루 4~6M건씩 누적되는 환경에서 단일 테이블 구조로는 성능과 운영이 한계에 도달했습니다.  
이를 해결하기 위해 **MySQL 파티셔닝 기반 로그 저장 구조와 자동 로테이션 시스템**을 설계·구현했습니다.

## 🎯 Goals
- 일 4–6M 로그 처리에도 안정적인 쿼리 성능 유지
- 파티션 생성·백업·삭제까지 전체 라이프사이클 자동화
- 운영 중단 없이 신규 파티션 테이블로 전환

## 🧱 Responsibilities
- ad_effect_log_v2 신규 파티션 테이블 스키마 설계
- RANGE COLUMNS(created_at) 기반 파티션 전략 수립
- 파티션 생성/백업/삭제 스크립트 구현 및 Cron 연동
- 기존 테이블 → 파티션 테이블 마이그레이션 스크립트 구현
- Go Ingestion 서비스 Dual-write 적용
- Prisma, TypeORM, Atlas 등 스키마 관리 도구 조사 및 제안

## 🛠 Implementation

### 1) Partitioned Table Design
- 기존 PK 제약(MySQL 파티션 시 PK 포함 필수)을 분석해, 파티션에 적합한 **전용 테이블(ad_effect_log_v2)** 신규 설계
- created_at 컬럼을 기준으로 **일 단위 RANGE 파티션** 구성

```sql
PARTITION BY RANGE COLUMNS (created_at) (
  PARTITION p20250901 VALUES LESS THAN ('2025-09-02'),
  PARTITION p20250902 VALUES LESS THAN ('2025-09-03'),
  ...
);
```

### 2) Partition Rotation Automation
- 매일 1개 파티션을 미리 생성
- 4일 이상 지난 파티션은 별도 백업 테이블로 데이터 이동 후 DROP
- 에러 발생 시 로그를 남기고 다음 날짜로 진행하는 복구 로직 포함
- Kubernetes CronJob으로 스케줄링 가능하도록 설계

### 3) Data Migration
- 대량 데이터 잠금을 피하기 위해 **날짜/시간 단위 Chunk 이관**
- 5만 건 이상 실제 데이터로 이관 스크립트 검증
- 중단 시 재시작 가능한 멱등성 고려

### 4) Dual-write 운영 적용
- Go 기반 ad analytics ingestion 서비스에서
  - 기존 테이블(ad_effect_log)
  - 신규 파티션 테이블(ad_effect_log_v2)
  두 곳에 동일 데이터 저장하도록 Dual-write 구조 적용  
- 운영 중 장애 없이 신규 파티션 테이블로 자연스럽게 전환 가능

### 5) Schema Tooling 제안 (Atlas + TypeORM)
- TypeORM 단독 사용 시 파티션/운영 스키마 관리 한계를 정리
- Atlas를 도입하면 다음 이점이 있음을 정리:
  - 환경별 스키마 drift detection
  - 마이그레이션 계획 자동 생성
  - CI/CD 파이프라인 내 스키마 검증
- 파티션 로테이션 자체는 스크립트로, 나머지 스키마 관리는 Atlas로 분리하는 전략 제안

## 📊 Impact / KPI
- 📈 일 4–6M 로그 처리 환경에서도 안정적인 테이블 관리 구조 확보
- 🔁 파티션 생성–백업–삭제 **100% 자동화**
- 🚫 Dual-write로 **마이그레이션 중 서비스 중단 0건**
- 🧩 파티셔닝 도입을 위한 조사·설계·PoC·운영 전 과정을 단독 수행
