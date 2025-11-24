# GPS-Based Advertisement View Heatmap

## 📌 Overview
버스 단말기에서 수집되는 **GPS 로그 + 광고 시청 데이터**를 결합하여  
광고가 어디에서 얼마나 시청되었는지 시각적으로 분석할 수 있는  
**광고 시청 히트맵 시스템**을 설계·구현했습니다.

## 🎯 Goals
- 광고/날짜/노선별 시청 집중 구간을 지도로 표현
- 대규모 GPS 데이터를 효율적으로 집계하고 캐시
- 광고주가 한눈에 이해할 수 있는 시각화/UX 제공

## 🧱 Responsibilities
- 히트맵 데이터 모델 및 가중치 알고리즘 설계
- MongoDB Aggregation 파이프라인 구현
- 집계 결과를 MySQL에 저장하는 캐시 계층 설계
- GPS 히트맵 조회 API 개발
- Naver Maps API 기반 프론트엔드 페이지 구현
- 샘플링 편향 문제 분석 및 보정 전략 수립
- 히트맵 UX 개선(필터, 드롭다운 구조, 옵션 슬라이더 등)

## 🛠 Data Pipeline

1. MongoDB에서 광고·날짜 기준의 시청 데이터 조회  
2. GPSHistory가 존재하는 레코드만 필터링  
3. 좌표를 100m 단위 Grid(위도/경도 0.001도)로 반올림  
4. Grid별 시청 이벤트를 `weight = count`로 집계  
5. 노선(RouteID)별로 heatmapData 배열 생성  
6. MySQL `ad_route_gps_heatmap` 테이블에 JSON으로 저장  
7. API 서버에서 날짜/ad_id(/route_id) 조건으로 조회 후 프론트에 전달

### Performance
- 집계(aggregate): **광고 1건당 약 2–5초**
- 조회(read): **50–100ms 수준**

## 🗺 Frontend (Naver Maps Heatmap)

- Naver Maps v3 + Visualization 모듈 사용
- `radius`, `opacity`, `maxWeight` 옵션을 UI 슬라이더로 조정
- 날짜/광고/노선 선택 필터 제공
- 줌 컨트롤, 정보 패널, 로딩 인디케이터 구현
- 반응형 레이아웃으로 데스크톱/노트북 환경 지원

예시 옵션:
- radius: 10–15px
- opacity: 0.6 (35% UI 기준)
- maxWeight: 100 (시청 횟수 분포 기준)

## 📊 Sampling Bias Improvement

### Problem
- 기기가 설치된 일부 차량/노선에 데이터가 편중  
- 실제 유동 인구 분포 대비 히트맵이 과대/과소 표현되는 문제

### Strategy
- 노선별 전체 차량 수, 설치 차량 수, 평균 승객 수 등 메타데이터 수집
- 역확률가중치(IPW) 및 샘플링 비율 기반 정규화 계수 설계
- 데이터 포인트 수가 적은 Grid는 confidence score를 낮게 부여하거나 제외
- 최종 weight = 원본 count × 정규화 계수 × 신뢰도 계수 구조로 정의

## 🧪 UX Improvements
- 광고 선택을 **캠페인 → 소재 → 광고** 3단 드롭다운으로 개선
- 항목이 10개를 초과하면 스크롤 가능한 드롭다운으로 변경
- 광고/소재명이 비어 있는 항목 자동 필터링
- 기간 선택 UI를 기존 서비스와 동일 패턴으로 맞추어 사용성 향상

## 📊 Impact / KPI
- 📍 광고 시청 분포를 공간적으로 직관적으로 파악 가능
- ⚡ 집계/조회 모두 실시간 분석에 적합한 성능 달성
- 📊 샘플링 편향을 고려한 설계로 히트맵 해석 신뢰도 향상
- 🧭 광고주/운영자가 활용할 수 있는 고품질 시각화 도구 제공
