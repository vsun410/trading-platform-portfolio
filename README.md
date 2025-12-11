# Trading Platform - Portfolio

포트폴리오 검증 & 최적화 환경

## 🎯 목적

- 전략 간 상관관계 분석
- 포트폴리오 최적화
- 리스크 지표 계산
- 자본 배분 결정

## 📊 분석 기능

| 기능 | 설명 | 상태 |
|:---|:---|:---:|
| 상관관계 분석 | 전략 간 상관계수 계산 | 🔴 예정 |
| Sharpe Ratio | 위험 대비 수익 비율 | 🔴 예정 |
| MDD | 최대 낙폭 | 🔴 예정 |
| VAR | Value at Risk | 🔴 예정 |
| 자본 배분 최적화 | Kelly, Mean-Variance | 🔴 예정 |

## 🏗️ 프로젝트 구조

```
trading-platform-portfolio/
├── README.md
├── pyproject.toml
├── docs/
│   ├── ANALYSIS_METHODS.md
│   └── OPTIMIZATION_GUIDE.md
├── src/
│   ├── analyzer/
│   │   ├── correlation.py
│   │   ├── risk.py
│   │   └── performance.py
│   └── optimizer/
│       ├── kelly.py
│       ├── mean_variance.py
│       └── risk_parity.py
└── tests/
```

## 🚀 빠른 시작

```bash
git clone https://github.com/vsun410/trading-platform-portfolio.git
cd trading-platform-portfolio
pip install -e .
```

## ⚙️ 리밸런싱

리밸런싱은 **수동**으로 진행합니다.

## 🔗 관련 레포

| 레포 | 역할 |
|:---|:---|
| [research](https://github.com/vsun410/trading-platform-research) | 전략 연구 |
| [order](https://github.com/vsun410/trading-platform-order) | 주문 실행 |
| [storage](https://github.com/vsun410/trading-platform-storage) | 데이터 저장소 |
