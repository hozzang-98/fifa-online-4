# FCO Card Stats

FC 온라인 Open API 로 랭커들의 최근 공식경기를 모아 선수 카드(시즌 + 선수) 별 출전 수와 평균 평점을 집계한 프로젝트. FIFA-Online-4 슈팅 모델의 후속 작업.

정리 문서: https://www.notion.so/FCO-Card-Stats-173c71a8a29e807ab788dc52e587bfe5?pvs=4

## 파일

| 파일 | 역할 | 산출물 |
| --- | --- | --- |
| `server.py` | API 키, 베이스 URL, 엔드포인트 상수 (`USER_FUNCTION`, `MATHCH_FUNCTION`, `METADATA_FUNCTION` 등). 다른 파일에서 `from server import *` | - |
| `user.ipynb` | Selenium 으로 `fconline.nexon.com/datacenter/rank` 크롤링 → 닉네임 → `/fconline/v1/id` 로 ouid 조회 | `data/user_ouid.json` (160명) |
| `matchid.ipynb` | ouid 별 공식경기(matchtype 50) 최근 5경기 matchId | `data/matchid.json` |
| `matchdetail.ipynb` | `/fconline/v1/match-detail` 호출(재시도 3회, 3초 간격), 선수별 `status` 를 평탄화 | `data/matchdetail.csv` |
| `preprocessing.ipynb` | spid/seasonid 메타데이터로 카드명·시즌명 부여, 평점 0(미출전) 제거 (26,938 → 16,621건) | `data/eda.csv` |
| `eda.ipynb` | 시즌 카드 1,982종 집계, 출전 10회 이상 중 평균 평점 상위 10 이중 막대 그래프 | `best.csv` (주석 처리) |
| `scheduler.ipynb` | `matchid.ipynb` 로직을 Airflow DAG(`web_crawling_dag`, `@monthly`) 로 등록 | - |

## 실행

`server.py` 의 `my_api_key` 를 넥슨 개발자센터에서 발급받은 키로 바꾼 뒤 `user → matchid → matchdetail → preprocessing → eda` 순서로 실행한다. `data/` 폴더는 저장소에 없으므로 직접 만든다.
