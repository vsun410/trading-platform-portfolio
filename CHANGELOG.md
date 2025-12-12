# Changelog

이 문서는 trading-platform-portfolio 레포지토리의 모든 주요 변경사항을 기록합니다.

형식: [Keep a Changelog](https://keepachangelog.com/ko/1.0.0/)  
버전 관리: [Semantic Versioning](https://semver.org/lang/ko/)

---

## [2.0.0] - 2025-12-12

### 🎯 핵심 변경: Risk Management 및 Paper Trading 시스템

Order 레포와 연동하는 **실시간 리스크 관리** 모듈과 실거래 전 검증을 위한 **Paper Trading** 시스템을 도입했습니다.

### Added (추가)

#### Risk Management 모듈 (P0)

##### 사전 거래 검사
- **PreTradeChecker** (`src/risk/pre_trade_check.py`)
  - 모든 검사를 **병렬 실행**하여 지연 최소화
  - 검사 항목:
  
  | 검사 | 기준 | 초과 시 |
  |------|------|--------|
  | 포지션 한도 | 자본의 95% | 주문 거부 |
  | 일일 손실 한도 | 5% | 신규 진입 금지 |
  | 드로다운 한도 | 20% | 신규 진입 금지 |
  | 익스포저 한도 | 100% | 주문 거부 |

##### 사후 거래 검사
- **PostTradeChecker** (`src/risk/post_trade_check.py`)
  - 포지션 불일치 감지 (업비트-바이낸스)
  - 마진 상태 모니터링
  - 일일 손익 및 익스포저 업데이트

##### 통합 관리자
- **RiskManager** (`src/risk/risk_manager.py`)
  - Order 레포에서 호출하는 메인 인터페이스
  - 검사 실패 시 Discord 알림 발송

#### VaR 계산기 (P1)
- **VaRCalculator** (`src/risk/var_calculator.py`)
  - Historical VaR: 과거 수익률 분포 기반
  - Parametric VaR: 정규분포 가정
  - CVaR (Expected Shortfall) 계산
  
  | VaR 값 | 의미 | 조치 |
  |--------|------|------|
  | < 2% | ✅ 낮은 위험 | 정상 운영 |
  | 2~5% | ⚠️ 중간 위험 | 모니터링 강화 |
  | > 5% | 🚨 높은 위험 | 포지션 축소 검토 |

#### Paper Trading 시스템 (P1)
- **VirtualExchange** (`src/paper_trading/virtual_exchange.py`)
  - 가상 거래소 시뮬레이션
  - 실제 시장 데이터 사용, 거래는 가상
  - 슬리피지 모델 적용 (VWAP 기반)
  - 수수료 계산 (0.1%)
  
  ```
  검증 기준:
  • 기간: 1주일 이상
  • 거래 수: 10회 이상
  • 승률: > 50%
  • 샤프: > 0.5
  • 슬리피지: < 0.1%
  ```

#### 점진적 자본 확대 (P1)
- **GradualScaler** (`src/capital/gradual_scaler.py`)
  - Paper Trading 통과 후 단계별 자본 확대
  
  | 단계 | 자본 비율 | 최대 손실 | 최소 기간 |
  |------|----------|----------|----------|
  | Paper | 0% | - | 7일 |
  | Stage 1 | 10% | 2% | 7일 |
  | Stage 2 | 25% | 3% | 7일 |
  | Stage 3 | 50% | 4% | 7일 |
  | Stage 4 | 95% | 5% | 무기한 |

  - 승격 조건: 손실 < 허용치, 승률 유지, 시스템 오류 없음
  - 강등 조건: 최대 손실 초과, 연속 3회 손실

### Changed (변경)

#### DETAILED_SPEC.md 전면 개편
- Risk Management 흐름도 추가
  ```
  Order → 사전검사(Portfolio) → 승인/거부 → 실행 → 사후검사
  ```
- VaR 해석 가이드
- Paper Trading 아키텍처
- 점진적 확대 일정표

#### 연관 레포 관계 업데이트
```
Before: research → portfolio (백테스트 결과만)
After:  order ↔ portfolio (양방향 리스크 검사)
```

#### 핵심 책임 확장
```
Before: 분석 및 최적화 (수동)
After:  + 실시간 리스크 관리 (자동)
        + Paper Trading (검증)
        + 자본 확대 관리 (단계별)
```

### 디렉토리 구조 변경

```
src/
├── risk/                       # 🆕 리스크 관리
│   ├── __init__.py
│   ├── risk_manager.py         # 통합 관리자
│   ├── pre_trade_check.py      # 사전 검사
│   ├── post_trade_check.py     # 사후 검사
│   ├── var_calculator.py       # VaR 계산
│   └── exposure_tracker.py     # 익스포저 추적
│
├── paper_trading/              # 🆕 Paper Trading
│   ├── __init__.py
│   ├── virtual_exchange.py     # 가상 거래소
│   ├── slippage_simulator.py   # 슬리피지 시뮬레이션
│   └── execution_bridge.py     # Live/Paper 전환
│
├── capital/                    # 🆕 자본 관리
│   ├── __init__.py
│   └── gradual_scaler.py       # 점진적 확대
│
├── analyzer/                   # 기존
│   ├── __init__.py
│   ├── correlation.py
│   ├── risk.py
│   └── performance.py
│
└── optimizer/                  # 기존
    ├── __init__.py
    ├── kelly.py
    ├── mean_variance.py
    └── risk_parity.py
```

### Order 레포 연동 예시

```python
# Order 레포에서 Portfolio의 RiskManager 사용
from portfolio.risk import RiskManager

class KimpExecutor:
    async def execute(self, signal: Signal):
        # 1. 사전 검사
        pre_check = await self.risk_manager.pre_trade_check(order)
        if not pre_check.approved:
            return None  # 거부
        
        # 2. 주문 실행
        fill = await self._execute_order(order)
        
        # 3. 사후 검사
        await self.risk_manager.post_trade_check(fill)
        
        return fill
```

---

## [1.0.0] - 2025-12-11

### Added
- 초기 레포지토리 생성
- 기본 분석 기능 설계
  - 리스크 지표 (Sharpe, Sortino, MDD, VaR, CVaR)
  - 상관관계 분석
  - 자본 배분 최적화 (Kelly, Mean-Variance, Risk Parity)

### 문서
- DETAILED_SPEC.md - 세부 기획서
- ANALYSIS_METHODS.md - 분석 방법론
- OPTIMIZATION_GUIDE.md - 최적화 가이드
- DESIGN_SYSTEM.md - Kinetic Minimalism 디자인 시스템

---

## 향후 계획 (Upcoming)

### [2.1.0] - 예정
- [ ] 상관관계 분석 구현 (P2)
- [ ] Kelly Criterion 최적화 (P2)
- [ ] 리스크 대시보드 (P2)
- [ ] 성과 리포트 자동 생성 (P3)

---

*— Changelog 끝 —*
