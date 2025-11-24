# 🚀 2025 Advertisement Data Engineering Projects

본 저장소는 2025년 수행한 광고 데이터 기반 백엔드·데이터 엔지니어링 프로젝트 전체를 정리합니다.

## 📌 Overview
하루 4~6M 규모의 광고·GPS 로그를 처리하는 시스템을 구축했습니다.
파티셔닝 기반 데이터 엔진, 광고 대시보드 성능 개선, GPS 히트맵 구축 등을 포함합니다.

## ⭐ Key Highlights
- 4~6M/day 로그 처리 위한 MySQL 파티셔닝 자동화
- 파티션 생성·백업·삭제 100% 자동화
- 통계 정확도 10~15% 향상 (중복 제거)
- GPS 집계 2~5초
- GPS 조회 50~100ms
- 샘플링 편향(IPW) 기반 보정 설계
- Naver Maps 기반 광고 히트맵 페이지 개발

---

## 🏗 1. MySQL 로그 파티셔닝 자동화

### Problem
- 하루 4~6M 로그가 누적되어 단일 테이블로는 성능 저하 및 유지보수 어려움

### Solution
- 파티션 전용 테이블(ad_effect_log_v2) 설계
- RANGE(created_at) 기반 일 단위 파티션 생성
- 4일 경과 파티션 자동 백업·삭제
- Dual-write 운영 적용

### Result
- 운영 자동화 100%
- 쿼리 성능 개선 기반 확보
- 무중단 마이그레이션 성공

---

## ⚡ 2. 광고 분석 대시보드 성능 개선

### Improvements
- N+1 제거, Batch Query 도입
- 엑셀 다운로드 속도 개선
- 시청·노출 중복 제거

### Accuracy Gains
- 노출: 12,660 → 10,737명  
- 시청: 3,907 → 3,445명  

---

## 🔁 3. 외부 MongoDB → NKS MongoDB 동기화

### Key Work
- Replica Set 3노드 구성
- 일일 데이터 Batch Upsert 자동화
- ObjectStorage 백업 구축

---

## 🗺️ 4. GPS 기반 광고 히트맵

### Features
- 100m grid 기반 spatial weight 알고리즘
- 집계: 2~5초  
- 조회: 50~100ms  
- Naver Maps 기반 Heatmap 시각화
- UX: 광고 선택 3단 구조, Radius/Opacity 옵션

---

## 📘 What I Learned
- 대규모 로그 파이프라인의 실질적 운영 난이도 체험
- 샘플링 편향 보정의 핵심 원리(IPW)
- 운영 자동화와 데이터 품질 관리의 중요성
- 데이터 엔지니어링·백엔드·프론트의 end-to-end 개발 경험

---

## 📁 Documentation
모든 상세 문서는 `/docs` 폴더에 정리되어 있습니다.

- `partitioning.md` — 로그 파티셔닝 및 자동화
- `dashboard_performance.md` — 광고 대시보드 성능 개선
- `mongo_sync.md` — 외부 Mongo → NKS MongoDB 동기화
- `gps_heatmap.md` — GPS 기반 광고 히트맵 구축

---
