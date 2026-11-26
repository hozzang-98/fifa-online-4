# FIFA-Online-4

넥슨 FIFA 온라인 4 / FC 온라인 공개 API로 유저 경기 데이터를 모아 분석한 프로젝트. 1차는 슈팅 기록으로 골 여부를 예측하는 모델과 웹 이벤트, 2차(FCO)는 선수 카드별 경기 스탯 집계.

## 데이터

| 출처 | 용도 |
| --- | --- |
| `fifaonline4.nexon.com/datacenter/rank` (HTML 크롤링) | 랭킹 500페이지 × 20명 = 닉네임 10,000개 |
| `api.nexon.co.kr/fifaonline4/v1.0/users` | 닉네임 → accessId |
| `.../users/{accessId}/matches?matchtype=50` | 유저당 공식경기 최근 5경기 matchId (49,945건) |
| `.../matches/{matchId}` | 경기별 `shootDetail` → 슈팅 기록 617,169건 |
| `open.api.nexon.com/fconline/v1` (2차) | FC 온라인 ouid, 매치, 매치 상세, 선수/시즌 메타데이터 |

API 키는 넥슨 개발자센터에서 발급받아 `Authorization` 헤더(1차) / `x-nxopen-api-key` 헤더(2차)로 보낸다.

## 1차: 슈팅 이벤트 모델

| 파일 | 역할 |
| --- | --- |
| `1. user_crawling.ipynb` | 랭킹 페이지 크롤링 → 닉네임 → accessId (`user_info.csv`) |
| `2. match_id.ipynb` | accessId → matchId → 슈팅 상세 15개 컬럼 추출 (`shoot_info.csv`) |
| `3. modeling.ipynb` | 전처리, XGBoost 학습, 입력값 → 골 여부 예측 함수 |
| `HadoopSpark_FIFA.ipynb` | 같은 데이터를 PySpark 로 EDA, 슈팅 타입 + 능력치 합으로 LogisticRegression |
| `event.py` | Flask 서버. `/shoot` 입력 폼 → `/formproc` 예측 결과 |
| `shoot_info.html`, `result.html` | Flask 템플릿 (좌표, 슈팅 타입, 포지션, 능력치 12개 입력 / 결과 표시) |

모델링 입력은 슈팅 기록에 선수 능력치(골 결정력, 슛 파워, 중거리 슛, 위치 선정, 발리 슛, 페널티 킥, 프리킥, 커브, 볼 컨트롤, 밸런스, 헤더, 점프, 침착성)를 붙인 `final_df.csv`(32컬럼)이다. 능력치 조인 단계는 노트북에 포함돼 있지 않다.

전처리
- 파생 변수: `점핑 헤더 = (헤더 + 점프) / 2`, `distance = sqrt((x-1)^2 + (y-0.5)^2)` (골대 기준 거리)
- 타깃: `result` 3(골) → 1, 1·2(유효슛·빗나감) → 0 이진분류
- `type`(슈팅 타입 1~10), `포지션` 원-핫, 능력치 12개 MinMaxScaler
- 7:3 분할 후 테스트를 다시 7:3 으로 나눠 검증셋 생성, 학습셋은 RandomUnderSampler 로 클래스당 98,920건

모델: `XGBClassifier(n_estimators=100, learning_rate=0.03, max_depth=10, subsample=0.7, ...)`

결과 (테스트셋)

| 지표 | 값 |
| --- | --- |
| Accuracy | 0.642 |
| Precision (macro) | 0.598 |
| Recall (macro) | 0.635 |

PySpark 쪽 LogisticRegression(피처: 슈팅 타입, 능력치 합) 은 accuracy 0.656.

## 2차: FCO 카드 스탯

FC 온라인(구 FIFA 온라인 4) 으로 바뀐 뒤 새 Open API 로 랭커 160명의 최근 공식경기를 모아, 선수 카드(시즌 + 선수) 별 출전 수와 평균 평점을 집계했다. 상세는 [`fco/`](fco/) 폴더 README 참고.

- 수집: Selenium 랭킹 크롤링 → ouid → matchId → match-detail 의 선수별 `status`
- 전처리: spid/seasonid 메타데이터로 카드명·시즌명 부여, 평점 0(미출전) 제외 → 16,621건
- 결과: 시즌 카드 1,982종, 출전 10회 이상 중 평균 평점 상위 10 시각화
- Airflow DAG(`@monthly`) 로 매치 수집 자동화 시도

## 실행 방법

```bash
pip install pandas numpy requests beautifulsoup4 tqdm scikit-learn imbalanced-learn xgboost flask
```

1. 노트북 상단에 넥슨 API 키를 `api_key` 변수로 넣고 `1 → 2 → 3` 순서로 실행한다. 3번은 `train_scaler.pkl`, `xgb_model.model` 을 저장한다.
2. `shoot_info.html`, `result.html` 을 `templates/` 아래로 옮긴 뒤 서버를 띄운다.

```bash
python event.py   # http://localhost:4000/shoot
```

학습 데이터와 모델 파일은 저장소에 포함돼 있지 않다.
