# datetime — 시간 다루기

- datetime = 특정 시각 / timedelta = 시간의 길이 / timezone = 시간대
- 핵심 공식: **시각 + 길이 = 새로운 시각**
  `datetime.now(timezone.utc) + timedelta(minutes=30)`
- timedelta 단위: days, hours, minutes, seconds 등 키워드 인자
- 서버 내부는 항상 UTC로 계산, 사용자에게 보여줄 때만 현지 시간 변환
- `datetime.now()`처럼 인자 없이 쓰면 서버 지역 시간 → 시간대 버그의 원인