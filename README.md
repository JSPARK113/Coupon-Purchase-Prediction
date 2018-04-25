# Coupon Purchase Prediction

## 1. 프로젝트 개요

1. 내용: 고객의 패턴(방문, 구매 내역 등)을 통해 해당 고객이 어떤 쿠폰을 구매할지 예측
    - https://ponpare.jp/ 사이트의 데이터
2. 평가 기준: the Mean Average Precision @ 10 (MAP@10)
3. 데이터: 8개 테이블로 구성
    - coupon_area_test: 테스트용 쿠폰의 지역 정보
    - coupon_area_train: train용 쿠폰의 지역 정보
    - coupon_detail_train: 쿠폰 구매 내역
    - coupon_list_test: 테스트용 쿠폰 리스트
    - coupon_list_train: train용 쿠폰 리스트
    - coupon_visit_train: 쿠폰 조회 내역
    - user_list: 고객 리스트
    - prefecture_location: 지역 정보
4. 모델링 방식
    - Cosine Similarity (최종 채택)
    - 그레디언트 부스팅 (XGBoost 라이브러리 사용)
5. 최종 성적: 0.006201, 1076명 중 76위 (상위 7%)


## 2. 전처리
1. 일본어 포함된 데이터는 영어로 변환.
2. 쿠폰 구매 내역, 쿠폰 조회 내역, 쿠폰 리스트, 고객 리스트 Merge:

    1. detail_train : 쿠폰 구매 내역: 168,996rows * 6columns
        - I_DATE: 구매한 날짜
        - ITEM_COUNT: 구매한 제품 갯수
        - COUPON_ID_hash: COUPON number
        - PURCHASEID_hash: 구매 번호
        - SMALL_AREA_NAME: 고객 거주지역(소단위)
        - USER_ID_hash: USER ID
    2. visit_train: 쿠폰 조회 내역: 2,833,180rows * 8 columns
        - I_DATE: 홈페이지 방문 날짜
        - PAGE_SERIAL: page number
        - PURCHASE_FLG: 구매 여부(0: 구매 안함, 1: 구매)
        - REFERRER_hash, SESSION_ID_hash: 접속 로그와 비슷한 개념
        - USER_ID_hash: USER ID
        - VIEW_COUPON_ID_hash: 고객이 조회한 COUPON ID
    3. Coupon_list, 쿠폰 리스트: 19,413rows * 24 columns
        - DISPFROM, DISPEND, DISPPERIOD: 상품 release 기간
        - VALIDFROM, VALIDEND, VALIDPERIOD: 상품 유효 기간
        - CATALOG_PRICE: 소비자가(정가)
        - DISCOUNT_PRICE: 할인가격
        - PRICE_RATE: 할인율
        - USABLE_DATE_MON~FRI, HOLIDAY: 해당 요일 사용 가능 여부
        - CAPSULE_TEXT: 상품 소분류
        - GENRE_NAME: 상품 중분류
        - LARGE_AREA_NAME: 쿠폰 사용 가능 지역(대분류)
        - ken_name: 쿠폰 사용 가능 지역(중분류, Prefecture)
        - SMALL_AREA_NAME: 쿠폰 사용 가능 지역(소분류)
    4. User_list
        - REG_DATE: 가입한 날짜
        - WITHDRAW_DATE: 탈퇴한 날짜
        - AGE: 연령
        - PREF_NAME: 거주 지역
        - SEX_ID: 성별
        - USER_ID_hash: USER ID

## 3. EDA & Feature selection

1. 세션 아이디 등 분석에 필요없는 데이터 삭제
2. PURCHASE_FLG를 y 값으로 분포 파악
    - PURCHASE_FLG 값이 1이면 구매, 0이면 비구매
    - PURCHASE_FLG 값이 1인 데이터는 0인 데이터에 비해 어떤 형태를 보일 것인가?

3. numeric values
    - PRICE_RATE: 50% 수준이 대다수 분포
    - CATALOG_PRICE / DISCOUNT_PRICE / Price는 분포 및 상관관계가 유사 -> Price 유지하며, 정규화 처리
    - DISPPERIOD: 기간이 긴 것들이 일부 존재(평균을 10이라고 했을때, 4만(구매는 9,300)) 하지만 OUTLIER라고 활 명확한 근거가 없음
    - VALIDPERIOD: 유효기간의 제한이 없는 것들이 대다수를 차지함
4. categorical data
    - Case: DELIVERY, FOOD가 대다수를 차지 -> 연령 및 성별 등과 복합적으로 봐야할 것으로 보임
    - REGION: 9개의 large_area에서 55개의 small area로 나뉨, 대다수가 Tokyo, Osaka등 대도시임
    - 30대~50대가 대부분의 소비를 차지하고 있으며, 이들이 delivery와 food를 주로 구매하고 있음
    - 전체로 봤을 때는 여성의 소비 비중이 높음
    - 여성과 남성의 구입 품목에서도 차이를 보이는데, 여성의 경우 delivery / 남성의 경우 Food 소비가 주를 이룸
    - HOTLE과 LEISURE는 도쿄 중심이 아니라 다양한 지역으로 분포 되어 있음, 여행목적이기 때문인 것으로 보임
    - 남성의 비중이 크고, 연령대도 높음
