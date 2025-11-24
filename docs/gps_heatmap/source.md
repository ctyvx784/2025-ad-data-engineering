# GPS 히트맵 분석 페이지 추가

## 🎯 주요 기능

### 1. GPS 히트맵 시각화

- **네이버 지도 API** 활용한 실시간 히트맵 렌더링
- GPS 데이터를 기반으로 한 위치별 집중도 시각화
- 가중치(weight) 기반 데이터 포인트 표시

### 2. 사용자 인터페이스

- **날짜 선택기**: 특정 날짜의 데이터 조회
- **광고 선택기**: 특정 광고의 GPS 데이터 필터링
- **줌 컨트롤**: 지도 확대/축소 버튼
- **정보 패널**: 현재 표시 중인 데이터 통계
- **설정 패널**: 히트맵 반경 및 투명도 조절

### 3. API 연동

- 환경별 API URL 설정 (`conf.php`)
- RESTful API를 통한 GPS 데이터 조회
- URL 파라미터를 통한 초기 데이터 로딩

## 🔧 기술적 구현

### 프론트엔드

- **네이버 지도 API v3** + Visualization 모듈
- **Vanilla JavaScript** (의존성 최소화)
- **반응형 CSS** (모바일 친화적)
- **디바운싱** 최적화 (성능 향상)

### 백엔드

- **CodeIgniter 라우팅** 추가
- **API URL 설정** 중앙화
- **PHP 기반** 설정 관리

### 데이터 처리

- **GPS 좌표 유효성 검증**
- **WeightedLocation 객체** 지원
- **배치 데이터 처리** 최적화

## 🚀 사용법

### 기본 접근

```
/adddi_main/busDashboard/adAnalysis/heatmap
```

### 파라미터를 통한 초기 로딩

```
/adddi_main/busDashboard/adAnalysis/heatmap?date=2025-09-15&ad_id=33
```

### 선택적 노선 필터링

```
/adddi_main/busDashboard/adAnalysis/heatmap?date=2025-09-15&ad_id=33&route_id=123
```

## 🎨 UI/UX 특징

### 컨트롤 요소

- **토글 버튼**: 정보 패널과 설정 패널 전환
- **슬라이더**: 히트맵 반경(10-80px), 투명도(10-100%) 조절
- **줌 버튼**: 직관적인 확대/축소 컨트롤

### 정보 표시

- **실시간 통계**: 현재 표시 중인 데이터 포인트 수
- **선택 정보**: 날짜, 광고 ID 표시
- **로딩 상태**: 데이터 로딩 중 시각적 피드백

- 히트맵 옵션 설명
    
    ## 🎯 현재 설정
    
    ```jsx
    heatmapLayer = new naver.maps.visualization.HeatMap({
        map: null,    
        data: [],    
        radius: 10,        // 반경: 10px    
        opacity: 0.6,      // 투명도: 0.6    
        maxWeight: 100,    // 최대 가중치: 100    
        colorMap: naver.maps.visualization.SpectrumStyle.WARM
    });
    ```
    
    ---
    
    ## 1️⃣ radius (반경)
    
    ### 역할
    
    각 데이터 포인트의 **영향 범위**를 픽셀 단위로 설정합니다.
    
    ### 시각적 효과
    
    ```
    radius: 5px  → ●     (작은 점)
    radius: 10px → ⬤     (중간 크기)
    radius: 20px → ⬤⬤   (큰 원)
    ```
    
    ### 실제 거리 환산 (줌 레벨 13 기준)
    
    ```jsx
    radius: 10px ≈ 15-20m (미터)
    // 줌 레벨별 변화
    줌 11: 10px ≈ 60-80m
    줌 13: 10px ≈ 15-20m  ← 현재 설정
    줌 15: 10px ≈ 3-5m
    ```
    
    ### 설정 가이드
    
    | 값 | 효과 | 사용 케이스 |
    | --- | --- | --- |
    | 5-8 | 작은 점 | 정밀한 데이터 표현 |
    | 10-15 | 중간 크기 | **GPS 노선 데이터 (권장)** |
    | 20-30 | 큰 원 | 넓은 영역 표현 |
    | 40+ | 매우 큼 | 전체 개요 보기 |
    
    ### 코드 예시
    
    ```jsx
    // 슬라이더로 조정 가능 (10-80px)
    heatmapLayer.setOptions({ radius: 15 });
    ```
    
    ---
    
    ## 2️⃣ opacity (투명도)
    
    ### 역할
    
    히트맵 레이어의 **불투명도**를 설정합니다 (0.0 ~ 1.0).
    
    ### 시각적 효과
    
    ```
    opacity: 0.0  → 완전 투명 (보이지 않음)
    opacity: 0.3  → 매우 투명 (지도가 잘 보임)
    opacity: 0.6  → 반투명 (균형) ← 현재 설정
    opacity: 0.8  → 불투명 (히트맵이 선명)
    opacity: 1.0  → 완전 불투명 (지도가 안 보임)
    ```
    
    ### 레이어 겹침 효과
    
    ```jsx
    // 포인트가 겹치는 영역
    opacity: 0.3  → 겹쳐도 밝음
    opacity: 0.6  → 겹치면 진해짐 ← 현재 (밀도 표현 좋음)
    opacity: 0.9  → 겹치면 매우 진함
    ```
    
    ### 설정 가이드
    
    | 값 | 효과 | 사용 케이스 |
    | --- | --- | --- |
    | 0.3-0.4 | 은은함 | 지도 정보 강조 |
    | 0.5-0.7 | 균형 | **일반적인 히트맵 (권장)** |
    | 0.8-1.0 | 선명함 | 히트맵 데이터 강조 |
    
    ### 코드 예시
    
    ```jsx
    // 슬라이더로 조정 가능 (10-100%)
    heatmapLayer.setOptions({ opacity: 0.7 });
    ```
    
    ---
    
    ## 3️⃣ maxWeight (최대 가중치)
    
    ### 역할
    
    히트맵 색상 스케일의 **최대값**을 설정합니다. 이 값을 기준으로 색상이 정규화됩니다.
    
    ### 동작 원리
    
    ```jsx
    // 데이터 포인트
    { lat: 37.5, lng: 127.0, weight: 7 }
    // 색상 계산
    normalizedWeight = weight / maxWeight
                     = 7 / 100                 
                     = 0.07  // 7% 강도
    // 색상 매핑 (WARM 스타일)
    0.00 → 노란색 (낮음)
    0.07 → 주황색 (현재 weight: 7)
    0.50 → 빨간색 (중간)
    1.00 → 진한 빨간색 (최대)
    ```
    
    ### 설정 비교
    
    ```jsx
    // 현재 데이터: weight 범위 1-7
    maxWeight: 10   // 7이 70% 강도 → 빨간색에 가까움
    maxWeight: 100  // 7이 7% 강도 → 주황색 ← 현재 설정
    maxWeight: 1000 // 7이 0.7% 강도 → 노란색에 가까움
    ```
    
    ### 시각적 효과
    
    ```
    maxWeight: 10
    ━━━━━━━━━━━━━━━━━━━━
    weight: 1  ▓░░░░░░░░░  (10% - 주황)
    weight: 7  ▓▓▓▓▓▓▓░░░  (70% - 빨강)
    
    maxWeight: 100 (현재)
    ━━━━━━━━━━━━━━━━━━━━
    weight: 1  ░░░░░░░░░░  (1% - 노랑)
    weight: 7  ▓░░░░░░░░░  (7% - 주황)
    
    maxWeight: 1000
    ━━━━━━━━━━━━━━━━━━━━
    weight: 1  ░░░░░░░░░░  (0.1% - 연노랑)
    weight: 7  ░░░░░░░░░░  (0.7% - 노랑)
    ```
    
    ### 설정 가이드
    
    | 데이터 범위 | maxWeight | 효과 |
    | --- | --- | --- |
    | 1-10 | 10-20 | 강렬한 색상 |
    | 1-100 | 100-200 | **균형잡힌 색상 (권장)** |
    | 1-1000 | 1000-2000 | 부드러운 색상 |
    
    ---
    
    ## 🎨 옵션 조합 예시
    
    ### 1. 균형잡힌 표현 (현재 설정)
    
    ```jsx
    radius: 10,      // 적당한 크기
    opacity: 0.6,    // 반투명 (지도 보임)
    maxWeight: 100   // 낮은 강도 (부드러운 색상)
    ```
    
    **결과**:
    - ✅ 데이터 포인트가 자연스럽게 연결
    - ✅ 지도 정보와 히트맵 모두 보임
    - ✅ 색상이 과하지 않고 부드러움
    
    ### 2. 선명한 표현
    
    ```jsx
    radius: 15,      // 큰 크기
    opacity: 0.8,    // 불투명
    maxWeight: 10    // 높은 강도
    ```
    
    **결과**: 히트맵이 매우 선명하고 강렬함
    
    ### 3. 은은한 표현
    
    ```jsx
    radius: 8,       // 작은 크기
    opacity: 0.4,    // 투명
    maxWeight: 200   // 낮은 강도
    ```
    
    **결과**: 히트맵이 배경처럼 은은함
    
    ---
    
    ## 📈 데이터 특성별 권장 설정
    
    ### GPS 버스 노선 데이터 (현재)
    
    ```jsx
    radius: 10,      // GPS 간격 5-10m에 맞춤
    opacity: 0.6,    // 노선 겹침 고려
    maxWeight: 100   // weight 1-7 범위에 맞춤
    ```
    
    ### 매장 방문 데이터 (포인트 밀집)
    
    ```jsx
    radius: 20,      // 넓은 영향 범위
    opacity: 0.7,    // 선명하게
    maxWeight: 50    // 높은 방문수 강조
    ```
    
    ### 인구 밀도 데이터 (넓은 영역)
    
    ```jsx
    radius: 30,      // 매우 넓은 범위
    opacity: 0.5,    // 투명하게
    maxWeight: 1000  // 큰 숫자 범위
    ```
    
    ---
    
    ## 📚 참고 자료
    
    ### Naver Maps API 문서
    
    - [HeatMap 가이드](https://navermaps.github.io/maps.js.ncp/docs/tutorial-heatmap-simple.example.html)
    - [Visualization API](https://navermaps.github.io/maps.js.ncp/docs/naver.maps.visualization.HeatMap.html)
    
    ### 색상 스타일
    
    - `WARM`: 노란색 → 주황색 → 빨간색 (현재 사용)
    - `HOT`: 검정색 → 빨간색 → 노란색 → 흰색
    - `JET`: 파란색 → 초록색 → 노란색 → 빨간색
    
    ---
    
    ## 🐛 문제 해결
    
    ### 히트맵이 너무 흐릿함
    
    ```jsx
    // 해결: opacity 증가, radius 감소
    heatmapLayer.setOptions({
        radius: 8,
        opacity: 0.8
    });
    ```
    
    ### 히트맵이 너무 강렬함
    
    ```jsx
    // 해결: opacity 감소, maxWeight 증가h
    eatmapLayer.setOptions({
        opacity: 0.4,
        maxWeight: 200
    });
    ```
    
    ### 데이터 포인트가 띄엄띄엄 보임
    
    ```jsx
    // 해결: radius 증가
    heatmapLayer.setOptions({
        radius: 20
    });
    ```
    
    ### 색상이 모두 비슷함
    
    ```jsx
    // 해결: maxWeight를 데이터 범위에 맞게 조정// 예: weight 범위가 1-10이면
    heatmapLayer.setOptions({
        maxWeight: 15  // 최대값의 1.5배
    });
    ```
    
    ---
    

## 🔍 성능 최적화

### 메모리 관리

- **이벤트 리스너 정리**: 페이지 언로드 시 자동 정리
- **디바운싱**: 줌 변경 시 300ms 지연 적용
- **데이터 캐싱**: 전체 데이터를 한 번에 로드하여 필터링 제거

### 렌더링 최적화

- **히트맵 라이브러리**: 네이버 지도 내장 최적화 활용
- **Viewport 기반 렌더링**: 화면에 보이는 영역만 렌더링
- **가중치 기반 표시**: 데이터 밀도에 따른 시각적 차별화

## 🧪 테스트 시나리오

### 기본 기능 테스트

1. 페이지 로드 시 지도 초기화 확인
2. 날짜/광고 선택 후 데이터 로딩 확인
3. 히트맵 렌더링 및 시각화 확인

### UI 컨트롤 테스트

1. 줌 인/아웃 버튼 동작 확인
2. 반경/투명도 슬라이더 조절 확인
3. 정보 패널 토글 기능 확인

### API 연동 테스트

1. 유효한 파라미터로 데이터 로딩 확인
2. 잘못된 파라미터에 대한 에러 처리 확인
3. 네트워크 오류 시 사용자 피드백 확인

## ✅ 체크리스트

- [x]  네이버 지도 API 연동
- [x]  히트맵 시각화 구현
- [x]  사용자 인터페이스 구현
- [x]  API URL 설정
- [x]  라우팅 설정
- [x]  에러 처리
- [x]  로딩 상태 표시
- [x]  반응형 디자인
- [x]  성능 최적화
- [x]  코드 주석 및 문서화

## 📝 결론

이 PR은 버스 대시보드의 광고 분석 기능을 크게 향상시키는 중요한 기능 추가입니다. GPS 데이터를 직관적인 히트맵으로 시각화함으로써 사용자가 광고의 지리적 영향력을 쉽게 파악할 수 있게 됩니다.

**주요 성과:**
- 📊 **데이터 시각화**: GPS 데이터의 공간적 분포를 직관적으로 표현
- 🎛️ **사용자 제어**: 다양한 UI 컨트롤을 통한 맞춤형 분석
- ⚡ **성능 최적화**: 효율적인 렌더링과 메모리 관리
- 🔧 **확장성**: 향후 기능 추가를 고려한 모듈화된 구조

이 기능은 광고주와 마케터가 버스 광고의 효과를 공간적으로 분석하고 최적화하는 데 큰 도움이 될 것입니다.

# GPS 히트맵 집계 기능 보고서

## 📋 개요

광고 시청 데이터의 GPS 정보를 기반으로 노선별 히트맵을 생성하여, 어느 지역에서 광고가 많이 시청되었는지 시각화할 수 있는 기능입니다.

**작성일**: 2025-10-24

**버전**: 1.0.0

---

## 🎯 주요 기능

### 1. GPS 히트맵 집계

- 특정 날짜/광고의 시청 데이터에서 GPS 좌표 추출
- 100m 단위 그리드로 좌표 그룹핑
- 노선별로 시청 빈도(weight) 계산
- MySQL에 집계 결과 저장

### 2. 히트맵 데이터 조회

- 날짜, 광고 ID 기반 조회
- 특정 노선 또는 전체 노선 조회 가능
- Naver Maps V3 호환 JSON 형식 반환

---

## 🏗 시스템 아키텍처

```
MongoDB (ad_viewing_rounds_YYYYMM)
         ↓
    [집계 파이프라인]
         ↓
    GPS 그리드화 (0.001도 단위)
         ↓
    노선별 그룹핑 & Weight 계산
         ↓
MySQL (ad_route_gps_heatmap)
         ↓
    [API 조회]
         ↓
    JSON 응답 (히트맵 포인트 배열)
```

---

## 📊 데이터 처리 과정

### 1단계: GPS 좌표 필터링

```jsx
// MongoDB Aggregation{
  $match: {
    date: "2025-09-15",    
    ad_id: 33,    
    gps_info: { $exists: true, $ne: null }
  }
}
```

- 해당 날짜/광고의 GPS 정보가 있는 시청 데이터만 선택

### 2단계: 그리드 변환

```jsx
{
  $addFields: {
    grid_lat: $round(lat * 1000) / 1000,  
    // 0.001도 = 약 100m    
    grid_lng: $round(lng * 1000) / 1000  }
}
```

- 원본 GPS 좌표를 100m 단위 그리드로 반올림
- 근접한 좌표들을 하나의 포인트로 통합

**예시:**
- `37.5665` → `37.567`
- `126.9780` → `126.978`

### 3단계: Weight 계산

```jsx
{
  $group: {
    _id: { route_id, lat, lng },    
    weight: { $sum: 1 }  // 시청 횟수 카운트  
  }
}
```

- 노선별, 그리드별로 그룹핑
- **weight**: 해당 위치에서 광고가 시청된 횟수

### 4단계: 노선별 히트맵 배열 생성

```jsx
{
  $group: {
    _id: "$route_id",    
    heatmap_data: {
      $push: { lat, lng, weight }
    }
  }
}
```

- 각 노선별로 히트맵 포인트 배열 생성

### 5단계: MySQL 저장

```sql
INSERT INTO ad_route_gps_heatmap (date, ad_id, route_id, heatmap_data)
VALUES (?, ?, ?, ?)
ON DUPLICATE KEY UPDATE  
heatmap_data = VALUES(heatmap_data), 
updated_at = CURRENT_TIMESTAMP
```

- JSON 형태로 직렬화하여 저장
- 중복 시 업데이트

---

## 🔧 데이터베이스 스키마

### ad_route_gps_heatmap 테이블

```sql
CREATE TABLE ad_route_gps_heatmap (
  date DATE NOT NULL,
  ad_id BIGINT NOT NULL,
  route_id VARCHAR(255) NOT NULL,
  heatmap_data JSON NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (date, ad_id, route_id),
  INDEX idx_date (date),
  INDEX idx_ad_id (ad_id)
);
```

### heatmap_data JSON 구조

```json
[  
	{    
		"lat": 37.567,    
		"lng": 126.978,    
		"weight": 15  
	},  
	{    
		"lat": 37.568,    
		"lng": 126.979,    
		"weight": 23  
	}
]
```

---

## 🌐 API 엔드포인트

### 1. GPS 히트맵 집계 API

**Endpoint**: `POST /ad-analytics/gps-heatmap/aggregate`

**Request Body**:

```json
{  "date": "2025-09-15",  "ad_id": "33"}
```

**Parameters**:
- `date` (필수): 집계 대상 날짜 (YYYY-MM-DD)
- `ad_id` (선택): 특정 광고 ID. 없으면 해당 날짜의 모든 광고 집계

**Response**:

```json
{  "status": "success",  "message": "광고 33의 GPS 히트맵 집계 완료",  "date": "2025-09-15"}
```

**처리 시간**: 광고당 평균 2-5초

---

### 2. GPS 히트맵 조회 API

**Endpoint**: `GET /ad-analytics/gps-heatmap`

**Query Parameters**:
- `date` (필수): 조회 날짜 (YYYY-MM-DD)
- `ad_id` (필수): 광고 ID
- `route_id` (선택): 특정 노선 ID. 없으면 전체 노선

**Request Example**:

```
GET /ad-analytics/gps-heatmap?date=2025-09-15&ad_id=33
GET /ad-analytics/gps-heatmap?date=2025-09-15&ad_id=33&route_id=100100006
```

**Response**:

```json
[  
	{    
		"date": "2025-09-15",    
		"adId": 33,    
		"routeId": "100100006",    
		"heatmapData": [      
			{        
				"lat": 37.567,        
				"lng": 126.978,        
				"weight": 15      
			},      
			{        
				"lat": 37.568,        
				"lng": 126.979,        
				"weight": 23      
			}    
		]  
	}
]
```

**Response (데이터 없음)**:

```json
[]
```

---

## 💡 Weight 가중치 설명

### Weight의 의미

- **Weight = 시청 횟수**: 해당 GPS 그리드에서 광고가 시청된 총 횟수
- 같은 위치(100m 반경)에서 여러 번 시청되면 weight 증가

### Weight 계산 예시

| GPS 좌표 (원본) | 그리드 좌표 | 시청 횟수 | Weight |
| --- | --- | --- | --- |
| 37.5671, 126.9782 | 37.567, 126.978 | 10회 | 10 |
| 37.5673, 126.9785 | 37.567, 126.978 | 5회 | 15 (누적) |
| 37.5681, 126.9791 | 37.568, 126.979 | 23회 | 23 |

### 히트맵 시각화

- **Weight 1-10**: 낮은 시청 빈도 (파란색)
- **Weight 11-50**: 중간 시청 빈도 (노란색)
- **Weight 51+**: 높은 시청 빈도 (빨간색)

---

## ⚡ 성능 특징

### 최적화 포인트

1. **그리드 집계**: 100m 단위로 GPS 좌표를 그룹핑하여 데이터 크기 대폭 감소
2. **MongoDB Aggregation**: 서버 측에서 집계 처리 (클라이언트 부하 최소화)
3. **JSON 저장**: 미리 집계된 결과를 MySQL에 캐싱
4. **인덱스 활용**: (date, ad_id, route_id) 복합 키로 빠른 조회

### 예상 성능

- **집계 시간**: 광고당 2-5초 (1만 건 기준)
- **조회 시간**: 50-100ms
- **데이터 크기**: 노선당 평균 100-500 포인트

---

## 🔍 활용 사례

### 1. 광고 효과 분석

- 어느 구간에서 광고가 많이 시청되는지 파악
- 특정 지역의 광고 효과 측정

### 2. 노선 최적화

- 시청 빈도가 높은 노선 식별
- 광고 배치 전략 수립

### 3. 지역별 타겟팅

- 특정 지역 집중 광고 효과 분석
- 지역 맞춤형 광고 전략 수립

---

## 🚨 제한사항 및 고려사항

### 현재 제한사항

1. **그리드 단위 고정**: 100m 단위로 고정 (조정 불가)
2. **GPS 정확도**: 원본 GPS 데이터 품질에 의존

### 향후 개선 방향

1. **그리드 크기 조정 가능**: API 파라미터로 그리드 크기 설정
2. **시간대별 히트맵**: 특정 시간대의 히트맵 조회
3. **필터링 옵션**: 성별, 연령대별 히트맵 분리

---

## 📝 운영 가이드

### 일일 배치 작업

```bash
# 전체 광고 히트맵 집계 (전날 기준)
curl -X POST http://localhost:3010/ad-analytics/gps-heatmap/aggregate \  -H "Content-Type: application/json" \  -d '{"date":"2025-09-15"}'
```

### 특정 광고 재집계

```bash
# 특정 광고만 재집계
curl -X POST http://localhost:3010/ad-analytics/gps-heatmap/aggregate \  -H "Content-Type: application/json" \  -d '{"date":"2025-09-15","ad_id":"33"}'
```

### 데이터 확인

```sql
-- 집계된 히트맵 확인
SELECT
  date,
  ad_id,
  route_id,
  JSON_LENGTH(heatmap_data) as point_count,
  updated_at
FROM ad_route_gps_heatmap
WHERE date = '2025-09-15' AND ad_id = 33;
-- Weight 분포 확인 (JSON 파싱 필요)
SELECT
  route_id,
  JSON_EXTRACT(heatmap_data, '$[*].weight') as weights
FROM ad_route_gps_heatmap
WHERE date = '2025-09-15' AND ad_id = 33;
```

---

## 🧪 테스트 시나리오

### 1. 기본 집계 테스트

```bash
# 1. 히트맵 집계
curl -X POST http://localhost:3010/ad-analytics/gps-heatmap/aggregate \  -H "Content-Type: application/json" \  -d '{"date":"2025-09-15","ad_id":"33"}'
# 2. 결과 조회
curl "http://localhost:3010/ad-analytics/gps-heatmap?date=2025-09-15&ad_id=33"
```

### 2. 노선별 조회 테스트

```bash
# 특정 노선만 조회
curl "http://localhost:3010/ad-analytics/gps-heatmap?date=2025-09-15&ad_id=33&route_id=100100006"
```

### 3. 에러 케이스 테스트

```bash
# 필수 파라미터 누락
curl "http://localhost:3010/ad-analytics/gps-heatmap?date=2025-09-15"
# → 400 Bad Request: ad_id parameter is required
# 잘못된 날짜 형식
curl "http://localhost:3010/ad-analytics/gps-heatmap?date=2025/09/15&ad_id=33"
# → 400 Bad Request: 날짜 형식이 올바르지 않습니다
# 존재하지 않는 데이터
curl "http://localhost:3010/ad-analytics/gps-heatmap?date=2025-01-01&ad_id=999"
# → 200 OK: []
```

---

## 📚 참고 자료

### 관련 코드 파일

- `services/v2_daily_ad_metrics.go`: 히트맵 집계 로직
- `handlers/handlers.go`: API 핸들러
- `models/route.go`: GPS 데이터 모델

### 관련 테이블

- `ad_viewing_rounds_{YYYYMM}`: 원본 시청 데이터 (MongoDB)
- `ad_route_gps_heatmap`: 집계된 히트맵 데이터 (MySQL)
- `ad_effect_log_v2`: 광고 효과 데이터

---

## ✅ 체크리스트

### 배포 전 확인사항

- [ ]  MySQL 테이블 생성 완료
- [ ]  인덱스 생성 완료
- [ ]  API 엔드포인트 테스트 완료
- [ ]  샘플 데이터로 히트맵 시각화 확인
- [ ]  에러 핸들링 테스트 완료
- [ ]  로그 레벨 확인

### 운영 모니터링

- [ ]  일일 배치 작업 스케줄링
- [ ]  API 응답 시간 모니터링
- [ ]  MySQL 저장 공간 모니터링
- [ ]  GPS 데이터 품질 모니터링

---