# ad-effecct-log 테이블 파티셔닝

담당자: 강영선
상태: 닫힘
우선순위: 높음
서비스: Model Light
ID: JOB-302
시작 예정일: 2025년 9월 8일
시작일: 2025년 9월 8일
작업 구분: Data
작업 유형: 기능 개선
종료 예정일: 2025년 9월 10일

# 상세정보

[파티셔닝 PoC](https://www.notion.so/PoC-2459fb013f358053b360d1565a98d30a?pvs=21) 을 따라 파티셔닝 진행

# 작업 계획

## 📋 배포 목표

1. **ad_effect_log_v2 테이블 생성**
2. **ad_effect_log_v2에 파티셔닝 적용**
3. **새로 들어오는 데이터를 기존 ad_effect_log와 ad_effect_log_v2에 저장** (기존 데이터 copy는 안함)
4. **cron job 등록 또는 수동으로 partition-manager 실행**

---

## 🎯 1단계: ad_effect_log_v2 테이블 생성

### 실행 명령어

```bash
# 실서버에서 실행node scripts/create_ad_effect_log_v2.js
```

### 확인 방법

```bash
# 테이블 생성 확인mysql -h [DB_HOST] -P [DB_PORT] -u [DB_USER] -p[DB_PASSWORD] [DB_NAME] -e "SHOW TABLES LIKE 'ad_effect_log_v2';"
```

### 예상 결과

- `ad_effect_log_v2` 테이블이 생성됨
- 기존 `ad_effect_log` 테이블과 동일한 스키마 구조
- `created_at` 컬럼 추가 (파티셔닝용)

---

## 🎯 2단계: ad_effect_log_v2에 파티셔닝 적용

### 실행 명령어

```bash
# 실서버에서 실행node scripts/setup_partitions.js
```

### 확인 방법

```bash
# 파티션 설정 확인node scripts/check_partitions.js
```

### 예상 결과

- 날짜별 파티션 생성 (오늘부터 60일 전까지)
- `pMAX` 파티션 생성 (미래 데이터용)
- 파티션 타입: `RANGE (TO_DAYS(created_at))`

---

## 🎯 3단계: Dual-Write 설정 (기존 + 새 테이블 동시 저장)

### 수정 대상 파일

**`apps/ad-analytics-go/services/daily_ad_metrics.go`** (1484-1485줄)

### 기존 코드

```go
insertQuery := `INSERT INTO ad_effect_log (ad_id, ad_name, ad_start_time, ad_end_time, vehicle_number, route_code, date, exposure, views, attention, male_views, female_views, watch_history, exposure_history, attention_history) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)`_, err := s.mysqlDB.Exec(insertQuery, adInfoMap[sLog.MaterialID].ID, adInfoMap[sLog.MaterialID].FileName, sLog.Time, fLog.Time, sLog.VehicleNumber, sLog.BusRouteID, sLog.Date, pairResult.ExposedCount, pairResult.WatchedCount, pairResult.AttentionCount, pairResult.MaleCount, pairResult.FemaleCount, watchHistoriesStr, exposureHistoriesStr, attentionHistoriesStr)
```

### 수정된 코드

```go
// 1. 기존 테이블에 저장insertQuery := `INSERT INTO ad_effect_log (ad_id, ad_name, ad_start_time, ad_end_time, vehicle_number, route_code, date, exposure, views, attention, male_views, female_views, watch_history, exposure_history, attention_history) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)`_, err := s.mysqlDB.Exec(insertQuery, adInfoMap[sLog.MaterialID].ID, adInfoMap[sLog.MaterialID].FileName, sLog.Time, fLog.Time, sLog.VehicleNumber, sLog.BusRouteID, sLog.Date, pairResult.ExposedCount, pairResult.WatchedCount, pairResult.AttentionCount, pairResult.MaleCount, pairResult.FemaleCount, watchHistoriesStr, exposureHistoriesStr, attentionHistoriesStr)if err != nil {    log.Printf("❌ ad_effect_log 삽입 실패: %v", err)    return err
}// 2. 새 테이블에도 저장 (created_at 추가)insertQueryV2 := `INSERT INTO ad_effect_log_v2 (ad_id, ad_name, ad_start_time, ad_end_time, vehicle_number, route_code, date, exposure, views, attention, male_views, female_views, watch_history, exposure_history, attention_history, created_at) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, NOW())`_, errV2 := s.mysqlDB.Exec(insertQueryV2, adInfoMap[sLog.MaterialID].ID, adInfoMap[sLog.MaterialID].FileName, sLog.Time, fLog.Time, sLog.VehicleNumber, sLog.BusRouteID, sLog.Date, pairResult.ExposedCount, pairResult.WatchedCount, pairResult.AttentionCount, pairResult.MaleCount, pairResult.FemaleCount, watchHistoriesStr, exposureHistoriesStr, attentionHistoriesStr)if errV2 != nil {    log.Printf("❌ ad_effect_log_v2 삽입 실패: %v", errV2)    // 기존 테이블 삽입은 성공했으므로 에러를 반환하지 않음    // 필요시 별도 알림 처리}
```

### 배포 후 확인

- Go 서비스 재배포 후 새 데이터가 두 테이블에 모두 저장되는지 확인
- 데이터 일관성 검증

---

## 🎯 4단계: Partition Manager 실행

### 옵션 1: 쿠버네티스 CronJob 배포

```bash
# 쿠버네티스 환경에서 실행kubectl apply -f k8s/partition-management-cronjob.yaml
# CronJob 상태 확인kubectl get cronjobs
kubectl describe cronjob partition-management
```

### 옵션 2: 수동 실행

```bash
# 실서버에서 수동 실행npm run partition-manager
# 또는 중단된 작업 재시작npm run partition-manager:continue
```

### 옵션 3: 시스템 cron 등록

```bash
# crontab 편집crontab -e# 매일 새벽 2시에 실행0 2 * * * cd /path/to/bus-dashboard-server && npm run partition-manager >> /var/log/partition-manager.log 2>&1
```

---

## ⚙️ 배포 전 준비사항

### 환경변수 설정

```bash
# .env 파일 또는 환경변수 설정DB_HOST=실서버_DB_HOST
DB_PORT=실서버_DB_PORT
DB_USERNAME=실서버_DB_USER
DB_PASSWORD=실서버_DB_PASSWORD
DB_DATABASE=실서버_DB_NAME
TARGET_TABLE=ad_effect_log_v2
BACKUP_RETENTION_DAYS=60
```

### 데이터베이스 권한 확인

- ✅ 테이블 생성 권한
- ✅ 파티션 생성/삭제 권한
- ✅ 데이터 삽입/삭제 권한

---

## 🚀 배포 실행 순서

```bash
# 1. 테이블 생성node scripts/create_ad_effect_log_v2.js
# 2. 파티션 설정node scripts/setup_partitions.js
# 3. 파티션 상태 확인node scripts/check_partitions.js
# 4. Go 서비스 재배포 (dual-write 코드 적용 후)# 5. partition-manager 실행 (cron 또는 수동)npm run partition-manager
# 6. 백업 파티션 상태 확인npm run check-backup-partitions
```

---

## 🔍 배포 후 검증

### 1. 테이블 생성 확인

```bash
mysql -e "SHOW TABLES LIKE 'ad_effect_log_v2';"
```

### 2. 파티션 설정 확인

```bash
node scripts/check_partitions.js
```

### 3. 데이터 삽입 테스트

- Go 서비스에서 새 데이터가 두 테이블에 모두 저장되는지 확인
- 데이터 일관성 검증

### 4. partition-manager 로그 확인

```bash
tail -f logs/partition-manager-$(date +%Y-%m-%d).log
```

### 5. 백업 파티션 상태 확인

```bash
npm run check-backup-partitions
```

---

## 📊 Partition Manager 기능

### 주요 기능

- **월별 백업**: 오래된 파티션 데이터를 월별 파티셔닝된 백업 테이블로 자동 백업
- **동적 파티션 생성**: 실서버 배포 시점 기준 과거 1년 ~ 미래 2개월 범위의 월별 파티션 자동 생성
- **스마트 정리**: 백업 보관 정책에 따른 오래된 월별 파티션 자동 정리
- **자동 변환**: 기존 백업 테이블을 월별 파티셔닝으로 자동 변환
- **누락 파티션 자동 생성**: 매일 실행 시 누락된 월별 파티션 자동 감지 및 생성

### 백업 프로세스

1. **파티션 확인**: 백업 대상 파티션 식별 (기본 60일 이전)
2. **백업 테이블 생성**: 월별 파티셔닝된 백업 테이블 생성/확인
3. **동적 파티션 생성**: 누락된 월별 파티션 자동 감지 및 생성
4. **데이터 백업**: 오래된 파티션 데이터를 해당 월의 백업 파티션으로 복사
5. **원본 삭제**: 백업 완료 후 원본 파티션 삭제
6. **정리 작업**: 백업 테이블의 오래된 월별 파티션 정리

---

## 🛠️ 유용한 명령어

### 파티션 상태 확인

```bash
# 메인 테이블 파티션 상태npm run check-partitions
# 백업 테이블 파티션 상태npm run check-backup-partitions
```

### Partition Manager 실행

```bash
# 기본 실행npm run partition-manager
# 중단된 작업 재시작npm run partition-manager:continue
# 테스트 실행npm run partition-manager:test
```

### 로그 확인

```bash
# 실시간 로그 확인tail -f logs/partition-manager-$(date +%Y-%m-%d).log
# 쿠버네티스 환경에서 로그 확인kubectl logs -f deployment/partition-manager
```

---

## ⚠️ 주의사항

1. **데이터 일관성**: Dual-write 설정 시 두 테이블 간 데이터 일관성 유지 필요
2. **성능 영향**: 두 테이블에 동시 저장으로 인한 약간의 성능 저하 예상
3. **백업 정책**: `BACKUP_RETENTION_DAYS` 설정에 따른 백업 보관 기간 관리
4. **모니터링**: partition-manager 실행 상태 및 로그 지속적 모니터링 필요

---

## 📞 문제 해결

### 자주 발생하는 문제

1. **파티션 생성 실패**: 데이터베이스 권한 확인
2. **백업 실패**: 디스크 공간 및 백업 테이블 상태 확인
3. **Dual-write 실패**: Go 서비스 로그 확인 및 데이터베이스 연결 상태 점검

### 로그 위치

- **로컬**: `./logs/`
- **쿠버네티스**: `/app/logs/`
- **로그 파일명**: `partition-manager-YYYY-MM-DD.log`

---

## ✅ 체크리스트

- [ ]  환경변수 설정 완료
- [ ]  데이터베이스 권한 확인
- [ ]  ad_effect_log_v2 테이블 생성
- [ ]  파티션 설정 적용
- [ ]  Go 서비스 dual-write 코드 수정 및 배포
- [ ]  partition-manager 실행 (cron 또는 수동)
- [ ]  배포 후 검증 완료
- [ ]  모니터링 설정 완료

---

## 작업 계획 변경 - 9/12

- cronjob으로 미실행된 파티셔닝 시나리오에서 복구 과정에서 복잡함이 존재함
    - pMAX(분류되지 않은 데이터가 모이는 파티셔닝)로 부터 누락된 파티셔닝을 생성할 때, 데이터 이동을 고려해야함 → 즉, 파티션 분배와 데이터 이동 중 데이터 손실이 발생하는 경우를 또 고려 해야하는 상황
- 따라서 pMAX 파티셔닝을 삭제하고, 데이터 삽입 전에 파티셔닝을 현황을 스캔하고 누락된 파티셔닝을 미리 생성하는 로직으로 변경함
- 오래된 파티셔닝을 백업으로 이동하는 로직은 그대로 cronjob으로 진행할 예정

# 관련 파일

[https://www.notion.so](https://www.notion.so)

# 파티셔닝 자동화 PoC

담당자: 강영선
상태: 닫힘
우선순위: 보통
서비스: DataBase/Server
ID: JOB-227
시작 예정일: 2025년 8월 4일
시작일: 2025년 8월 4일
작업 구분: backend
작업 유형: 환경구성
종료 예정일: 2025년 8월 8일

# 상세정보

- [파티셔닝 테이블 마이그레이션 자동화를 위한 조사](https://www.notion.so/2419fb013f3580519ab5dccb04f526ef?pvs=21) 결과로 Atlas 도입을 제안함
- 하지만 기존 Prisma가 하던 역할을 대체하거나 호환성에 큰 이점이 없고 파티셔닝 스키마 관리는 여전히 수동으로 해야하기 때문에 도입을 철회함
- 따라서 파티셔닝 스키마를 스크립트로 자동 관리하기 위한 PoC를 진행

# 성공 지표

- 자동화 스크립트 PoC 테스트 성공

# 작업 계획

- [x]  Prisma 실행 환경 DB 셋팅
- [x]  ad_effect_log 테이블에 더미 데이터 생성
- [x]  ad_effect_log_v2 테이블 생성
    - 기존 테이블은 생성 시간을 기준으로 RANGE 파티셔닝을 하기 위한 조건에 부적합함
    - 기존 기능 호환성을 고려해 새로운 테이블에 연결 포인트 추가
- [x]  ad_effect_log_v2 파티션 설정
- [x]  데이터 이관
- [x]  일별 자동 파티셔닝 스크립트 실행

# 작업 결과

- [파티셔닝 PoC](https://www.notion.so/PoC-2459fb013f358053b360d1565a98d30a?pvs=21)

# 관련 파일

[https://www.notion.so](https://www.notion.so)

# 파티셔닝 PoC

## 기존 파티셔닝 PoC

- ~~Atlas PoC~~
    
    atlas 마이그레이션 워크플로우
    
    - Atlas .hcl 파일 마이그레이션 워크플로우 예시
        1. **현재 스키마 추출**
            
            ```bash
            atlas schema inspect --env local > schema.hcl
            ```
            
        2. **HCL 파일에 새로운 테이블 정의 추가**
            - 기존 테이블 정의는 그대로 유지
            - 새로운 테이블 정의를 추가
        3. **변경사항 확인 (Diff)**
            
            ```bash
            atlas schema diff \
            --from "mysql://root:toor@localhost:3307/partition_poc" \
            --to file://schema.hcl \
            --dev-url "mysql://root:toor@localhost:3307/temp_migration"
            ```
            
        4. **마이그레이션 파일 생성**
            
            ```bash
            atlas migrate new add_table_name
            ```
            
        5. **마이그레이션 파일에 SQL 추가**
            - diff 결과의 SQL을 마이그레이션 파일에 복사
        6.  **체크섬 재계산**
            
            ```bash
            atlas migrate hash
            ```
            
        7. **마이그레이션 적용**
            
            ```bash
            atlas migrate apply --env local
            ```
            
    - Atlas .sql 파일 마이그레이션 워크플로우 예시
        1. **현재 스키마 추출**
            
            ```bash
            atlas schema inspect --env local > schema.hcl
            ```
            
        2. **HCL 파일 → SQL 파일로 변환**
            
            ```bash
            atlas schema inspect --env local --format '{{ sql . }}' > atlas_schema.sql
            ```
            
        3. SQL 파일에 스키마 업데이트
        4. **변경사항 확인 (Diff)**
            
            ```bash
            atlas schema diff \
            --from "mysql://root:toor@localhost:3307/partition_poc" \
            --to file://schema.hcl \
            --dev-url "mysql://root:toor@localhost:3307/temp_migration"
            ```
            
        5. **마이그레이션 파일 생성**
            
            ```bash
            atlas migrate new add_table_name
            ```
            
        6. **마이그레이션 파일에 SQL 추가**
            - diff 결과의 SQL을 마이그레이션 파일에 복사
        7.  **체크섬 재계산**
            
            ```bash
            atlas migrate hash
            ```
            
        8. **마이그레이션 적용**
            
            ```bash
            atlas migrate apply --env local
            ```
            
    
    ---
    
    - 기존 테이블 Init
        
        ```sql
        -- create_old_ad_effect_log.sql
        CREATE TABLE IF NOT EXISTS ad_effect_log_old (
          idx BIGINT AUTO_INCREMENT,
          ad_id BIGINT NOT NULL,
          ad_name VARCHAR(255) NOT NULL,
          ad_start_time VARCHAR(255) NOT NULL,
          ad_end_time VARCHAR(255) NOT NULL,
          vehicle_number VARCHAR(255) NOT NULL,
          route_code VARCHAR(255) NOT NULL,
          date DATE NOT NULL,
          exposure INT,
          views INT,
          attention INT,
          male_views INT,
          female_views INT,
          created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
          PRIMARY KEY (idx),
          INDEX IDX_vehicle_number (vehicle_number),
          INDEX IDX_route_code (route_code),
          INDEX IDX_date (date),
          INDEX IDX_ad_id (ad_id),
          INDEX IDX_created_at (created_at)
        ) ENGINE=InnoDB
          DEFAULT CHARSET=utf8mb4
          COLLATE=utf8mb4_0900_ai_ci;
        ```
        
        ```bash
        # 명령어 실행 후, .sql 파일과 .sum 파일이 생성되야함
        atlas migrate diff initial \
        	--to file://create_old_ad_effect_log.sql \
        	--env local \
        	--format '{{ sql . "  " }}'
        ```
        
        ```bash
        atlas migrate apply --env local
        ```
        
    - 기존 테이블에 더미 데이터 생성
        - *seed_old_ad_effect_log.Js*
            
            ```bash
            // scripts/seed_old_ad_effect_log.js
            const mysql = require('mysql2/promise');
            const dayjs = require('dayjs');
            
            (async () => {
              console.log('🚀 Starting dummy data generation...');
              const startTime = new Date();
              
              const conn = await mysql.createConnection({
                host: '127.0.0.1',
                port: 3307,
                user: 'root',
                password: 'toor',
                database: 'partition_poc',
              });
            
              console.log('✅ Database connection established');
              
              const startDate = dayjs('2025-08-01T00:00:00');
              const endDate = dayjs('2025-08-30T23:59:59');
              const totalDays = 30; // 8월 1일 ~ 8월 30일
              const totalRecords = 20000;
              const progressInterval = Math.floor(totalRecords / 10); // 10%마다 진행상황 출력
            
              console.log(`📊 Generating ${totalRecords.toLocaleString()} records from ${startDate.format('YYYY-MM-DD')} to ${endDate.format('YYYY-MM-DD')}`);
            
              for (let i = 0; i < totalRecords; i++) {
                // 진행상황 로그 (10%마다)
                if (i % progressInterval === 0 && i > 0) {
                  const progress = Math.round((i / totalRecords) * 100);
                  const elapsed = Math.round((new Date() - startTime) / 1000);
                  console.log(`📈 Progress: ${progress}% (${i.toLocaleString()}/${totalRecords.toLocaleString()} records) - ${elapsed}s elapsed`);
                }
                
                // 랜덤 날짜 생성 (8월 1일 ~ 8월 30일)
                const randomDay = Math.floor(Math.random() * totalDays);
                const randomDate = startDate.add(randomDay, 'day');
                
                // 해당 날짜 내에서 랜덤 시간 생성
                const randomHour = Math.floor(Math.random() * 24);
                const randomMinute = Math.floor(Math.random() * 60);
                const randomSecond = Math.floor(Math.random() * 60);
                
                const createdAt = randomDate
                  .hour(randomHour)
                  .minute(randomMinute)
                  .second(randomSecond)
                  .format('YYYY-MM-DD HH:mm:ss');
                
                const dateOnly = randomDate.format('YYYY-MM-DD');
            
                await conn.query(
                  `INSERT INTO ad_effect_log_old 
                    (ad_id, ad_name, ad_start_time, ad_end_time, vehicle_number, route_code, date, exposure, views, attention, male_views, female_views, created_at)
                    VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)`,
                  [
                    Math.floor(Math.random() * 1000),
                    `Ad_${Math.floor(Math.random() * 100)}`,
                    '08:00',
                    '10:00',
                    `VN${Math.floor(Math.random() * 1000)}`,
                    `RC${Math.floor(Math.random() * 100)}`,
                    dateOnly,
                    Math.floor(Math.random() * 10000),
                    Math.floor(Math.random() * 5000),
                    Math.floor(Math.random() * 3000),
                    Math.floor(Math.random() * 2500),
                    Math.floor(Math.random() * 2500),
                    createdAt,
                  ]
                );
              }
            
              const endTime = new Date();
              const totalTime = Math.round((endTime - startTime) / 1000);
              const avgTimePerRecord = (totalTime / totalRecords * 1000).toFixed(2);
              
              console.log('✅ Dummy data generation completed!');
              console.log(`📊 Summary:`);
              console.log(`   - Total records: ${totalRecords.toLocaleString()}`);
              console.log(`   - Date range: ${startDate.format('YYYY-MM-DD')} ~ ${endDate.format('YYYY-MM-DD')}`);
              console.log(`   - Total time: ${totalTime}s`);
              console.log(`   - Average time per record: ${avgTimePerRecord}ms`);
              
              await conn.end();
              console.log('🔌 Database connection closed');
            })();
            ```
            
        - package.json
            
            ```bash
            {
              "name": "mysql-poc",
              "version": "1.0.0",
              "description": "MySQL Partition POC with dummy data seeding",
              "main": "index.js",
              "scripts": {
                "seed": "node scripts/seed_old_ad_effect_log.js",
                "copy": "node scripts/copy_chunk.js",
                "start": "node scripts/seed_old_ad_effect_log.js"
              },
              "dependencies": {
                "mysql2": "^3.6.5",
                "dayjs": "^1.11.10"
              },
              "devDependencies": {},
              "keywords": ["mysql", "partition", "poc"],
              "author": "",
              "license": "ISC"
            } 
            ```
            
        - 패키지 설치
            
            ```bash
            npm install
            ```
            
        - 더미 생성 스크립트 실행
            
            ```bash
            npm run seed
            OR
            node scripts/seed_old_ad_effect_log.js
            ```
            
        - 생성된 데이터 확인
            
            ```sql
            SELECT COUNT(*) as total_records FROM ad_effect_log_old;
            ----
            total_records
            20000
            ```
            
            ```sql
            SELECT MIN(date) as min_date, 
            	MAX(date) as max_date, 
            	COUNT(DISTINCT date) as unique_dates 
            FROM ad_effect_log_old;
            ----
            min_date        max_date        unique_dates
            2025-08-01      2025-08-30      30
            ```
            
    - 새 테이블 생성
        - 기존 테이블은 Range 파티셔닝 조건에 부적합해서 새로운 테이블에 데이터를 옮겨 파티셔닝 해야함
            
            > 당신은 created_at으로 RANGE COLUMNS 파티셔닝을 하려는데, 
            MySQL에서 파티셔닝 키가 모든 UNIQUE/PRIMARY KEY에 포함되어야 한다.
            현재 스키마는 idx만 PK여서 그대로는 불가능하다
            > 
            
            ```sql
            -- create_new_ad_effect_log.sql
            CREATE TABLE IF NOT EXISTS ad_effect_log_new (
              idx BIGINT AUTO_INCREMENT,
              ad_id BIGINT NOT NULL,
              ad_name VARCHAR(255) NOT NULL,
              ad_start_time VARCHAR(255) NOT NULL,
              ad_end_time VARCHAR(255) NOT NULL,
              vehicle_number VARCHAR(255) NOT NULL,
              route_code VARCHAR(255) NOT NULL,
              date DATE NOT NULL,
              exposure INT,
              views INT,
              attention INT,
              male_views INT,
              female_views INT,
              created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
              PRIMARY KEY (idx, created_at),
              INDEX IDX_vehicle_number (vehicle_number),
              INDEX IDX_route_code (route_code),
              INDEX IDX_date (date),
              INDEX IDX_ad_id (ad_id),
              INDEX IDX_created_at (created_at),
              INDEX IDX_idx_only (idx)  -- 기존 단독 조회 성능 보존
            ) ENGINE=InnoDB
              DEFAULT CHARSET=utf8mb4
              COLLATE=utf8mb4_0900_ai_ci;
            ```
            
    - 새 테이블에 파티셔닝 적용
        - 마이그레이션 파일 생성
            
            ```bash
            atlas migrate new add_partitioning
            ```
            
        - 생성된 빈 파일에 쿼리문 추가
            
            ```bash
               -- Add partitioning to ad_effect_log_new by created_at (2025-08-01 ~ 2025-08-04)
               ALTER TABLE ad_effect_log_new
               PARTITION BY RANGE COLUMNS (created_at) (
                 PARTITION p20250731 VALUES LESS THAN ('2025-08-01'),
                 PARTITION p20250801 VALUES LESS THAN ('2025-08-02'),
                 PARTITION p20250802 VALUES LESS THAN ('2025-08-03'),
                 PARTITION p20250803 VALUES LESS THAN ('2025-08-04'),
                 PARTITION p20250804 VALUES LESS THAN ('2025-08-05'),
                 PARTITION pMAX VALUES LESS THAN (MAXVALUE)
               );
            ```
            
        - 체크섭 업데이트
            
            ```bash
            atlas migrate hash --env local
            ```
            
        - 마이그레이션 진행
            
            ```bash
            atlas migrate apply --env local
            ```
            
    - 기존 테이블 → 새 테이블로 데이터 이관
        - *copy_chunk.Js*
            
            ```bash
            // scripts/copy_chunk.js
            const mysql = require('mysql2/promise');
            const dayjs = require('dayjs');
            
            (async () => {
              let conn;
              const startTime = new Date();
              
              try {
                console.log('🚀 Starting chunked data copy from ad_effect_log_old to ad_effect_log_new...');
                
                conn = await mysql.createConnection({
                  host: '127.0.0.1',
                  port: 3307,
                  user: 'root',
                  password: 'toor',
                  database: 'partition_poc',
                });
            
                console.log('✅ Database connection established');
            
                // 데이터 존재 여부 확인
                const [existingCount] = await conn.query(
                  'SELECT COUNT(*) as count FROM ad_effect_log_new'
                );
                
                if (existingCount[0].count > 0) {
                  console.log(`⚠️  Warning: ad_effect_log_new already contains ${existingCount[0].count.toLocaleString()} records`);
                  console.log('   Using INSERT IGNORE to skip duplicates');
                }
            
                // 날짜 범위 설정
                const start = dayjs('2025-08-01');
                const end = dayjs('2025-08-30');
                const totalDays = end.diff(start, 'day') + 1;
                
                console.log(`📊 Copying data from ${start.format('YYYY-MM-DD')} to ${end.format('YYYY-MM-DD')} (${totalDays} days)`);
            
                let currentDay = start;
                let totalCopied = 0;
                let dayCount = 0;
            
                while (currentDay.isBefore(end) || currentDay.isSame(end, 'day')) {
                  dayCount++;
                  const nextDay = currentDay.add(1, 'day');
                  const currentDate = currentDay.format('YYYY-MM-DD');
                  const nextDate = nextDay.format('YYYY-MM-DD');
                  
                  console.log(`📅 Processing day ${dayCount}/${totalDays}: ${currentDate}`);
                  
                  try {
                    // 해당 날짜의 데이터 수 확인
                    const [countResult] = await conn.query(
                      'SELECT COUNT(*) as count FROM ad_effect_log_old WHERE created_at >= ? AND created_at < ?',
                      [currentDate, nextDate]
                    );
                    
                    const dayRecordCount = countResult[0].count;
                    
                    if (dayRecordCount === 0) {
                      console.log(`   ⏭️  No data found for ${currentDate}, skipping...`);
                    } else {
                      // 데이터 복사 실행
                      const [insertResult] = await conn.query(`
                        INSERT IGNORE INTO ad_effect_log_new (
                          idx, ad_id, ad_name, ad_start_time, ad_end_time,
                          vehicle_number, route_code, date,
                          exposure, views, attention, male_views, female_views, created_at
                        )
                        SELECT 
                          idx, ad_id, ad_name, ad_start_time, ad_end_time,
                          vehicle_number, route_code, date,
                          exposure, views, attention, male_views, female_views, created_at
                        FROM ad_effect_log_old
                        WHERE created_at >= ? AND created_at < ?
                      `, [currentDate, nextDate]);
                      
                      const copiedCount = insertResult.affectedRows;
                      totalCopied += copiedCount;
                      
                      console.log(`   ✅ Copied ${copiedCount.toLocaleString()}/${dayRecordCount.toLocaleString()} records (${((copiedCount/dayRecordCount)*100).toFixed(1)}% success)`);
                    }
                    
                  } catch (dayError) {
                    console.error(`   ❌ Error processing ${currentDate}:`, dayError.message);
                    // 개별 날짜 에러가 있어도 계속 진행
                  }
                  
                  currentDay = nextDay;
                }
            
                // 최종 결과 확인
                const [finalCount] = await conn.query(
                  'SELECT COUNT(*) as count FROM ad_effect_log_new'
                );
                
                const endTime = new Date();
                const totalTime = Math.round((endTime - startTime) / 1000);
                
                console.log('\n🎉 Chunked copy completed successfully!');
                console.log('📊 Summary:');
                console.log(`   - Total days processed: ${dayCount}`);
                console.log(`   - Total records copied: ${totalCopied.toLocaleString()}`);
                console.log(`   - Final table count: ${finalCount[0].count.toLocaleString()}`);
                console.log(`   - Total time: ${totalTime}s`);
                console.log(`   - Average time per day: ${(totalTime/dayCount).toFixed(1)}s`);
            
              } catch (error) {
                console.error('❌ Fatal error during copy process:', error.message);
                console.error('Stack trace:', error.stack);
                process.exit(1);
              } finally {
                if (conn) {
                  await conn.end();
                  console.log('🔌 Database connection closed');
                }
              }
            })();
            
            ```
            
    - 파티셔닝 실행 스크립트
        
        ```json
        {
          "name": "mysql-poc",
          "version": "1.0.0",
          "description": "MySQL Partition POC with dummy data seeding",
          "main": "index.js",
          "scripts": {
            "seed": "node scripts/seed_old_ad_effect_log.js",
            "copy": "node scripts/copy_chunk.js",
            "partition": "node scripts/manage_partitions.js"
          },
          "dependencies": {
            "mysql2": "^3.6.5",
            "dayjs": "^1.11.10"
          },
          "devDependencies": {},
          "keywords": ["mysql", "partition", "poc"],
          "author": "",
          "license": "ISC"
        } 
        ```
        
        ```json
        // scripts/manage_partitions.js
        const mysql = require('mysql2/promise');
        const dayjs = require('dayjs');
        
        (async () => {
          const TABLE = 'ad_effect_log_new';
          const BACKUP_TABLE = 'backup_partition';
        
          const conn = await mysql.createConnection({
            host: '127.0.0.1',
            port: 3307,
            user: 'root',
            password: 'toor',
            database: 'partition_poc',
          });
        
          try {
            const now = dayjs();
            const todayPart = 'p' + now.format('YYYYMMDD');
            const tomorrow = now.add(1, 'day').format('YYYY-MM-DD');
            const fourDaysAgo = now.subtract(4, 'day');
            const fourDaysAgoPart = 'p' + fourDaysAgo.format('YYYYMMDD');
            const fourDaysAgoStr = fourDaysAgo.format('YYYY-MM-DD');
            const threeDaysAgo = now.subtract(3, 'day').format('YYYY-MM-DD');
        
            console.log(`[${dayjs().format('YYYY-MM-DD HH:mm:ss')}] 시작: 파티션 관리 스크립트`);
        
            // 1. 오늘 파티션이 있는지 확인
            const [todayRows] = await conn.query(
              `SELECT PARTITION_NAME FROM INFORMATION_SCHEMA.PARTITIONS WHERE TABLE_SCHEMA=DATABASE() AND TABLE_NAME=? AND PARTITION_NAME=?`,
              [TABLE, todayPart]
            );
            if (todayRows.length === 0) {
              console.log(`[${dayjs().format('YYYY-MM-DD HH:mm:ss')}] 오늘 파티션 ${todayPart} 없음. 생성 시도...`);
              const alterSql = `ALTER TABLE \`${TABLE}\` ADD PARTITION (PARTITION ${todayPart} VALUES LESS THAN ('${tomorrow}'));`;
              try {
                await conn.query(alterSql);
                console.log(`[${dayjs().format('YYYY-MM-DD HH:mm:ss')}] 파티션 ${todayPart} 생성 완료.`);
              } catch (err) {
                console.error(`[${dayjs().format('YYYY-MM-DD HH:mm:ss')}] ERROR: 파티션 ${todayPart} 생성 실패.`, err.message);
              }
            } else {
              console.log(`[${dayjs().format('YYYY-MM-DD HH:mm:ss')}] 오늘 파티션 ${todayPart} 이미 존재.`);
            }
        
            // 2. 4일 전 파티션이 있는지 확인
            const [oldPartRows] = await conn.query(
              `SELECT PARTITION_NAME FROM INFORMATION_SCHEMA.PARTITIONS WHERE TABLE_SCHEMA=DATABASE() AND TABLE_NAME=? AND PARTITION_NAME=?`,
              [TABLE, fourDaysAgoPart]
            );
            if (oldPartRows.length === 0) {
              console.log(`[${dayjs().format('YYYY-MM-DD HH:mm:ss')}] 4일 전 파티션 ${fourDaysAgoPart} 없음. 백업 및 삭제 스킵.`);
            } else {
              // 3. 백업 테이블이 없으면 생성
              await conn.query(`
                CREATE TABLE IF NOT EXISTS \`${BACKUP_TABLE}\` LIKE \`${TABLE}\`
              `);
              console.log(`[${dayjs().format('YYYY-MM-DD HH:mm:ss')}] backup_partition 테이블 준비 완료.`);
        
              // 4. 4일 전 파티션 데이터 백업
              // created_at >= fourDaysAgoStr AND created_at < threeDaysAgo
              const [countRows] = await conn.query(
                `SELECT COUNT(*) as cnt FROM \`${TABLE}\` WHERE created_at >= ? AND created_at < ?`,
                [fourDaysAgoStr, threeDaysAgo]
              );
              const backupCount = countRows[0].cnt;
              if (backupCount === 0) {
                console.log(`[${dayjs().format('YYYY-MM-DD HH:mm:ss')}] 4일 전 파티션 데이터 없음. 백업 스킵.`);
              } else {
                console.log(`[${dayjs().format('YYYY-MM-DD HH:mm:ss')}] 4일 전 파티션 데이터 ${backupCount}건 백업 시작...`);
                const [insertResult] = await conn.query(
                  `INSERT INTO \`${BACKUP_TABLE}\` SELECT * FROM \`${TABLE}\` WHERE created_at >= ? AND created_at < ?`,
                  [fourDaysAgoStr, threeDaysAgo]
                );
                console.log(`[${dayjs().format('YYYY-MM-DD HH:mm:ss')}] 백업 완료: ${insertResult.affectedRows}건`);
              }
        
              // 5. 파티션 삭제
              try {
                await conn.query(`ALTER TABLE \`${TABLE}\` DROP PARTITION \`${fourDaysAgoPart}\``);
                console.log(`[${dayjs().format('YYYY-MM-DD HH:mm:ss')}] 4일 전 파티션 ${fourDaysAgoPart} 삭제 완료.`);
              } catch (err) {
                console.error(`[${dayjs().format('YYYY-MM-DD HH:mm:ss')}] ERROR: 4일 전 파티션 삭제 실패.`, err.message);
              }
            }
        
            console.log(`[${dayjs().format('YYYY-MM-DD HH:mm:ss')}] 파티션 관리 스크립트 종료.`);
          } catch (err) {
            console.error(`[${dayjs().format('YYYY-MM-DD HH:mm:ss')}] 치명적 오류 발생:`, err.message);
            process.exit(1);
          } finally {
            await conn.end();
          }
        })();
        ```
        

---

## 스크립트 기반 파티셔닝 PoC

- DB 셋팅
    - .env
        
        ```bash
        DATABASE_URL="mysql://root:toor@127.0.0.1:3309/addd-nks"
        OPERATION_DATABASE_URL="mysql://root:toor@localhost:3308/bus_operation_logs"
        AI_SERVER_DATABASE_URL="mysql://addd:WaTU%23R%40GVN12%25@175.106.96.249:3306/ai_dev"
        PORT=4000
        NODE_ENV=development
        PORT=4000
        NODE_ENV=development
        DATA_SCALE=1.2
        api - .env
        NCP_ACCESS_KEY=
        NCP_SECRET_KEY=
        NCP_BUCKET_NAME=
        DB_HOST=127.0.0.1
        # DB_HOST=host.docker.internal
        DB_PORT=3309
        DB_USERNAME=root
        DB_PASSWORD=toor
        DB_DATABASE=addd-nks
        DB_SSL=false
        NODE_ENV=development
        # RabbitMQ 연결 정보
        RABBITMQ_HOST=.kr.vrmq.naverncp.com
        RABBITMQ_PORT=
        RABBITMQ_USERNAME=
        RABBITMQ_PASSWORD=
        RABBITMQ_VHOST=/
        RABBIT_LOCAL_MODE=true
        bus-operation-log .env
        DATABASE_HOST=localhost
        DATABASE_PORT=3308
        DATABASE_USER=root
        DATABASE_PASSWORD=toor
        DATABASE_NAME=bus_operation_logs
        DATABASE_NAME2=addd-nks-db
        # BUS_SERVICE_KEY=p4%2BdHTkH%2BM9uJ477NPnT4B2TF8zMAkT04k2CrW4Ae8L%2FLVUtrO%2BY7OUyyqkAZEUZDKOfoH8sApTUaHnYvV6b8A%3D%3D
        BUS_SERVICE_KEY=p4+dHTkH+M9uJ477NPnT4B2TF8zMAkT04k2CrW4Ae8L/LVUtrO+Y7OUyyqkAZEUZDKOfoH8sApTUaHnYvV6b8A==
        ```
        
    - docker-compose.dev.yml
        
        ```yaml
        version: '0.0.1'
        
        services:
          # MySQL 데이터베이스 (addd-nks)
          mysql:
            image: mysql:8.0
            container_name: addd-nks-db
            restart: always
            environment:
              MYSQL_ROOT_PASSWORD: toor
              MYSQL_DATABASE: addd-nks
            ports:
              - "3309:3306"
            volumes:
              - mysql_data:/var/lib/mysql
            command: --default-authentication-plugin=mysql_native_password
            networks:
              - addd-network
        
          # Bus Operation Logs 데이터베이스
          mysql-bus-logs:
            image: mysql:8.0
            container_name: bus-operation-logs-db
            restart: always
            environment:
              MYSQL_ROOT_PASSWORD: toor
              MYSQL_DATABASE: bus_operation_logs
            ports:
              - "3308:3306"
            volumes:
              - bus_logs_data:/var/lib/mysql
            command: --default-authentication-plugin=mysql_native_password
            networks:
              - addd-network
        
        volumes:
          mysql_data:
          bus_logs_data:
        
        networks:
          addd-network:
            driver: bridge 
        ```
        
    - **데이터베이스 컨테이너 실행**
        
        ```bash
        docker-compose -f docker-compose.dev.yml up -d
        ```
        
    - prisma 마이그레이션 실행
        
        ```bash
        npm run adddnks-db:migrate:dev
        ```
        
- ad_effect_log 테이블에 더미 데이터 생성
    - 의존성 패키지 설치
        
        ```bash
        pnpm add mysql2 dayjs dotenv
        ```
        
    - seed_ad_effect_log.js
        - 오늘 날짜를 기준으로 60일전까지의 더미 데이터를 랜덤 생성
        
        ```jsx
        // scripts/seed_ad_effect_log.js
        const mysql = require('mysql2/promise');
        const dayjs = require('dayjs');
        require('dotenv').config();
        
        (async () => {
          console.log('🚀 Starting ad_effect_log dummy data generation...');
          const startTime = new Date();
          
          // 환경변수에서 데이터베이스 설정 가져오기
          const dbConfig = {
            host: process.env.DB_HOST || '127.0.0.1',
            port: parseInt(process.env.DB_PORT) || 3309,
            user: process.env.DB_USERNAME || 'root',
            password: process.env.DB_PASSWORD || 'toor',
            database: process.env.DB_DATABASE || 'addd-nks',
          };
        
          console.log('📋 Database Configuration:');
          console.log(`   - Host: ${dbConfig.host}:${dbConfig.port}`);
          console.log(`   - Database: ${dbConfig.database}`);
          console.log(`   - User: ${dbConfig.user}`);
          
          const conn = await mysql.createConnection(dbConfig);
          console.log('✅ Database connection established');
          
          // 오늘 날짜를 기준으로 60일 전까지의 데이터 생성
          const today = dayjs();
          const startDate = today.subtract(60, 'day');
          const endDate = today;
          const totalDays = 60;
          const totalRecords = 50000; // 60일치 데이터를 위해 더 많은 레코드 생성
          const progressInterval = Math.floor(totalRecords / 10); // 10%마다 진행상황 출력
        
          console.log(`📊 Generating ${totalRecords.toLocaleString()} records`);
          console.log(`📅 Date range: ${startDate.format('YYYY-MM-DD')} to ${endDate.format('YYYY-MM-DD')} (${totalDays} days)`);
        
          for (let i = 0; i < totalRecords; i++) {
            // 진행상황 로그 (10%마다)
            if (i % progressInterval === 0 && i > 0) {
              const progress = Math.round((i / totalRecords) * 100);
              const elapsed = Math.round((new Date() - startTime) / 1000);
              const remaining = Math.round((elapsed / i) * (totalRecords - i));
              console.log(`📈 Progress: ${progress}% (${i.toLocaleString()}/${totalRecords.toLocaleString()} records) - ${elapsed}s elapsed, ~${remaining}s remaining`);
            }
            
            // 랜덤 날짜 생성 (60일 전 ~ 오늘)
            const randomDay = Math.floor(Math.random() * totalDays);
            const randomDate = startDate.add(randomDay, 'day');
            
            // 해당 날짜 내에서 랜덤 시간 생성
            const randomHour = Math.floor(Math.random() * 24);
            const randomMinute = Math.floor(Math.random() * 60);
            const randomSecond = Math.floor(Math.random() * 60);
            
            const createdAt = randomDate
              .hour(randomHour)
              .minute(randomMinute)
              .second(randomSecond)
              .format('YYYY-MM-DD HH:mm:ss');
            
            const dateOnly = randomDate.format('YYYY-MM-DD');
        
            // ad_effect_log 테이블 구조에 맞는 데이터 생성
            await conn.query(
              `INSERT INTO ad_effect_log 
                (ad_id, ad_name, ad_start_time, ad_end_time, vehicle_number, route_code, date, exposure, views, attention, male_views, female_views, created_at)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)`,
              [
                Math.floor(Math.random() * 1000) + 1, // ad_id (1-1000)
                `Ad_${Math.floor(Math.random() * 100)}_${randomDate.format('MMDD')}`, // ad_name with date
                `${String(Math.floor(Math.random() * 24)).padStart(2, '0')}:${String(Math.floor(Math.random() * 60)).padStart(2, '0')}`, // ad_start_time
                `${String(Math.floor(Math.random() * 24)).padStart(2, '0')}:${String(Math.floor(Math.random() * 60)).padStart(2, '0')}`, // ad_end_time
                `VN${String(Math.floor(Math.random() * 1000)).padStart(3, '0')}`, // vehicle_number
                `RC${String(Math.floor(Math.random() * 100)).padStart(2, '0')}`, // route_code
                dateOnly, // date
                Math.floor(Math.random() * 10000) + 100, // exposure (100-10100)
                Math.floor(Math.random() * 5000) + 50, // views (50-5050)
                Math.floor(Math.random() * 3000) + 30, // attention (30-3030)
                Math.floor(Math.random() * 2500) + 25, // male_views (25-2525)
                Math.floor(Math.random() * 2500) + 25, // female_views (25-2525)
                createdAt, // created_at
              ]
            );
          }
        
          const endTime = new Date();
          const totalTime = Math.round((endTime - startTime) / 1000);
          const avgTimePerRecord = (totalTime / totalRecords * 1000).toFixed(2);
          
          console.log('✅ ad_effect_log dummy data generation completed!');
          console.log(`📊 Summary:`);
          console.log(`   - Total records: ${totalRecords.toLocaleString()}`);
          console.log(`   - Date range: ${startDate.format('YYYY-MM-DD')} ~ ${endDate.format('YYYY-MM-DD')}`);
          console.log(`   - Total time: ${totalTime}s`);
          console.log(`   - Average time per record: ${avgTimePerRecord}ms`);
          console.log(`   - Records per day: ${Math.round(totalRecords / totalDays)}`);
          
          await conn.end();
          console.log('🔌 Database connection closed');
        })(); 
        ```
        
    - 실행 명령어
        
        ```bash
        node scripts/seed_ad_effect_log.js
        ```
        
- ad_effect_log_v2 테이블 생성
    - create_ad_effect_log_v2.js
        
        ```jsx
        // scripts/create_ad_effect_log_v2.js
        // 파티셔닝에 적합한 ad_effect_log_v2 테이블 생성
        const mysql = require('mysql2/promise');
        require('dotenv').config();
        
        (async () => {
          console.log('🚀 ad_effect_log_v2 테이블 생성 시작...');
        
          // 환경변수에서 데이터베이스 설정 가져오기
          const dbConfig = {
            host: process.env.DB_HOST || '127.0.0.1',
            port: parseInt(process.env.DB_PORT) || 3309,
            user: process.env.DB_USERNAME || 'root',
            password: process.env.DB_PASSWORD || 'toor',
            database: process.env.DB_DATABASE || 'addd-nks',
          };
        
          console.log('📋 Database Configuration:');
          console.log(`   - Host: ${dbConfig.host}:${dbConfig.port}`);
          console.log(`   - Database: ${dbConfig.database}`);
          console.log(`   - User: ${dbConfig.user}`);
        
          const conn = await mysql.createConnection(dbConfig);
        
          try {
            // 1. 기존 테이블 존재 확인
            const [tableExists] = await conn.query(`
              SELECT COUNT(*) as count 
              FROM INFORMATION_SCHEMA.TABLES 
              WHERE TABLE_SCHEMA = ? 
              AND TABLE_NAME = 'ad_effect_log_v2'
            `, [dbConfig.database]);
        
            if (tableExists[0].count > 0) {
              console.log('⚠️  ad_effect_log_v2 테이블이 이미 존재합니다.');
              console.log('💡 기존 테이블을 삭제하고 새로 생성하시겠습니까? (y/N)');
              return;
            }
        
            // 2. ad_effect_log_v2 테이블 생성 (파티셔닝에 최적화된 구조)
            const createTableSql = `
              CREATE TABLE \`ad_effect_log_v2\` (
                \`idx\` int NOT NULL AUTO_INCREMENT,
                \`ad_id\` bigint NOT NULL,
                \`ad_name\` varchar(255) NOT NULL,
                \`ad_start_time\` varchar(255) NOT NULL,
                \`ad_end_time\` varchar(255) NOT NULL,
                \`vehicle_number\` varchar(255) NOT NULL,
                \`route_code\` varchar(255) NOT NULL,
                \`date\` date NOT NULL,
                \`exposure\` int NOT NULL,
                \`views\` int NOT NULL,
                \`attention\` int NOT NULL,
                \`male_views\` int NOT NULL,
                \`female_views\` int NOT NULL,
                \`created_at\` datetime NOT NULL,
                PRIMARY KEY (\`idx\`),
                KEY \`IDX_ad_effect_log_v2_route_code\` (\`route_code\`),
                KEY \`IDX_ad_effect_log_v2_vehicle_number\` (\`vehicle_number\`),
                KEY \`IDX_ad_effect_log_v2_ad_id\` (\`ad_id\`),
                KEY \`IDX_ad_effect_log_v2_date\` (\`date\`),
                KEY \`IDX_ad_effect_log_v2_created_at\` (\`created_at\`)
              ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
            `;
        
            console.log('📝 테이블 생성 SQL:');
            console.log(createTableSql);
        
            await conn.query(createTableSql);
            console.log('✅ ad_effect_log_v2 테이블 생성 완료!');
        
            // 3. 테이블 구조 확인
            const [columns] = await conn.query(`
              SELECT COLUMN_NAME, DATA_TYPE, IS_NULLABLE, COLUMN_DEFAULT
              FROM INFORMATION_SCHEMA.COLUMNS 
              WHERE TABLE_SCHEMA = ? 
              AND TABLE_NAME = 'ad_effect_log_v2'
              ORDER BY ORDINAL_POSITION
            `, [dbConfig.database]);
        
            console.log('\n📊 생성된 테이블 구조:');
            columns.forEach(col => {
              console.log(`   ${col.COLUMN_NAME}: ${col.DATA_TYPE} ${col.IS_NULLABLE === 'NO' ? 'NOT NULL' : 'NULL'} ${col.COLUMN_DEFAULT ? `DEFAULT ${col.COLUMN_DEFAULT}` : ''}`);
            });
        
            console.log('\n🎉 ad_effect_log_v2 테이블이 파티셔닝에 적합하게 생성되었습니다!');
            console.log('💡 다음 단계:');
            console.log('   1. npm run setup-partitions (파티션 설정)');
            console.log('   2. npm run copy-ad-effect-log (데이터 복사)');
        
          } catch (error) {
            console.error('❌ 테이블 생성 실패:', error.message);
            process.exit(1);
          } finally {
            await conn.end();
            console.log('🔌 DB 연결 종료');
          }
        })(); 
        ```
        
    - 실행 명령어
        
        ```bash
        node scripts/create_ad_effect_log_v2.js
        ```
        
    - 실행 결과
        
        ```jsx
        [dotenv@17.2.1] injecting env (28) from .env -- tip: 🔐 prevent building .env in docker: https://dotenvx.com/prebuild
        🚀 ad_effect_log_v2 테이블 생성 시작...
        📋 Database Configuration:
           - Host: 127.0.0.1:3309
           - Database: addd-nks
           - User: root
        📝 테이블 생성 SQL:
        
              CREATE TABLE `ad_effect_log_v2` (
                `idx` int NOT NULL AUTO_INCREMENT,
                `ad_id` bigint NOT NULL,
                `ad_name` varchar(255) NOT NULL,
                `ad_start_time` varchar(255) NOT NULL,
                `ad_end_time` varchar(255) NOT NULL,
                `vehicle_number` varchar(255) NOT NULL,
                `route_code` varchar(255) NOT NULL,
                `date` date NOT NULL,
                `exposure` int NOT NULL,
                `views` int NOT NULL,
                `attention` int NOT NULL,
                `male_views` int NOT NULL,
                `female_views` int NOT NULL,
                `created_at` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`idx`, `created_at`),
                INDEX `IDX_ad_id` (`ad_id`),
                INDEX `IDX_created_at` (`created_at`),
                INDEX `IDX_date` (`date`),
                INDEX `IDX_idx_only` (`idx`),
                INDEX `IDX_route_code` (`route_code`),
                INDEX `IDX_vehicle_number` (`vehicle_number`)
              ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
            
        ✅ ad_effect_log_v2 테이블 생성 완료!
        
        📊 생성된 테이블 구조:
           idx: int NOT NULL 
           ad_id: bigint NOT NULL 
           ad_name: varchar NOT NULL 
           ad_start_time: varchar NOT NULL 
           ad_end_time: varchar NOT NULL 
           vehicle_number: varchar NOT NULL 
           route_code: varchar NOT NULL 
           date: date NOT NULL 
           exposure: int NOT NULL 
           views: int NOT NULL 
           attention: int NOT NULL 
           male_views: int NOT NULL 
           female_views: int NOT NULL 
           created_at: datetime NOT NULL DEFAULT CURRENT_TIMESTAMP
        
        🎉 ad_effect_log_v2 테이블이 파티셔닝에 적합하게 생성되었습니다!
        💡 다음 단계:
           1. npm run setup-partitions (파티션 설정)
           2. npm run copy-ad-effect-log (데이터 복사)
        🔌 DB 연결 종료
        ```
        
- ad_effect_log_v2 파티션 설정
    - setup_partitions
        
        ```jsx
        // scripts/setup_partitions.js
        // Prisma로 생성된 테이블에 파티션을 추가하는 스크립트
        const mysql = require('mysql2/promise');
        const dayjs = require('dayjs');
        require('dotenv').config();
        
        (async () => {
          const startTime = new Date();
          console.log(`🚀 [${startTime.toISOString()}] 파티션 초기 설정 시작...`);
        
          // 환경변수에서 데이터베이스 설정 가져오기
          const dbConfig = {
            host: process.env.DB_HOST || '127.0.0.1',
            port: parseInt(process.env.DB_PORT) || 3309,
            user: process.env.DB_USERNAME || 'root',
            password: process.env.DB_PASSWORD || 'toor',
            database: process.env.DB_DATABASE || 'addd-nks',
          };
        
          // 타겟 테이블 설정 (환경변수 또는 기본값)
          const TABLE_NAME = process.env.TARGET_TABLE || 'ad_effect_log_v2';
          const PARTITION_RANGE_DAYS = parseInt(process.env.PARTITION_RANGE_DAYS) || 60; // 기본 60일 전까지
        
          console.log('📋 Database Configuration:');
          console.log(`   - Host: ${dbConfig.host}:${dbConfig.port}`);
          console.log(`   - Database: ${dbConfig.database}`);
          console.log(`   - User: ${dbConfig.user}`);
          console.log(`   - Target Table: ${TABLE_NAME}`);
          console.log(`   - Partition Range: ±${PARTITION_RANGE_DAYS} days`);
        
          const conn = await mysql.createConnection(dbConfig);
        
          try {
            // 1. 테이블 존재 확인
            const [tableExists] = await conn.query(`
              SELECT COUNT(*) as count 
              FROM INFORMATION_SCHEMA.TABLES 
              WHERE TABLE_SCHEMA = ? 
              AND TABLE_NAME = ?
            `, [dbConfig.database, TABLE_NAME]);
        
            if (tableExists[0].count === 0) {
              console.error(`❌ 테이블 ${TABLE_NAME}이 존재하지 않습니다.`);
              console.log('💡 먼저 Prisma로 테이블을 생성하세요:');
              console.log('   npm run adddnks-db:migrate:dev');
              return;
            }
        
            // 2. 현재 파티션 상태 확인
            const [partitions] = await conn.query(`
              SELECT 
                PARTITION_NAME,
                PARTITION_DESCRIPTION,
                TABLE_ROWS,
                PARTITION_ORDINAL_POSITION
              FROM INFORMATION_SCHEMA.PARTITIONS 
              WHERE TABLE_SCHEMA = ? 
              AND TABLE_NAME = ?
              ORDER BY PARTITION_ORDINAL_POSITION
            `, [dbConfig.database, TABLE_NAME]);
        
            console.log(`📊 현재 파티션 상태: ${partitions.length}개 파티션`);
        
            // 3. 기존 파티션이 있고 의미있는 파티션이면 중단
            if (partitions.length > 1 || (partitions.length === 1 && partitions[0].PARTITION_NAME !== null)) {
              console.log(`⚠️  테이블 ${TABLE_NAME}에 이미 의미있는 파티션이 설정되어 있습니다.`);
              console.log('💡 파티션 관리만 하려면: npm run manage-partitions-only');
              return;
            }
        
            // 4. 테이블 구조 확인 (created_at 컬럼 존재 여부)
            const [columns] = await conn.query(`
              SELECT COLUMN_NAME, DATA_TYPE, IS_NULLABLE
              FROM INFORMATION_SCHEMA.COLUMNS 
              WHERE TABLE_SCHEMA = ? 
              AND TABLE_NAME = ?
              AND COLUMN_NAME = 'created_at'
            `, [dbConfig.database, TABLE_NAME]);
        
            if (columns.length === 0) {
              console.error(`❌ 테이블 ${TABLE_NAME}에 'created_at' 컬럼이 없습니다.`);
              console.log('💡 파티션을 위해서는 created_at 컬럼이 필요합니다.');
              return;
            }
        
            console.log(`✅ 테이블 ${TABLE_NAME}의 created_at 컬럼 확인됨`);
        
            // 5. 파티션 설정
            console.log(`📋 ${TABLE_NAME} 테이블에 파티션 설정 중...`);
            
            const now = dayjs();
            const startDate = now.subtract(PARTITION_RANGE_DAYS, 'day'); // 설정된 일수 전부터
            const endDate = now; // 오늘까지
            
            let partitionSql = `ALTER TABLE \`${TABLE_NAME}\` PARTITION BY RANGE COLUMNS (created_at) (`;
            
            // 파티션 목록 생성
            const partitionList = [];
            let currentDate = startDate;
            
            while (currentDate.isBefore(endDate) || currentDate.isSame(endDate, 'day')) {
              const nextDate = currentDate.add(1, 'day');
              const partitionName = 'p' + currentDate.format('YYYYMMDD');
              const nextDateStr = nextDate.format('YYYY-MM-DD');
              
              partitionList.push(`PARTITION ${partitionName} VALUES LESS THAN ('${nextDateStr}')`);
              currentDate = nextDate;
            }
            
            // MAX 파티션 추가
            partitionList.push('PARTITION pMAX VALUES LESS THAN (MAXVALUE)');
            
            partitionSql += partitionList.join(', ') + ');';
            
            console.log('📝 파티션 SQL:');
            console.log(partitionSql);
            
            // 6. 파티션 적용
            try {
              await conn.query(partitionSql);
              console.log('✅ 파티션 설정 완료!');
              
              // 7. 설정된 파티션 확인
              const [newPartitions] = await conn.query(`
                SELECT 
                  PARTITION_NAME,
                  PARTITION_DESCRIPTION,
                  TABLE_ROWS
                FROM INFORMATION_SCHEMA.PARTITIONS 
                WHERE TABLE_SCHEMA = ? 
                AND TABLE_NAME = ?
                ORDER BY PARTITION_ORDINAL_POSITION
              `, [dbConfig.database, TABLE_NAME]);
        
              console.log('\n📊 설정된 파티션:');
              newPartitions.forEach(part => {
                const rows = part.TABLE_ROWS || 0;
                console.log(`   ${part.PARTITION_NAME}: ${part.PARTITION_DESCRIPTION || 'MAXVALUE'} (${rows.toLocaleString()} rows)`);
              });
              
            } catch (error) {
              console.error('❌ 파티션 설정 실패:', error.message);
              console.log('💡 가능한 원인:');
              console.log('   - 테이블에 데이터가 있는 경우');
              console.log('   - created_at 컬럼에 NULL 값이 있는 경우');
              console.log('   - 테이블 구조가 파티션과 호환되지 않는 경우');
              console.log('   - 테이블에 인덱스나 제약조건이 있는 경우');
              return;
            }
        
            const endTime = new Date();
            const duration = Math.round((endTime - startTime) / 1000);
            
            console.log(`\n🎉 파티션 초기 설정 완료! (${duration}s)`);
            console.log('💡 이제 다음 명령어로 파티션을 관리할 수 있습니다:');
            console.log('   npm run manage-partitions-only');
        
          } catch (error) {
            console.error('❌ 치명적 오류:', error.message);
            process.exit(1);
          } finally {
            await conn.end();
            console.log('🔌 DB 연결 종료');
          }
        })(); 
        ```
        
    - 실행 명령어
        
        ```bash
        node scripts/setup-partitions.js
        ```
        
    - 실행 결과
        
        ```jsx
        > bus_dashboard_server@0.0.1 setup-partitions
        > node scripts/setup_partitions.js
        
        [dotenv@17.2.1] injecting env (28) from .env -- tip: 🛠️  run anywhere with `dotenvx run -- yourcommand`
        🚀 [2025-08-06T01:57:39.196Z] 파티션 초기 설정 시작...
        📋 Database Configuration:
           - Host: 127.0.0.1:3309
           - Database: addd-nks
           - User: root
           - Target Table: ad_effect_log_v2
           - Partition Range: ±60 days
        📊 현재 파티션 상태: 1개 파티션
        ✅ 테이블 ad_effect_log_v2의 created_at 컬럼 확인됨
        📋 ad_effect_log_v2 테이블에 파티션 설정 중...
        📝 파티션 SQL:
        ALTER TABLE `ad_effect_log_v2` PARTITION BY RANGE COLUMNS (created_at) (PARTITION p20250607 VALUES LESS THAN ('2025-06-08'), PARTITION p20250608 VALUES LESS THAN ('2025-06-09'), PARTITION p20250609 VALUES LESS THAN ('2025-06-10'), PARTITION p20250610 VALUES LESS THAN ('2025-06-11'), PARTITION p20250611 VALUES LESS THAN ('2025-06-12'), PARTITION p20250612 VALUES LESS THAN ('2025-06-13'), PARTITION p20250613 VALUES LESS THAN ('2025-06-14'), PARTITION p20250614 VALUES LESS THAN ('2025-06-15'), PARTITION p20250615 VALUES LESS THAN ('2025-06-16'), PARTITION p20250616 VALUES LESS THAN ('2025-06-17'), PARTITION p20250617 VALUES LESS THAN ('2025-06-18'), PARTITION p20250618 VALUES LESS THAN ('2025-06-19'), PARTITION p20250619 VALUES LESS THAN ('2025-06-20'), PARTITION p20250620 VALUES LESS THAN ('2025-06-21'), PARTITION p20250621 VALUES LESS THAN ('2025-06-22'), PARTITION p20250622 VALUES LESS THAN ('2025-06-23'), PARTITION p20250623 VALUES LESS THAN ('2025-06-24'), PARTITION p20250624 VALUES LESS THAN ('2025-06-25'), PARTITION p20250625 VALUES LESS THAN ('2025-06-26'), PARTITION p20250626 VALUES LESS THAN ('2025-06-27'), PARTITION p20250627 VALUES LESS THAN ('2025-06-28'), PARTITION p20250628 VALUES LESS THAN ('2025-06-29'), PARTITION p20250629 VALUES LESS THAN ('2025-06-30'), PARTITION p20250630 VALUES LESS THAN ('2025-07-01'), PARTITION p20250701 VALUES LESS THAN ('2025-07-02'), PARTITION p20250702 VALUES LESS THAN ('2025-07-03'), PARTITION p20250703 VALUES LESS THAN ('2025-07-04'), PARTITION p20250704 VALUES LESS THAN ('2025-07-05'), PARTITION p20250705 VALUES LESS THAN ('2025-07-06'), PARTITION p20250706 VALUES LESS THAN ('2025-07-07'), PARTITION p20250707 VALUES LESS THAN ('2025-07-08'), PARTITION p20250708 VALUES LESS THAN ('2025-07-09'), PARTITION p20250709 VALUES LESS THAN ('2025-07-10'), PARTITION p20250710 VALUES LESS THAN ('2025-07-11'), PARTITION p20250711 VALUES LESS THAN ('2025-07-12'), PARTITION p20250712 VALUES LESS THAN ('2025-07-13'), PARTITION p20250713 VALUES LESS THAN ('2025-07-14'), PARTITION p20250714 VALUES LESS THAN ('2025-07-15'), PARTITION p20250715 VALUES LESS THAN ('2025-07-16'), PARTITION p20250716 VALUES LESS THAN ('2025-07-17'), PARTITION p20250717 VALUES LESS THAN ('2025-07-18'), PARTITION p20250718 VALUES LESS THAN ('2025-07-19'), PARTITION p20250719 VALUES LESS THAN ('2025-07-20'), PARTITION p20250720 VALUES LESS THAN ('2025-07-21'), PARTITION p20250721 VALUES LESS THAN ('2025-07-22'), PARTITION p20250722 VALUES LESS THAN ('2025-07-23'), PARTITION p20250723 VALUES LESS THAN ('2025-07-24'), PARTITION p20250724 VALUES LESS THAN ('2025-07-25'), PARTITION p20250725 VALUES LESS THAN ('2025-07-26'), PARTITION p20250726 VALUES LESS THAN ('2025-07-27'), PARTITION p20250727 VALUES LESS THAN ('2025-07-28'), PARTITION p20250728 VALUES LESS THAN ('2025-07-29'), PARTITION p20250729 VALUES LESS THAN ('2025-07-30'), PARTITION p20250730 VALUES LESS THAN ('2025-07-31'), PARTITION p20250731 VALUES LESS THAN ('2025-08-01'), PARTITION p20250801 VALUES LESS THAN ('2025-08-02'), PARTITION p20250802 VALUES LESS THAN ('2025-08-03'), PARTITION p20250803 VALUES LESS THAN ('2025-08-04'), PARTITION p20250804 VALUES LESS THAN ('2025-08-05'), PARTITION p20250805 VALUES LESS THAN ('2025-08-06'), PARTITION p20250806 VALUES LESS THAN ('2025-08-07'), PARTITION pMAX VALUES LESS THAN (MAXVALUE));
        ✅ 파티션 설정 완료!
        
        📊 설정된 파티션:
           p20250607: '2025-06-08' (0 rows)
           p20250608: '2025-06-09' (0 rows)
           p20250609: '2025-06-10' (0 rows)
           p20250610: '2025-06-11' (0 rows)
           p20250611: '2025-06-12' (0 rows)
           p20250612: '2025-06-13' (0 rows)
           p20250613: '2025-06-14' (0 rows)
           p20250614: '2025-06-15' (0 rows)
           p20250615: '2025-06-16' (0 rows)
           p20250616: '2025-06-17' (0 rows)
           p20250617: '2025-06-18' (0 rows)
           p20250618: '2025-06-19' (0 rows)
           p20250619: '2025-06-20' (0 rows)
           p20250620: '2025-06-21' (0 rows)
           p20250621: '2025-06-22' (0 rows)
           p20250622: '2025-06-23' (0 rows)
           p20250623: '2025-06-24' (0 rows)
           p20250624: '2025-06-25' (0 rows)
           p20250625: '2025-06-26' (0 rows)
           p20250626: '2025-06-27' (0 rows)
           p20250627: '2025-06-28' (0 rows)
           p20250628: '2025-06-29' (0 rows)
           p20250629: '2025-06-30' (0 rows)
           p20250630: '2025-07-01' (0 rows)
           p20250701: '2025-07-02' (0 rows)
           p20250702: '2025-07-03' (0 rows)
           p20250703: '2025-07-04' (0 rows)
           p20250704: '2025-07-05' (0 rows)
           p20250705: '2025-07-06' (0 rows)
           p20250706: '2025-07-07' (0 rows)
           p20250707: '2025-07-08' (0 rows)
           p20250708: '2025-07-09' (0 rows)
           p20250709: '2025-07-10' (0 rows)
           p20250710: '2025-07-11' (0 rows)
           p20250711: '2025-07-12' (0 rows)
           p20250712: '2025-07-13' (0 rows)
           p20250713: '2025-07-14' (0 rows)
           p20250714: '2025-07-15' (0 rows)
           p20250715: '2025-07-16' (0 rows)
           p20250716: '2025-07-17' (0 rows)
           p20250717: '2025-07-18' (0 rows)
           p20250718: '2025-07-19' (0 rows)
           p20250719: '2025-07-20' (0 rows)
           p20250720: '2025-07-21' (0 rows)
           p20250721: '2025-07-22' (0 rows)
           p20250722: '2025-07-23' (0 rows)
           p20250723: '2025-07-24' (0 rows)
           p20250724: '2025-07-25' (0 rows)
           p20250725: '2025-07-26' (0 rows)
           p20250726: '2025-07-27' (0 rows)
           p20250727: '2025-07-28' (0 rows)
           p20250728: '2025-07-29' (0 rows)
           p20250729: '2025-07-30' (0 rows)
           p20250730: '2025-07-31' (0 rows)
           p20250731: '2025-08-01' (0 rows)
           p20250801: '2025-08-02' (0 rows)
           p20250802: '2025-08-03' (0 rows)
           p20250803: '2025-08-04' (0 rows)
           p20250804: '2025-08-05' (0 rows)
           p20250805: '2025-08-06' (0 rows)
           p20250806: '2025-08-07' (0 rows)
           pMAX: MAXVALUE (0 rows)
        
        🎉 파티션 초기 설정 완료! (4s)
        💡 이제 다음 명령어로 파티션을 관리할 수 있습니다:
           npm run manage-partitions-only
        🔌 DB 연결 종료
        ```
        
- 데이터 이관
    - copy_ad_effect_log.js
        
        ```jsx
        // scripts/copy_ad_effect_log.js
        // ad_effect_log에서 ad_effect_log_v2로 데이터 복사
        const mysql = require('mysql2/promise');
        const dayjs = require('dayjs');
        require('dotenv').config();
        
        (async () => {
          console.log('🚀 ad_effect_log 데이터 복사 시작...');
          const startTime = new Date();
        
          // 환경변수에서 데이터베이스 설정 가져오기
          const dbConfig = {
            host: process.env.DB_HOST || '127.0.0.1',
            port: parseInt(process.env.DB_PORT) || 3309,
            user: process.env.DB_USERNAME || 'root',
            password: process.env.DB_PASSWORD || 'toor',
            database: process.env.DB_DATABASE || 'addd-nks',
          };
        
          console.log('📋 Database Configuration:');
          console.log(`   - Host: ${dbConfig.host}:${dbConfig.port}`);
          console.log(`   - Database: ${dbConfig.database}`);
          console.log(`   - User: ${dbConfig.user}`);
        
          const conn = await mysql.createConnection(dbConfig);
        
          try {
            // 1. 소스 테이블 데이터 수 확인
            const [sourceCount] = await conn.query(`
              SELECT COUNT(*) as count FROM ad_effect_log
            `);
            
            const totalRecords = sourceCount[0].count;
            console.log(`📊 소스 테이블 데이터 수: ${totalRecords.toLocaleString()}개`);
        
            if (totalRecords === 0) {
              console.log('⚠️  복사할 데이터가 없습니다.');
              return;
            }
        
            // 2. 대상 테이블 기존 데이터 확인
            const [targetCount] = await conn.query(`
              SELECT COUNT(*) as count FROM ad_effect_log_v2
            `);
            
            if (targetCount[0].count > 0) {
              console.log(`⚠️  대상 테이블에 이미 ${targetCount[0].count.toLocaleString()}개 데이터가 있습니다.`);
              console.log('💡 기존 데이터를 삭제하고 복사하시겠습니까? (y/N)');
              return;
            }
        
            // 3. 배치 크기 설정
            const BATCH_SIZE = 1000;
            const totalBatches = Math.ceil(totalRecords / BATCH_SIZE);
            const progressInterval = Math.max(1, Math.floor(totalBatches / 10)); // 10%마다 진행상황 출력
        
            console.log(`📦 배치 크기: ${BATCH_SIZE.toLocaleString()}개`);
            console.log(`📦 총 배치 수: ${totalBatches.toLocaleString()}개`);
        
            let copiedRecords = 0;
        
            // 4. 배치 단위로 데이터 복사
            for (let offset = 0; offset < totalRecords; offset += BATCH_SIZE) {
              const batchNumber = Math.floor(offset / BATCH_SIZE) + 1;
              
              // 진행상황 로그
              if (batchNumber % progressInterval === 0 || batchNumber === totalBatches) {
                const progress = Math.round((batchNumber / totalBatches) * 100);
                const elapsed = Math.round((new Date() - startTime) / 1000);
                const remaining = Math.round((elapsed / batchNumber) * (totalBatches - batchNumber));
                console.log(`📈 Progress: ${progress}% (${batchNumber}/${totalBatches} batches) - ${elapsed}s elapsed, ~${remaining}s remaining`);
              }
        
              // 배치 데이터 조회
              const [sourceData] = await conn.query(`
                SELECT 
                  ad_id, ad_name, ad_start_time, ad_end_time, 
                  vehicle_number, route_code, date, exposure, 
                  views, attention, male_views, female_views, created_at
                FROM ad_effect_log 
                ORDER BY idx 
                LIMIT ? OFFSET ?
              `, [BATCH_SIZE, offset]);
        
              if (sourceData.length === 0) break;
        
              // 배치 데이터 삽입
              const insertPromises = sourceData.map(record => {
                return conn.query(`
                  INSERT INTO ad_effect_log_v2 
                    (ad_id, ad_name, ad_start_time, ad_end_time, vehicle_number, route_code, date, exposure, views, attention, male_views, female_views, created_at)
                  VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
                `, [
                  record.ad_id,
                  record.ad_name,
                  record.ad_start_time,
                  record.ad_end_time,
                  record.vehicle_number,
                  record.route_code,
                  record.date,
                  record.exposure,
                  record.views,
                  record.attention,
                  record.male_views,
                  record.female_views,
                  record.created_at
                ]);
              });
        
              await Promise.all(insertPromises);
              copiedRecords += sourceData.length;
            }
        
            // 5. 복사 결과 확인
            const [finalCount] = await conn.query(`
              SELECT COUNT(*) as count FROM ad_effect_log_v2
            `);
        
            const endTime = new Date();
            const totalTime = Math.round((endTime - startTime) / 1000);
            const avgTimePerRecord = (totalTime / copiedRecords * 1000).toFixed(2);
        
            console.log('\n✅ 데이터 복사 완료!');
            console.log(`📊 Summary:`);
            console.log(`   - 소스 데이터: ${totalRecords.toLocaleString()}개`);
            console.log(`   - 복사된 데이터: ${finalCount[0].count.toLocaleString()}개`);
            console.log(`   - 총 소요 시간: ${totalTime}s`);
            console.log(`   - 평균 처리 시간: ${avgTimePerRecord}ms/레코드`);
        
            if (totalRecords !== finalCount[0].count) {
              console.log(`⚠️  경고: 복사된 데이터 수가 일치하지 않습니다.`);
              console.log(`   - 소스: ${totalRecords.toLocaleString()}개`);
              console.log(`   - 대상: ${finalCount[0].count.toLocaleString()}개`);
            }
        
            console.log('\n💡 다음 단계:');
            console.log('   npm run setup-partitions (파티션 설정)');
        
          } catch (error) {
            console.error('❌ 데이터 복사 실패:', error.message);
            process.exit(1);
          } finally {
            await conn.end();
            console.log('🔌 DB 연결 종료');
          }
        })(); 
        ```
        
    - 실행 명령어
        
        ```bash
        node scripts/copy_ad_effect_log.js
        ```
        
    - 실행 결과
        
        ```jsx
        > bus_dashboard_server@0.0.1 copy-ad-effect-log
        > node scripts/copy_ad_effect_log.js
        
        [dotenv@17.2.1] injecting env (28) from .env -- tip: 🔐 prevent building .env in docker: https://dotenvx.com/prebuild
        🚀 ad_effect_log 데이터 복사 시작...
        📋 Database Configuration:
           - Host: 127.0.0.1:3309
           - Database: addd-nks
           - User: root
        📊 소스 테이블 데이터 수: 50,000개
        📦 배치 크기: 1,000개
        📦 총 배치 수: 50개
        📈 Progress: 10% (5/50 batches) - 15s elapsed, ~135s remaining
        📈 Progress: 20% (10/50 batches) - 32s elapsed, ~128s remaining
        📈 Progress: 30% (15/50 batches) - 49s elapsed, ~114s remaining
        📈 Progress: 40% (20/50 batches) - 66s elapsed, ~99s remaining
        📈 Progress: 50% (25/50 batches) - 83s elapsed, ~83s remaining
        📈 Progress: 60% (30/50 batches) - 100s elapsed, ~67s remaining
        📈 Progress: 70% (35/50 batches) - 118s elapsed, ~51s remaining
        📈 Progress: 80% (40/50 batches) - 135s elapsed, ~34s remaining
        📈 Progress: 90% (45/50 batches) - 152s elapsed, ~17s remaining
        📈 Progress: 100% (50/50 batches) - 169s elapsed, ~0s remaining
        
        ✅ 데이터 복사 완료!
        📊 Summary:
           - 소스 데이터: 50,000개
           - 복사된 데이터: 50,000개
           - 총 소요 시간: 172s
           - 평균 처리 시간: 3.44ms/레코드
        
        💡 다음 단계:
           npm run setup-partitions (파티션 설정)
        🔌 DB 연결 종료
        ```
        
- 파티셔닝 확인 DDL
    - DDL 확인 (파티션 정의가 실제로 반영됐는지)
        
        ```sql
        SHOW CREATE TABLE ${TABLE_NAME}\G
        ```
        
    - 파티션 메타정보 조회 (정보 스키마)
        
        ```sql
        SELECT 
          PARTITION_NAME,
          PARTITION_ORDINAL_POSITION,
          TABLE_ROWS,
          DATA_LENGTH,
          INDEX_LENGTH
        FROM INFORMATION_SCHEMA.PARTITIONS
        WHERE TABLE_SCHEMA = `${DB_NAME}`
          AND TABLE_NAME = `${TABLE_NAME}`
        ORDER BY PARTITION_ORDINAL_POSITION;
        ```
        
    - 특정 파티션의 데이터 직접 세기
        
        ```sql
        SELECT COUNT(*) FROM ${TABLE_NAME} PARTITION (${PARTITION_NAME});
        ```
        
    - 파티션 프루닝(선택) 확인 — Optimizer가 파티션을 잘 활용하는지
        
        ```sql
        EXPLAIN
        SELECT *
        FROM ${TABLE_NAME}
        WHERE created_at >= '2025-08-02' AND created_at < '2025-08-03';
        ```
        
    - 각 파티션의 경계값 검사 (범위가 예상과 일치하는지)
        
        ```sql
        SELECT 
          'p20250801' AS partition_name, 
          MIN(created_at) AS min_dt, 
          MAX(created_at) AS max_dt 
        FROM ad_effect_log_new PARTITION (p20250801)
        UNION ALL
        SELECT 
          'p20250802' AS partition_name, 
          MIN(created_at), 
          MAX(created_at) 
        FROM ad_effect_log_new PARTITION (p20250802);
        ```
        
    - 생성된 파티션 확인
        
        ```jsx
        mysql> SELECT 
            ->   PARTITION_NAME,
            ->   PARTITION_ORDINAL_POSITION,
            ->   TABLE_ROWS,
            ->   DATA_LENGTH,
            ->   INDEX_LENGTH
            -> FROM INFORMATION_SCHEMA.PARTITIONS
            -> WHERE TABLE_SCHEMA = 'addd-nks'
            ->   AND TABLE_NAME = 'ad_effect_log_v2'
            -> ORDER BY PARTITION_ORDINAL_POSITION;
        +----------------+----------------------------+------------+-------------+--------------+
        | PARTITION_NAME | PARTITION_ORDINAL_POSITION | TABLE_ROWS | DATA_LENGTH | INDEX_LENGTH |
        +----------------+----------------------------+------------+-------------+--------------+
        | p20250607      |                          1 |       1683 |      196608 |       409600 |
        | p20250608      |                          2 |        797 |      114688 |       163840 |
        | p20250609      |                          3 |        799 |      114688 |       163840 |
        | p20250610      |                          4 |        793 |       16384 |        98304 |
        | p20250611      |                          5 |        868 |      114688 |       196608 |
        | p20250612      |                          6 |        834 |       49152 |        98304 |
        | p20250613      |                          7 |        816 |      114688 |       131072 |
        | p20250614      |                          8 |        869 |       98304 |        98304 |
        | p20250615      |                          9 |        872 |      114688 |       196608 |
        | p20250616      |                         10 |        797 |       81920 |        98304 |
        | p20250617      |                         11 |        787 |       49152 |        98304 |
        | p20250618      |                         12 |        811 |       49152 |        98304 |
        | p20250619      |                         13 |        838 |       81920 |        98304 |
        | p20250620      |                         14 |        856 |      114688 |       196608 |
        | p20250621      |                         15 |        813 |       98304 |        98304 |
        | p20250622      |                         16 |        843 |      114688 |       196608 |
        | p20250623      |                         17 |        818 |      114688 |       163840 |
        | p20250624      |                         18 |        839 |      114688 |       196608 |
        | p20250625      |                         19 |        798 |      114688 |       163840 |
        | p20250626      |                         20 |        836 |      114688 |       196608 |
        | p20250627      |                         21 |        794 |      114688 |       163840 |
        | p20250628      |                         22 |        835 |      114688 |       196608 |
        | p20250629      |                         23 |        868 |      114688 |       196608 |
        | p20250630      |                         24 |        851 |       49152 |        98304 |
        | p20250701      |                         25 |        787 |      114688 |       131072 |
        | p20250702      |                         26 |        839 |      114688 |       196608 |
        | p20250703      |                         27 |        802 |       16384 |        98304 |
        | p20250704      |                         28 |        802 |      114688 |       163840 |
        | p20250705      |                         29 |        844 |       65536 |        98304 |
        | p20250706      |                         30 |        885 |       16384 |        98304 |
        | p20250707      |                         31 |        864 |      114688 |       196608 |
        | p20250708      |                         32 |        814 |      114688 |       163840 |
        | p20250709      |                         33 |        776 |      114688 |       131072 |
        | p20250710      |                         34 |        811 |       65536 |        98304 |
        | p20250711      |                         35 |        868 |       16384 |        98304 |
        | p20250712      |                         36 |        889 |       81920 |        98304 |
        | p20250713      |                         37 |        878 |      114688 |       196608 |
        | p20250714      |                         38 |        859 |       65536 |        98304 |
        | p20250715      |                         39 |        862 |       16384 |        98304 |
        | p20250716      |                         40 |        819 |       81920 |        98304 |
        | p20250717      |                         41 |        797 |       81920 |        98304 |
        | p20250718      |                         42 |        808 |      114688 |       163840 |
        | p20250719      |                         43 |        833 |       98304 |        98304 |
        | p20250720      |                         44 |        863 |      114688 |       196608 |
        | p20250721      |                         45 |        856 |      114688 |       196608 |
        | p20250722      |                         46 |        867 |      114688 |       163840 |
        | p20250723      |                         47 |        877 |       16384 |        98304 |
        | p20250724      |                         48 |        886 |      114688 |       163840 |
        | p20250725      |                         49 |        859 |       98304 |        98304 |
        | p20250726      |                         50 |        847 |      114688 |       196608 |
        | p20250727      |                         51 |        818 |      114688 |       163840 |
        | p20250728      |                         52 |        818 |       81920 |        98304 |
        | p20250729      |                         53 |        778 |      114688 |       131072 |
        | p20250730      |                         54 |        841 |       98304 |        98304 |
        | p20250731      |                         55 |        873 |       65536 |        98304 |
        | p20250801      |                         56 |        799 |      114688 |       163840 |
        | p20250802      |                         57 |        800 |      114688 |       131072 |
        | p20250803      |                         58 |        814 |       16384 |        98304 |
        | p20250804      |                         59 |        852 |      114688 |       196608 |
        | p20250805      |                         60 |          0 |       16384 |        98304 |
        | p20250806      |                         61 |          0 |       16384 |        98304 |
        | pMAX           |                         62 |          0 |       16384 |        98304 |
        +----------------+----------------------------+------------+-------------+--------------+
        62 rows in set (0.02 sec)
        ```
        
- 일별 자동 파티셔닝
    - manage_partitions_only.js
        
        ```jsx
        // scripts/manage_partitions_only.js
        // 파티션이 설정된 테이블의 일일 파티션 관리 스크립트
        const mysql = require('mysql2/promise');
        const dayjs = require('dayjs');
        const fs = require('fs').promises;
        const path = require('path');
        require('dotenv').config();
        
        // 파일 로그 기록 함수
        async function writeFileLog(logData) {
          try {
            const logDir = path.join(process.cwd(), 'logs');
            await fs.mkdir(logDir, { recursive: true });
            
            const logFile = path.join(logDir, `partition_management_${dayjs().format('YYYY-MM-DD')}.log`);
            const logEntry = `[${dayjs().format('YYYY-MM-DD HH:mm:ss')}] ${JSON.stringify(logData)}\n`;
            
            await fs.appendFile(logFile, logEntry);
          } catch (error) {
            console.error(`❌ 파일 로그 기록 실패:`, error.message);
          }
        }
        
        (async () => {
          const startTime = new Date();
          const jobName = 'partition_management_daily';
          
          console.log(`🚀 [${startTime.toISOString()}] 파티션 관리 시작...`);
        
          // 환경변수에서 데이터베이스 설정 가져오기
          const dbConfig = {
            host: process.env.DB_HOST || '127.0.0.1',
            port: parseInt(process.env.DB_PORT) || 3309,
            user: process.env.DB_USERNAME || 'root',
            password: process.env.DB_PASSWORD || 'toor',
            database: process.env.DB_DATABASE || 'addd-nks',
          };
        
          // 타겟 테이블 설정 (환경변수 또는 기본값)
          const TABLE_NAME = process.env.TARGET_TABLE || 'ad_effect_log_v2';
          const BACKUP_RETENTION_DAYS = parseInt(process.env.BACKUP_RETENTION_DAYS) || 60; // 기본 60일 전 데이터 백업 후 삭제
        
          console.log('📋 Database Configuration:');
          console.log(`   - Host: ${dbConfig.host}:${dbConfig.port}`);
          console.log(`   - Database: ${dbConfig.database}`);
          console.log(`   - User: ${dbConfig.user}`);
          console.log(`   - Target Table: ${TABLE_NAME}`);
          console.log(`   - Backup Retention: ${BACKUP_RETENTION_DAYS} days`);
        
          const conn = await mysql.createConnection(dbConfig);
          let jobStatus = 'RUNNING';
          let errorMessage = null;
          let partitionsCreated = 0;
          let partitionsBackedUp = 0;
          let partitionsDropped = 0;
          let recordsBackedUp = 0;
        
          try {
            // 1. 테이블 존재 확인
            const [tableExists] = await conn.query(`
              SELECT COUNT(*) as count 
              FROM INFORMATION_SCHEMA.TABLES 
              WHERE TABLE_SCHEMA = ? 
              AND TABLE_NAME = ?
            `, [dbConfig.database, TABLE_NAME]);
        
            if (tableExists[0].count === 0) {
              errorMessage = `테이블 ${TABLE_NAME}이 존재하지 않습니다.`;
              console.error(`❌ ${errorMessage}`);
              jobStatus = 'FAILED';
              return;
            }
        
            // 2. 파티션 설정 여부 확인
            const [partitions] = await conn.query(`
              SELECT 
                PARTITION_NAME,
                PARTITION_DESCRIPTION,
                TABLE_ROWS
              FROM INFORMATION_SCHEMA.PARTITIONS 
              WHERE TABLE_SCHEMA = ? 
              AND TABLE_NAME = ?
              ORDER BY PARTITION_ORDINAL_POSITION
            `, [dbConfig.database, TABLE_NAME]);
        
            if (partitions.length === 0) {
              errorMessage = `테이블 ${TABLE_NAME}에 파티션이 설정되어 있지 않습니다.`;
              console.log(`⚠️  ${errorMessage}`);
              console.log('💡 파티션 설정을 먼저 하세요: npm run setup-partitions');
              jobStatus = 'FAILED';
              return;
            }
        
            console.log(`✅ 테이블 ${TABLE_NAME}의 파티션 확인됨 (${partitions.length}개 파티션)`);
        
            const today = dayjs();
            const todayPartitionName = 'p' + today.format('YYYYMMDD');
            const backupDate = today.subtract(BACKUP_RETENTION_DAYS, 'day');
            const backupPartitionName = 'p' + backupDate.format('YYYYMMDD');
        
            // 3. 오늘 파티션 확인 및 생성
            const todayPartition = partitions.find(p => p.PARTITION_NAME === todayPartitionName);
            
            if (!todayPartition) {
              console.log(`📅 오늘 파티션 (${todayPartitionName}) 생성 중...`);
              
              const nextDate = today.add(1, 'day');
              const nextDateStr = nextDate.format('YYYY-MM-DD');
              
              try {
                await conn.query(`
                  ALTER TABLE \`${TABLE_NAME}\` 
                  ADD PARTITION (PARTITION ${todayPartitionName} VALUES LESS THAN ('${nextDateStr}'))
                `);
                console.log(`✅ 오늘 파티션 (${todayPartitionName}) 생성 완료`);
                partitionsCreated = 1;
              } catch (error) {
                errorMessage = `오늘 파티션 생성 실패: ${error.message}`;
                console.error(`❌ ${errorMessage}`);
                jobStatus = 'FAILED';
                return;
              }
            } else {
              console.log(`✅ 오늘 파티션 (${todayPartitionName}) 이미 존재함`);
            }
        
            // 4. 백업 대상 파티션 확인 및 처리
            const backupPartition = partitions.find(p => p.PARTITION_NAME === backupPartitionName);
            
            if (backupPartition && backupPartition.TABLE_ROWS > 0) {
              console.log(`📦 ${BACKUP_RETENTION_DAYS}일 전 파티션 (${backupPartitionName}) 백업 중...`);
              
              // 고정된 백업 테이블명 사용
              const backupTableName = `${TABLE_NAME}_backup`;
              
              try {
                // 백업 테이블이 이미 있는지 확인
                const [backupTableExists] = await conn.query(`
                  SELECT COUNT(*) as count 
                  FROM INFORMATION_SCHEMA.TABLES 
                  WHERE TABLE_SCHEMA = ? 
                  AND TABLE_NAME = ?
                `, [dbConfig.database, backupTableName]);
        
                if (backupTableExists[0].count === 0) {
                  // 백업 테이블 생성 (파티션 없이)
                  await conn.query(`
                    CREATE TABLE \`${backupTableName}\` LIKE \`${TABLE_NAME}\`
                  `);
                  
                  // 파티션 제거 (백업 테이블은 파티션 없이)
                  await conn.query(`
                    ALTER TABLE \`${backupTableName}\` REMOVE PARTITIONING
                  `);
                  
                  console.log(`✅ 백업 테이블 ${backupTableName} 생성 완료`);
                } else {
                  console.log(`✅ 기존 백업 테이블 ${backupTableName} 사용`);
                }
        
                // 데이터 복사
                const [copyResult] = await conn.query(`
                  INSERT INTO \`${backupTableName}\` 
                  SELECT * FROM \`${TABLE_NAME}\` 
                  PARTITION (${backupPartitionName})
                `);
                
                recordsBackedUp = copyResult.affectedRows;
                console.log(`✅ ${recordsBackedUp.toLocaleString()}개 레코드 백업 완료`);
                
                // 원본 파티션 삭제
                await conn.query(`
                  ALTER TABLE \`${TABLE_NAME}\` DROP PARTITION ${backupPartitionName}
                `);
                
                console.log(`✅ 파티션 ${backupPartitionName} 삭제 완료`);
                partitionsBackedUp = 1;
                partitionsDropped = 1;
                
              } catch (error) {
                errorMessage = `백업 처리 실패: ${error.message}`;
                console.error(`❌ ${errorMessage}`);
                jobStatus = 'FAILED';
                return;
              }
            } else if (backupPartition) {
              console.log(`📦 ${BACKUP_RETENTION_DAYS}일 전 파티션 (${backupPartitionName}) 데이터 없음 - 삭제`);
              
              try {
                await conn.query(`
                  ALTER TABLE \`${TABLE_NAME}\` DROP PARTITION ${backupPartitionName}
                `);
                console.log(`✅ 빈 파티션 ${backupPartitionName} 삭제 완료`);
                partitionsDropped = 1;
              } catch (error) {
                errorMessage = `파티션 삭제 실패: ${error.message}`;
                console.error(`❌ ${errorMessage}`);
                jobStatus = 'FAILED';
                return;
              }
            } else {
              console.log(`📦 ${BACKUP_RETENTION_DAYS}일 전 파티션 (${backupPartitionName}) 없음`);
            }
        
            // 5. 현재 파티션 상태 출력
            const [currentPartitions] = await conn.query(`
              SELECT 
                PARTITION_NAME,
                PARTITION_DESCRIPTION,
                TABLE_ROWS
              FROM INFORMATION_SCHEMA.PARTITIONS 
              WHERE TABLE_SCHEMA = ? 
              AND TABLE_NAME = ?
              ORDER BY PARTITION_ORDINAL_POSITION
            `, [dbConfig.database, TABLE_NAME]);
        
            console.log('\n📊 현재 파티션 상태:');
            currentPartitions.forEach(part => {
              const rows = part.TABLE_ROWS || 0;
              const isToday = part.PARTITION_NAME === todayPartitionName;
              const marker = isToday ? ' (오늘)' : '';
              console.log(`   ${part.PARTITION_NAME}: ${part.PARTITION_DESCRIPTION || 'MAXVALUE'} (${rows.toLocaleString()} rows)${marker}`);
            });
        
            jobStatus = 'SUCCESS';
        
          } catch (error) {
            errorMessage = `치명적 오류: ${error.message}`;
            console.error(`❌ ${errorMessage}`);
            jobStatus = 'FAILED';
          } finally {
            const endTime = new Date();
            const duration = Math.round((endTime - startTime) / 1000);
            
            // 파일 로그 기록
            await writeFileLog({
              jobName,
              targetTable: TABLE_NAME,
              status: jobStatus,
              duration,
              partitionsCreated,
              partitionsBackedUp,
              partitionsDropped,
              recordsBackedUp,
              errorMessage
            });
        
            if (jobStatus === 'SUCCESS') {
              console.log(`\n🎉 파티션 관리 완료! (${duration}s)`);
            } else {
              console.log(`\n❌ 파티션 관리 실패! (${duration}s)`);
              process.exit(1);
            }
        
            await conn.end();
            console.log('🔌 DB 연결 종료');
          }
        })(); 
        ```
        
    - 실행 명령어
        
        ```bash
        node scripts/manage_partitions_only.js
        ```
        
    - 실행 결과
        
        ```jsx
        > bus_dashboard_server@0.0.1 manage-partitions
        > node scripts/manage_partitions_only.js
        
        [dotenv@17.2.1] injecting env (28) from .env -- tip: 📡 version env with Radar: https://dotenvx.com/radar
        🚀 [2025-08-06T09:12:21.381Z] 파티션 관리 시작...
        📋 Database Configuration:
           - Host: 127.0.0.1:3309
           - Database: addd-nks
           - User: root
           - Target Table: ad_effect_log_v2
           - Backup Retention: 60 days
        ✅ 테이블 ad_effect_log_v2의 파티션 확인됨 (62개 파티션)
        ✅ 오늘 파티션 (p20250806) 이미 존재함
        📦 60일 전 파티션 (p20250607) 백업 중...
        ✅ 백업 테이블 ad_effect_log_v2_backup 생성 완료
        ✅ 1,683개 레코드 백업 완료
        ✅ 파티션 p20250607 삭제 완료
        
        📊 현재 파티션 상태:
           p20250608: '2025-06-09' (797 rows)
           p20250609: '2025-06-10' (799 rows)
           p20250610: '2025-06-11' (793 rows)
           p20250611: '2025-06-12' (868 rows)
           p20250612: '2025-06-13' (834 rows)
           p20250613: '2025-06-14' (746 rows)
           p20250614: '2025-06-15' (869 rows)
           p20250615: '2025-06-16' (872 rows)
           p20250616: '2025-06-17' (797 rows)
           p20250617: '2025-06-18' (787 rows)
           p20250618: '2025-06-19' (811 rows)
           p20250619: '2025-06-20' (838 rows)
           p20250620: '2025-06-21' (856 rows)
           p20250621: '2025-06-22' (813 rows)
           p20250622: '2025-06-23' (843 rows)
           p20250623: '2025-06-24' (818 rows)
           p20250624: '2025-06-25' (839 rows)
           p20250625: '2025-06-26' (798 rows)
           p20250626: '2025-06-27' (836 rows)
           p20250627: '2025-06-28' (794 rows)
           p20250628: '2025-06-29' (835 rows)
           p20250629: '2025-06-30' (868 rows)
           p20250630: '2025-07-01' (851 rows)
           p20250701: '2025-07-02' (787 rows)
           p20250702: '2025-07-03' (839 rows)
           p20250703: '2025-07-04' (802 rows)
           p20250704: '2025-07-05' (802 rows)
           p20250705: '2025-07-06' (844 rows)
           p20250706: '2025-07-07' (885 rows)
           p20250707: '2025-07-08' (864 rows)
           p20250708: '2025-07-09' (814 rows)
           p20250709: '2025-07-10' (776 rows)
           p20250710: '2025-07-11' (811 rows)
           p20250711: '2025-07-12' (868 rows)
           p20250712: '2025-07-13' (889 rows)
           p20250713: '2025-07-14' (878 rows)
           p20250714: '2025-07-15' (859 rows)
           p20250715: '2025-07-16' (862 rows)
           p20250716: '2025-07-17' (819 rows)
           p20250717: '2025-07-18' (797 rows)
           p20250718: '2025-07-19' (808 rows)
           p20250719: '2025-07-20' (833 rows)
           p20250720: '2025-07-21' (863 rows)
           p20250721: '2025-07-22' (856 rows)
           p20250722: '2025-07-23' (809 rows)
           p20250723: '2025-07-24' (877 rows)
           p20250724: '2025-07-25' (886 rows)
           p20250725: '2025-07-26' (859 rows)
           p20250726: '2025-07-27' (847 rows)
           p20250727: '2025-07-28' (818 rows)
           p20250728: '2025-07-29' (818 rows)
           p20250729: '2025-07-30' (778 rows)
           p20250730: '2025-07-31' (841 rows)
           p20250731: '2025-08-01' (873 rows)
           p20250801: '2025-08-02' (799 rows)
           p20250802: '2025-08-03' (749 rows)
           p20250803: '2025-08-04' (814 rows)
           p20250804: '2025-08-05' (852 rows)
           p20250805: '2025-08-06' (0 rows)
           p20250806: '2025-08-07' (0 rows) (오늘)
           pMAX: MAXVALUE (0 rows)
        
        🎉 파티션 관리 완료! (5s)
        🔌 DB 연결 종료
        ```
        

## 파티셔닝 이후 고려 해야할 점

### 1. 백업 테이블 조회 시 주요 리스크

1. 성능 관련 리스크
    - 데이터 크기
    - **인덱스 부재**: 백업 테이블은 파티셔닝이 없어서 전체 테이블 스캔 발생 가능
    - **메모리 부족**: 대용량 데이터 조회 시 MySQL 메모리 한계 초과
2. 시스템 안정성 리스크
    - 디스크 I/O 병목
        - **순차 읽기**: 백업 테이블은 파티셔닝 없이 단일 파일로 저장
        - **동시 조회**: 여러 사용자가 동시에 백업 테이블 조회 시 디스크 I/O 경합
    - 메모리 및 CPU 부하
        - **MySQL 버퍼 풀**: 대용량 데이터로 인한 메모리 부족
        - **연결 풀 고갈**: 장시간 실행되는 쿼리로 인한 연결 수 제한
3. 비지니스 리스크
    - 사용자 경험 저하
        - **응답 시간**: 백업 데이터 조회 시 수분~수십 분 대기
        - **타임아웃**: 웹 요청 타임 아웃으로 인한 사용자 불만
        - **서비스 중단**: 백업 테이블 조회로 인한 전체 시스템 성능 저하
    - 비용 증가
        - **스토리지 비용**: 백업 테이블 별도 저장으로 인한 추가 비용
        - **인프라 비용**: 백업 테이블 조회를 위한 추가 서버 리소스
        - **개발 비용**: 백업 테이블 조회 로직 구현 및 유지보수

### 2. 리스크 완화 방안

1. 백업 테이블 최적화
    - 인덱스 추가
    - 월별 파티셔닝 적용
2. 조회 전략 개선
    - 백업 데이터 조회를 비동기로 처리
    - 타임아웃 추가

### 3. 백업 테이블 성능 안정 보장 기간 예상 - 10초내 조회

1. **10초 제한 내 검색 가능 기간**
    
    
    | 최적화 수준 | 검색 가능 기간 | 데이터 크기 | 예상 조회 시간 | 안전도 |
    | --- | --- | --- | --- | --- |
    | **기본 인덱스** | 1-2개월 | 1.2억개 | 5-10초 | ⚠️ 위험 |
    | **최적화된 인덱스** | 2-3개월 | 1.8억개 | 8-12초 | ⚠️ 위험 |
    | **파티셔닝 적용** | 3-6개월 | 3.6억개 | 10-15초 | ⚠️ 위험 |
    | **고도 최적화** | 6개월 | 3.6억개 | 8-10초 | ✅ 안전 |
2. **실제 권장 검색 기간**
    - **1-2개월**: 매우 안전 (5초 이내)
    - **2-3개월**: 안전 (8초 이내)
    - **3-4개월**: 주의 필요 (10초 근처)
    - **4-6개월**: 위험 (10-15초)
    - **6개월 이상**: 매우 위험 (15초 이상)

3. MySQL vs MongoDB 비교

| 데이터베이스 | 최적화 수준 | 검색 가능 기간 | 데이터 크기 | 안전도 |
| --- | --- | --- | --- | --- |
| **MySQL** | 기본 인덱스 | 1-2개월 | 1.2억개 | ⚠️ 위험 |
| **MySQL** | 최적화된 인덱스 | 2-3개월 | 1.8억개 | ⚠️ 위험 |
| **MySQL** | 파티셔닝 | 3-4개월 | 2.4억개 | ⚠️ 위험 |
| **MongoDB** | 기본 인덱스 | 3-6개월 | 3.6억개 | ✅ 안전 |
| **MongoDB** | 시계열 컬렉션 | 6개월-1년 | 7.3억개 | ✅ 안전 |

# 파티셔닝 테이블 마이그레이션 자동화를 위한 조사

담당자: 강영선
상태: 닫힘
우선순위: 보통
서비스: DataBase/Server
ID: JOB-208
시작 예정일: 2025년 7월 31일
시작일: 2025년 7월 31일
작업 구분: backend
작업 유형: 환경구성
종료 예정일: 2025년 8월 5일
종료일: 2025년 8월 4일

# 상세정보

- 파티셔닝 대상 테이블(ad_effect_log)의 하루를 기준으로 들어오는 데이터 양은 400만개이고 600만개로 늘어날 수 있음
- 파티셔닝 계획은 일별 → 광고키, 차량번호로 나눌 예정
- 60~90 일 지난 데이터는 Legacy 테이블을 따로 생성해서 백업할 예정

# 성공 지표

- NKS 환경에서 파티셔닝 옵션을 지원하는 Tool 이나 Framework가 있다면 베스트

# 작업 계획

- [x]  NKS 환경에서 파티셔닝 옵션 지원이 되는 tool or framework 조사
- [x]  NKS 지원 서비스 중에 MySQL을 대체할 수 있는 DB가 있는지 조사
    - ~~파티셔닝 대상 테이블이 종속성이 없는 독립 테이블이기 때문에 NoSQL 기반 DB로 마이그레이션 했을 때 검색 속도가 더 용이할 것이라 생각됨~~
    - ~~데이터 검색 + 마이그레이션 자동화를 지원할 수 있는지를 중점으로 조사~~
    - 데이터의 송수신 영향은 쿼리 검색 속도 최적화 보다 연결 Pool의 문제를 해결하는게 더 근본적인 방법이기 때문에 패스

# 작업 결과

- [Atlas + TypeORM 연동 도입 제안 보고서](https://www.notion.so/Atlas-TypeORM-2429fb013f358016aa85f18c6678da8c?pvs=21)

# 관련 파일

[https://www.notion.so](https://www.notion.so)