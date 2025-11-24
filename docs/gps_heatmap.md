# GPS 기반 광고 히트맵 구축

## Data Pipeline
- GPSHistory × 시청 로그 매칭
- 100m grid 기반 가중치 집계
- Mongo Aggregation → MySQL 저장

## Performance
- 집계: 2–5초
- 조회: 50–100ms

## UX Features
- Radius/Opacity 조절
- Naver Maps 기반 실시간 시각화
