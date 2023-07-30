# Spring 개념 학습

## Test

### `DirtiesContext`

**Java DOC**
- 테스트에 `DirtiesContext` 를 표시하면 테스트에 사용되는 `ApplicationContext` 이 오염되었다고 (dirty) 간주하기 때문에 `ApplicationContext` 를 종료하고 컨텍스트 캐시에서 제거한다.
- 따라서 테스트가 컨텍스트를 변경할 때 `DirtiesContext` 를 사용해야 한다.
  - 컨텍스트에서 공유되는 싱글톤 빈의 상태를 변경했을 때 
  - 데이터베이스 상태를 변경했을 
- 컨텍스트가 종료되면 동일한 컨텍스트를 요청하는 연이은 테스트들은 새로운 `ApplicationContext` 를 할당받는다.

**사용 배경**
- `지하철 노선도` 미션의 테스트 뼈대 코드에서 `DirtiesContext` 를 사용하고 있다.
```java
@DirtiesContext(classMode = DirtiesContext.ClassMode.BEFORE_EACH_TEST_METHOD)
```
- 해당 설정은 모든 테스트 메서드를 실행할 때마다 새로운 `ApplicatioContext` 를 할당한다.
- 모든 테스트 메서드를 격리하기 위해 사용했다고 한다.
  - 인메모리 데이터베이스를 사용하고 있으므로 `ApplicationContext` 가 종료되고 생성될 때마다 데이터베이스도 새로 생성

**문제점**
- 통합 테스트의 각 메서드를 실행할 때마다 `ApplicationContext` 를 종료하고 새로 생성하므로 시간이 오래 걸린다.

<img width="331" src="https://private-user-images.githubusercontent.com/62409503/257034098-a38002ae-6ec2-4cd8-9590-a0e4c619e837.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTEiLCJleHAiOjE2OTA2OTU3MjQsIm5iZiI6MTY5MDY5NTQyNCwicGF0aCI6Ii82MjQwOTUwMy8yNTcwMzQwOTgtYTM4MDAyYWUtNmVjMi00Y2Q4LTk1OTAtYTBlNGM2MTllODM3LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFJV05KWUFYNENTVkVINTNBJTJGMjAyMzA3MzAlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjMwNzMwVDA1MzcwNFomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTQzMGVjMDA3NDYxNjgyYWI1N2MxODNkZDExZDM3YTQ0YTdjMzI5OTYzMjAyODA0YThkYjU5N2JjMDk3MzIzNWMmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.yeB9WTPs_4Wajbv19m4d0hHrmfrtdCw4PeKSjBLCoT8">

- 24개의 통합 테스트 수행에 31초의 시간이 소요되었다.

**해결 방법**
- 테스트는 빨라야 한다. 따라서 `DirtiesContext` 를 사용하지 않는다.
- `@Transaction` 어노테이션을 설정하여 테스트 메서드 호출 후 데이터베이스 롤백
  - 데이터베이스 롤백만을 위해 `@Transaction` 을 사용하는 것은 적절하지 않다고 생각
  - 롤백 외의 다른 기능을 제공하기 때문에 의도치 않은 부작용이 발생할 수 있고, 어노테이션만 보고 롤백을 한다고 유추하기도 어려움
- `@BeforeEach` 설정으로 각 테스트 메서드 호출 후 데이터베이스 초기화 쿼리 보내기
  - 어떤 작업을 수행하는지 의미가 명확함
  - 오직 테스트 데이터베이스 초기화를 위해 `dao` 에 `deleteAll` 를 구현하는 것은 잘못 사용될 위험이 있음
  - 데이터베이스 실행 환경을 테스트와 운영으로 분리하고, 테스트에서는 `jdbcTemplate` 을 주입받아 데이터베이스 초기화 ! 

**적용 결과**
- 24 개의 통합 테스트 수행 시간이 31초에서 4초로 감소하였다.

<img width="331" src="https://private-user-images.githubusercontent.com/62409503/257038335-7c205e90-18d0-45a3-86d8-a07f7bfc8b60.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTEiLCJleHAiOjE2OTA3MDI1MzgsIm5iZiI6MTY5MDcwMjIzOCwicGF0aCI6Ii82MjQwOTUwMy8yNTcwMzgzMzUtN2MyMDVlOTAtMThkMC00NWEzLTg2ZDgtYTA3ZjdiZmM4YjYwLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFJV05KWUFYNENTVkVINTNBJTJGMjAyMzA3MzAlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjMwNzMwVDA3MzAzOFomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWViMDkxYmE0MzU4Yzc1YzRmZjFlM2U0MmFmMzMwYmRlZjdlNGM5OWE0YTM5OTEyNmIxMmI1YWM2OThiZGFkOWQmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.soOyCDy-eDqmNlba62g1tHSa0U0fQrPXyZl_8NO8BxA">
