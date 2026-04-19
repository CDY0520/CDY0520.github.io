---
title: "팀 프로젝트1: First Isolation 규칙을 적용하여 시계열 예측을 위한 데이터 정제"
date: 2026-02-17 13:10:00 +0900
categories: [New_Run, LOG]
tags: [LOG]
---

<hr style="height: 5px; background-color: #777; border: none;">

## 개요
- 목표: 시계열 예측 모델(Prophet) 학습을 위한 고순도 데이터셋 구축  
- 핵심: 'First Isolation(첫 균주 분리)' 규칙을 SQL로 구현하여 중복 신호를 제거하고 데이터 신뢰도 확보  
<br>

>데이터의 양보다 중요한 것은 데이터의 '질'이다.  
>특히 시계열 모델에서 중복된 검사 결과는 트렌드를 왜곡하는 치명적인 노이즈가 된다.  

<hr style="height: 5px; background-color: #777; border: none;">

## 시계열 모델의 적: 중복 데이터
이전 단계에서 생성한 FOR_PREDICT 테이블에는 환자가 검사를 받을 때마다 생성된 모든 결과가 들어있다.  
하지만 이 데이터를 그대로 모델에 넣으면 안 된다.  
나중에 외부 데이터도 활용할 것이기 때문에 외부데이터 집계 방식이니 첫균주 분리 규칙이 적용되어야 하기 때문이다.  
<br>
동일 환자가 한 달 내에 여러 번 검사를 받아 모두 내성 판정이 나왔을 때,  
이를 개별 사건으로 처리하면 내성 발생률이 과다 집계(Overestimation) 된다.  
<br>
때문에 의료 표준 분석 원칙인 'First Isolation' 규칙을 적용해야 한다.  
이는 특정 기간(연도별) 내에 환자 한 명당, 검체별로 가장 처음 발견된 내성 결과만 유효한 시그널로 인정하는 것을 말한다.  

<hr style="height: 5px; background-color: #777; border: none;">

## First Isolation 로직의 SQL 구현: CRE_PRED_INPUT 테이블 설계
시계열 모델이 '진짜 추세'를 학습할 수 있도록, 환자별 최초 발생일만 남기는 파이프라인을 구축했다.  
<br>
<br>

```
-- 1. 환자별/연도별/검체별 최초 발생일(MIN) 추출
WITH first_occurrence AS (
  SELECT
    patient_no,
    spec_type,
    EXTRACT(YEAR FROM test_date) AS year,
    MIN(test_date) AS first_test_date
  FROM FOR_PREDICT
  WHERE "CIMP(R)" = 1 OR "CMEM(R)" = 1 OR "CETP(R)" = 1 -- 타겟 필터링
  GROUP BY patient_no, spec_type, year
),
```

위 코드처럼 환자별, 검체별, 연도별 그룹으로 묶어서 최초 발생일을 추출한다.  
<br>
<br>

```
-- 2. 원본 데이터와 JOIN하여 최초 기록만 남기기
final_filtered AS (
  SELECT w.*
  FROM FOR_PREDICT w
  JOIN first_occurrence f
    ON w.patient_no = f.patient_no
   AND w.test_date = f.first_test_date
),
```

이제 이렇게 걸러진 데이터만 모아서 카운트 한다.  
<br>
<br>

```
-- 3. 모델 인풋용 월별 집계
SELECT
  TO_CHAR(test_date, 'YYYY-MM') AS year_month,
  COUNT(*) AS CRE_COUNT
FROM final_filtered
GROUP BY year_month
ORDER BY year_month;
```

첫 균주 분리는 연도별로 했지만, 시계열 모델에는 시간을 월별로 집계할 것이기 때문에  
다시 월별로 나누고 내부 시계열 데이터로 최종 확정한다.
<br>
<br>

![](/assets/img/log_2-1.PNG)

그러면 이제 위쪽 사진처럼 병원 내부 시계열 데이터가 구성되었다.
<br>
<br>

![](/assets/img/log_2-2.PNG)

전국, 지역 단위 데이터도 모델 학습에 활용할 것이기 때문에  
최종적으로 위 사진처럼 모델 인풋용 데이터 CRE_PRED_INPUT 테이블이 설계되었다.

<hr style="height: 5px; background-color: #777; border: none;">

## Note: 데이터의 한계와 실무적 선택
데이터 중복 제거는 흔한 작업이지만, 의료 시계열 데이터에서는 '어떤 것을 남길 것인가'에 대한 의학적 근거가 필수적이다.  
'First Isolation' 규칙을 SQL로 자동화함으로써,  
분석가가 수작업으로 데이터를 정제할 때 발생할 수 있는 휴먼 에러를 방지하고 분석의 재현성을 확보했다.  
<br>
시계열 모델은 데이터의 연속성이 중요하다.  
이번 쿼리를 통해 환자 단위의 개별 기록을 월별 발생 건수라는 고정된 타임스탬프로 변환했다.  
<br>
집계 단위를 어떻게 설정할지는 모델의 성능을 결정짓는 핵심 과제였다.  
고민 끝에 첫 균주 분리에는 연도별 집계를 활용했지만, 시계열 데이터는 월별 집계를 선택했다.  
<br>
현재 확보된 내부 데이터는 약 2~3년 치로,  
연도별로 집계할 경우 학습 샘플이 단 2~3개에 불과해 모델이 유의미한 패턴을 학습하기에 턱없이 부족했기 때문이다.  
이를 월별 단위로 세분화하여 데이터 포인트(Data Points)를 늘림으로써,  
한정된 자원 안에서 시계열 모델이 계절성(Seasonality)과 장기적인 추세를 학습할 수 있는 최소한의 환경을 구축했다.  
<br>
또한 의료 현장에서 내성균 관리는 연 단위의 거시적 변화보다 월 단위의 미시적 변화에 대응하는 것이 훨씬 중요하다.  
결론적으로 월별 추이를 예측 데이터로 제공하는 것이 감염 관리 시스템이 적시에 대응 전략을 세우는 데 더 적합한 형태라고 판단했다.  
<br>
이렇게 구축된 CRE_PRED_INPUT 테이블은 모델 학습뿐만 아니라, 병원 내 감염 관리실에서 사용하는 원천 데이터로도 즉시 활용될 수 있다.  
하나의 잘 설계된 파이프라인이 연구와 실무를 동시에 지원하는 사례를 구축했다.  
