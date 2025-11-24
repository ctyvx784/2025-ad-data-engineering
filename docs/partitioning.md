# MySQL Log Partitioning & Automation

## Problem
- 하루 4–6M 광고 로그로 단일 테이블 성능 저하
- 인덱스 팽창 및 백업 어려움

## Solution
- ad_effect_log_v2 신규 파티션 테이블 설계
- RANGE(created_at) 기반 일 단위 파티션 생성
- 4일 경과 파티션 자동 백업/삭제 (로테이션)
- Dual-write로 무중단 전환

## Result
- 일 단위 자동화 100%
- 안정적 운영 및 쿼리 성능 개선 기반 확보
