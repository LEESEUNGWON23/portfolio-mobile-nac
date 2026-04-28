# 모바일 엔드포인트 행동 분석 기반 보안 정책 엔진 — 포트폴리오

> 본 문서는 팀 프로젝트에서 **본인이 담당한 부분**을 중심으로 작성한 포트폴리오입니다.  
> 원본 레포지토리: [Mobile-Endpoint-Policy-Engine](https://github.com/LEESEUNGWON23/Mobile-Endpoint-Policy-Engine)

**유형:** 대학교 졸업 작품 (4인 팀 프로젝트)  
**기간:** 2024년  
**본인 역할:** 서버 사이드 전체 — Django 메인 서버, FastAPI ML 서버, Policy Engine  
**스택:** Python · Django · DRF · FastAPI · PyTorch · Scikit-learn · PostgreSQL

---

## 1. Project Overview

스마트폰의 터치·센서·네트워크 행동 패턴을 실시간으로 수집·분석해 이상 행위를 탐지하고, 자동으로 보안 정책(기기 잠금 / 네트워크 차단)을 실행하는 **제로 트러스트 기반 모바일 NAC(Network Access Control) 시스템**입니다.

ML 팀(Isolation Forest, LSTM Auto-Encoder 모델)과 Android/프론트엔드 팀으로부터 각각 모델과 요구사항을 전달받아, **서버 사이드 전체를 단독으로 설계·구현**했습니다.

---

## 2. 담당 역할 — 서버 사이드 전체

```
Android Agent
    │  행동 데이터 전송 (터치·센서·네트워크 로그)
    ▼
Django 메인 서버          ◀── 본인 구현
    │  데이터 수신 및 DB 저장
    │  ML 서버로 분석 요청
    ▼
FastAPI ML 서버           ◀── 본인 구현
    │  Isolation Forest + LSTM Auto-Encoder 하이브리드 이상 탐지
    │  탐지 결과 반환
    ▼
Django 메인 서버 (Policy Engine)  ◀── 본인 구현
    │  최종 위험도 산정 및 보안 정책 결정
    │
    ├─▶ Android Agent    ── 기기 잠금 / 네트워크 차단 명령 전달
    └─▶ React 대시보드   ── 실시간 위험도·로그 API 제공
```

---

## 3. Key Technical Challenge

### 3-1. 두 개의 이질적인 ML 모델을 어떻게 하나의 판단으로 합칠까

**Situation**  
ML 팀이 서로 다른 두 모델을 각각 개발해 전달했습니다.

- **Isolation Forest**: 개별 로그의 통계적 이상치 탐지 (정상 분포에서 얼마나 벗어났는가)
- **LSTM Auto-Encoder**: 시계열 데이터의 패턴 이상 탐지 (복원 오차가 임계값을 벗어났는가)

두 모델은 출력 형식도, 판단 기준도 달랐습니다. 이 둘을 어떻게 하나의 "이상 여부" 판단으로 합칠지를 직접 설계해야 했습니다.

**Action**  
보안 시스템의 특성상 **false negative(이상을 정상으로 판단)** 가 **false positive(정상을 이상으로 판단)** 보다 훨씬 위험하다고 판단했습니다. 따라서 둘 중 하나라도 이상을 감지하면 최종 이상으로 처리하는 **OR 기반 이중 검증 구조**를 설계했습니다.

```python
# /predict_hybrid 엔드포인트 — 하이브리드 융합 로직
is_iforest = bool(iforest_res.get("is_anomaly", False))
is_lstm    = bool(lstm_res.get("is_anomaly", False))

# 둘 중 하나라도 이상이면 최종 이상으로 판정 (Recall 최대화)
is_combined    = is_iforest or is_lstm
combined_score = (s_iforest + s_lstm) / 2.0
```

판단 근거가 어느 모델에서 나왔는지 추적할 수 있도록, 최종 결과에 각 모델의 개별 결과도 함께 반환했습니다.

```python
return {
    "is_anomaly_iforest":  is_iforest,
    "is_anomaly_lstm":     is_lstm,
    "is_anomaly_combined": is_combined,   # 최종 판정
    "anomaly_score_combined": combined_score,
}
```

**Result**  
OR 전략이 Recall을 높이는 방향으로 작동해 평균 Recall **0.9160**, 센서 데이터에서는 F1-Score **0.9668**을 달성했습니다.

---

### 3-2. API contract 없이 멀티팀 연동을 맡았을 때

**Situation**  
Android·ML·프론트엔드 팀이 각자 개발을 진행하고, 본인이 이 모든 컴포넌트를 연결하는 서버 사이드를 맡았습니다. 사전에 payload 구조와 API 스펙을 명확히 정의하지 않은 채로 개발이 시작되었습니다.

**Problem**  
두 가지 문제가 겹쳤습니다.

첫째, ML 팀의 모델이 완성되기 전까지 서버 사이드 개발이 블로킹되는 상황이었습니다. 병렬 개발을 위해 ML 팀의 반환 포맷을 임의로 가정한 **mock 구현체**를 먼저 만들어 Django↔FastAPI 연동을 미리 완성해뒀습니다. 그러나 실제 ML 코드를 전달받았을 때, 출력 구조가 mock과 달라 연동 로직을 거의 전부 재작성해야 했습니다.

둘째, ML·Android 팀이 개발 중 모델 출력 포맷이나 데이터 전송 구조를 수정할 때마다 Django 서버 전체에 변경이 연쇄적으로 퍼졌습니다. 연동 지점이 3개(Android↔Django, Django↔FastAPI, Django↔Dashboard)였기 때문에, 한 쪽이 바뀌면 나머지 레이어도 따라서 수정해야 했습니다.

**Action**  
변경이 발생할 때마다 해당 연동 레이어를 빠르게 재구현하며 전체 흐름을 유지했습니다. 또한 각 컴포넌트 간 경계를 모듈 단위로 명확히 분리해, 한 쪽 변경이 다른 레이어로 최소한만 전파되도록 구조를 개선했습니다.

**Result & 회고**  
기능 구현은 완료했지만, mock 기반 선행 개발이 오히려 재작업을 늘린 경험을 통해 두 가지 교훈을 얻었습니다.

- **API contract 우선**: mock을 만들기 전에 실제 출력 포맷을 팀 간에 먼저 합의해야 한다. contract 없는 mock은 작업을 두 번 하는 것과 같다.  
- **진척도 공유의 중요성**: 각 팀이 중간 결과물을 지속적으로 공유했다면 mock과 실물의 괴리를 초기에 좁힐 수 있었다.

이 경험이 이후 업무에서 인터페이스 설계와 팀 간 스펙 합의를 개발보다 앞세우는 습관의 출발점이 되었습니다.

---

## 4. Performance

| 지표 | 수치 |
|------|------|
| 평균 F1-Score | **0.8961** |
| 평균 Recall | **0.9160** |
| 센서 데이터 F1-Score | **0.9668** |

---

## 5. Tech Stack

| 기술 | 역할 |
|------|------|
| **Django + DRF** | 메인 서버, REST API, DB 관리 |
| **FastAPI** | ML 모델 serving 서버 |
| **Scikit-learn** | Isolation Forest 이상 탐지 |
| **PyTorch** | LSTM Auto-Encoder 이상 탐지 |
| **PostgreSQL** | 행동 로그 및 탐지 결과 저장 |
| Android / React | 행동 데이터 수집 agent, 관리자 대시보드 (타팀 구현) |
