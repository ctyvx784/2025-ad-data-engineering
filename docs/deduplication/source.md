# 노출, 시청 중복 인구 제거 테스트

담당자: 강영선
상태: 닫힘
우선순위: 높음
서비스: Model Light
ID: JOB-306
시작 예정일: 2025년 9월 5일
시작일: 2025년 9월 5일
작업 구분: backend
작업 유형: 기능 개선
종료 예정일: 2025년 9월 8일
종료일: 2025년 9월 8일

# 상세정보

- 현대엘리베이터_소재교체_15s_7025 광고에 대한 9/3 일 노출, 시청 인구에 대한 중복 제거 요청
- raw_observation_data 테이블의 한 행을 unique 인구로 가정
- 광고 키에 대한 unique 인구를 배열로 셋팅
- watched_history의 노출, 시청 로직 조건을 만족하면 카운팅을 1로 고정

# 작업 결과

- 9/3일 실서버 데이터
    
    ![image.png](%EB%85%B8%EC%B6%9C,%20%EC%8B%9C%EC%B2%AD%20%EC%A4%91%EB%B3%B5%20%EC%9D%B8%EA%B5%AC%20%EC%A0%9C%EA%B1%B0%20%ED%85%8C%EC%8A%A4%ED%8A%B8/image.png)
    
    - 노출 인구: 12,660명
    - 시청 인구: 3,907 명
- 9/3일 로컬 데이터
    
    ![image (1).png](%EB%85%B8%EC%B6%9C,%20%EC%8B%9C%EC%B2%AD%20%EC%A4%91%EB%B3%B5%20%EC%9D%B8%EA%B5%AC%20%EC%A0%9C%EA%B1%B0%20%ED%85%8C%EC%8A%A4%ED%8A%B8/image_(1).png)
    
    - 노출 인구: 10,737 명
    - 시청 인구: 3,445 명

# 관련 파일

[https://github.com/addddev/k8s-data-api/commit/9162791dfdc77b33182b2fd73558db7900007ece](https://github.com/addddev/k8s-data-api/commit/9162791dfdc77b33182b2fd73558db7900007ece)

[https://www.notion.so](https://www.notion.so)