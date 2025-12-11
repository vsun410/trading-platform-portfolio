# 📊 Portfolio 세부 기획서

**Repository:** trading-platform-portfolio  
**Version:** 1.0  
**Date:** 2025-12-11

---

## 1. 개요

### 1.1 목적

전략 간 상관관계를 분석하고, 포트폴리오 최적화를 통해 위험 대비 수익을 극대화합니다. 리밸런싱은 **수동**으로 진행하며, 의사결정을 위한 분석 도구를 제공합니다.

### 1.2 핵심 책임

- **상관관계 분석:** 전략 간 상관계수 계산, 다각화 효과 검증
- **리스크 지표:** Sharpe Ratio, MDD, VaR, CVaR 계산
- **자본 배분:** Kelly Criterion, Mean-Variance, Risk Parity
- **리밸런싱 권고:** 최적 비중 제안 (수동 실행)

### 1.3 연관 레포지토리

| 레포 | 관계 | 데이터 흐름 |
|------|------|-------------|
| research | 백테스트 결과 제공 | research → 성과 데이터 → portfolio |
| storage | 거래 이력 제공 | storage → 체결/포지션 → portfolio |
| order | (간접) 실행 결과 | order → storage → portfolio |

---

## 2. 분석 기능 명세

### 2.1 리스크 지표

| 지표 | 공식 | 용도 |
|------|------|------|
| Sharpe Ratio | (Rp - Rf) / σp | 위험 조정 수익률 |
| Sortino Ratio | (Rp - Rf) / σd | 하방 위험만 고려 |
| MDD | max(Peak - Trough) / Peak | 최대 손실 위험 |
| VaR (95%) | Percentile(5%, returns) | 일별 예상 최대 손실 |
| CVaR (ES) | E[X \| X ≤ VaR] | 극단 손실 평균 |
| Calmar Ratio | CAGR / MDD | 수익 대비 낙폭 |

### 2.2 상관관계 분석

#### 2.2.1 상관계수 계산

```
ρ(X, Y) = Cov(X, Y) / (σx × σy)
```

전략 간 수익률 상관계수를 계산하여 다각화 효과를 측정합니다.

#### 2.2.2 해석 기준

| 상관계수 | 해석 | 포트폴리오 영향 |
|----------|------|-----------------|
| 0.7 ~ 1.0 | 강한 양의 상관 | 다각화 효과 낮음 |
| 0.3 ~ 0.7 | 중간 양의 상관 | 일부 다각화 효과 |
| -0.3 ~ 0.3 | 낮은 상관 | **우수한 다각화** ✓ |
| -1.0 ~ -0.3 | 음의 상관 | **헤지 효과** ✓ |

---

## 3. 자본 배분 최적화

### 3.1 Kelly Criterion

```
f* = (p × b - q) / b

  p = 승률
  q = 패률 (1 - p)
  b = 손익비 (평균 이익 / 평균 손실)
```

**주의:** Full Kelly는 변동성이 크므로, 실제 적용 시 **Half Kelly (f* / 2)** 권장

### 3.2 Mean-Variance (Markowitz)

```
목적함수: max(w'μ - λ/2 × w'Σw)

  w = 자산별 비중 벡터
  μ = 기대 수익률 벡터
  Σ = 공분산 행렬
  λ = 위험 회피 계수
```

### 3.3 Risk Parity

```
목표: 각 자산의 위험 기여도 동일화
RC_i = w_i × (Σw)_i / σp = 1/n
```

모든 전략이 포트폴리오 위험에 동등하게 기여하도록 비중 조절

### 3.4 최적화 방법 비교

| 방법 | 장점 | 단점 |
|------|------|------|
| Kelly | 복리 성장 최대화 | 높은 변동성, 과추정 위험 |
| Mean-Variance | 이론적 최적, Efficient Frontier | 추정 오차에 민감 |
| Risk Parity | 안정적, 집중 리스크 방지 | 수익률 정보 미사용 |

---

## 4. 디렉토리 구조

```
trading-platform-portfolio/
├── README.md
├── pyproject.toml
│
├── docs/
│   ├── README.md
│   ├── ANALYSIS_METHODS.md     # 분석 방법론
│   ├── OPTIMIZATION_GUIDE.md   # 최적화 가이드
│   └── DETAILED_SPEC.md        # 세부 기획서 (이 문서)
│
├── src/
│   ├── analyzer/               # 분석 모듈
│   │   ├── __init__.py
│   │   ├── correlation.py      # 상관관계 분석
│   │   ├── risk.py             # 리스크 지표
│   │   └── performance.py      # 성과 분석
│   │
│   └── optimizer/              # 최적화 모듈
│       ├── __init__.py
│       ├── kelly.py            # Kelly Criterion
│       ├── mean_variance.py    # Markowitz
│       └── risk_parity.py      # Risk Parity
│
└── tests/
```

---

## 5. 인터페이스 정의

### 5.1 PortfolioAnalysis 결과 구조

```python
@dataclass
class PortfolioAnalysis:
    # 리스크 지표
    sharpe_ratio: float
    sortino_ratio: float
    max_drawdown: float
    var_95: float
    cvar_95: float
    
    # 상관관계
    correlation_matrix: pd.DataFrame
    
    # 최적 비중
    optimal_weights: Dict[str, float]  # 전략별 비중
    rebalance_suggestion: List[RebalanceAction]
```

### 5.2 RebalanceAction 구조

```python
@dataclass
class RebalanceAction:
    strategy: str           # 전략명
    current_weight: float   # 현재 비중
    target_weight: float    # 목표 비중
    action: str             # INCREASE | DECREASE | HOLD
    amount: float           # 조정 금액 (KRW)
```

---

## 6. 구현 로드맵

| 주차 | 작업 | 산출물 |
|------|------|--------|
| 7주차 | 리스크 지표 계산 | Sharpe, MDD, VaR 구현 |
| 7주차 | 상관관계 분석 | 상관 행렬, 히트맵 |
| 8주차+ | 자본 배분 최적화 | Kelly, Mean-Variance |

---

*— 문서 끝 —*
