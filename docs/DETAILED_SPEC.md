# 📊 Portfolio 세부 기획서

**Repository:** trading-platform-portfolio  
**Version:** 2.0  
**Date:** 2025-12-12  
**Updated:** Risk Management 모듈, Paper Trading, 점진적 자본 확대

---

## 1. 개요

### 1.1 목적

전략 간 상관관계를 분석하고, **실시간 리스크 관리** 및 포트폴리오 최적화를 통해 위험 대비 수익을 극대화합니다. 리밸런싱은 **수동**으로 진행하며, 의사결정을 위한 분석 도구를 제공합니다.

### 1.2 핵심 책임

- **Risk Management:** 사전/사후 거래 검사, 실시간 모니터링
- **상관관계 분석:** 전략 간 상관계수 계산, 다각화 효과 검증
- **리스크 지표:** Sharpe Ratio, MDD, VaR, CVaR 계산
- **자본 배분:** Kelly Criterion, Mean-Variance, Risk Parity
- **Paper Trading:** 실거래 전 검증 시스템
- **점진적 확대:** 단계별 자본 투입 관리

### 1.3 연관 레포지토리

| 레포 | 관계 | 데이터 흐름 |
|------|------|-------------|
| research | 백테스트 결과 제공 | research → 성과 데이터 → portfolio |
| storage | 거래 이력 제공 | storage → 체결/포지션 → portfolio |
| order | **리스크 검사 요청** | order ↔ portfolio (사전 검사) |

---

## 2. Risk Management 모듈 (P0)

### 2.1 개요

Order 레포와 연동하여 **모든 거래에 대한 사전/사후 리스크 검사**를 수행합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Risk Management 흐름                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Order 레포                     Portfolio 레포                 │
│   ──────────                     ──────────────                 │
│                                                                 │
│   1. 신호 수신 ─────────────────▶ 2. 사전 검사                  │
│                                    • 포지션 한도                │
│                                    • 일일 손실 한도             │
│                                    • 익스포저 한도              │
│                                    • 드로다운 한도              │
│                                                                 │
│   3. 검사 통과? ◀────────────────── (OK/REJECT)                │
│      │                                                          │
│      ├── YES → 주문 실행                                        │
│      │         │                                                │
│      │         ▼                                                │
│      │    4. 체결 완료 ──────────▶ 5. 사후 검사                  │
│      │                              • 포지션 검증               │
│      │                              • 마진 상태                 │
│      │                              • 익스포저 업데이트         │
│      │                                                          │
│      └── NO → 주문 거부, 알림 발송                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 사전 거래 검사 (Pre-Trade Check)

```python
# src/risk/pre_trade_check.py

from dataclasses import dataclass
from decimal import Decimal
from typing import List, Optional
import asyncio

@dataclass
class PreTradeCheckResult:
    """사전 검사 결과"""
    approved: bool
    checks_passed: List[str]
    checks_failed: List[str]
    rejection_reason: Optional[str] = None


class PreTradeChecker:
    """
    사전 거래 검사기
    
    모든 검사를 병렬로 실행하여 지연 최소화
    """
    
    def __init__(
        self,
        position_limit: Decimal = Decimal("0.95"),    # 자본의 95%
        daily_loss_limit: Decimal = Decimal("0.05"),  # 일일 최대 손실 5%
        max_drawdown: Decimal = Decimal("0.20"),      # 최대 드로다운 20%
        max_exposure: Decimal = Decimal("1.0"),       # 최대 익스포저 100%
    ):
        self.position_limit = position_limit
        self.daily_loss_limit = daily_loss_limit
        self.max_drawdown = max_drawdown
        self.max_exposure = max_exposure
    
    async def check(
        self, 
        order: 'Order',
        portfolio_state: 'PortfolioState'
    ) -> PreTradeCheckResult:
        """
        모든 사전 검사 실행 (병렬)
        """
        checks = await asyncio.gather(
            self._check_position_limit(order, portfolio_state),
            self._check_daily_loss_limit(portfolio_state),
            self._check_drawdown_limit(portfolio_state),
            self._check_exposure_limit(order, portfolio_state),
            return_exceptions=True
        )
        
        passed = []
        failed = []
        
        check_names = [
            "position_limit",
            "daily_loss_limit", 
            "drawdown_limit",
            "exposure_limit"
        ]
        
        for name, result in zip(check_names, checks):
            if isinstance(result, Exception):
                failed.append(f"{name}: {str(result)}")
            elif result:
                passed.append(name)
            else:
                failed.append(name)
        
        approved = len(failed) == 0
        
        return PreTradeCheckResult(
            approved=approved,
            checks_passed=passed,
            checks_failed=failed,
            rejection_reason=failed[0] if failed else None
        )
    
    async def _check_position_limit(
        self, 
        order: 'Order',
        state: 'PortfolioState'
    ) -> bool:
        """
        포지션 한도 검사
        
        규칙: 신규 포지션이 자본의 95%를 초과하면 거부
        """
        total_capital = state.total_equity
        current_position = state.current_position_value
        order_value = order.quantity * order.estimated_price
        
        new_position = current_position + order_value
        position_ratio = new_position / total_capital
        
        return position_ratio <= self.position_limit
    
    async def _check_daily_loss_limit(
        self, 
        state: 'PortfolioState'
    ) -> bool:
        """
        일일 손실 한도 검사
        
        규칙: 오늘 손실이 5%를 초과하면 신규 진입 금지
        """
        daily_pnl = state.daily_pnl
        daily_loss_ratio = -daily_pnl / state.day_start_equity
        
        return daily_loss_ratio < self.daily_loss_limit
    
    async def _check_drawdown_limit(
        self, 
        state: 'PortfolioState'
    ) -> bool:
        """
        드로다운 한도 검사
        
        규칙: 현재 드로다운이 20%를 초과하면 신규 진입 금지
        """
        current_drawdown = state.current_drawdown
        return current_drawdown < self.max_drawdown
    
    async def _check_exposure_limit(
        self, 
        order: 'Order',
        state: 'PortfolioState'
    ) -> bool:
        """
        익스포저 한도 검사
        
        규칙: 총 익스포저가 100%를 초과하면 거부
        (김프 차익거래는 헤지되어 있으므로 Net Exposure ≈ 0)
        """
        # 김프 차익거래의 경우 Net Exposure는 0에 가까움
        # Gross Exposure 기준으로 검사
        gross_exposure = state.gross_exposure
        return gross_exposure <= self.max_exposure
```

### 2.3 사후 거래 검사 (Post-Trade Check)

```python
# src/risk/post_trade_check.py

from dataclasses import dataclass
from typing import List

@dataclass
class PostTradeCheckResult:
    """사후 검사 결과"""
    all_ok: bool
    warnings: List[str]
    alerts: List[str]


class PostTradeChecker:
    """
    사후 거래 검사기
    
    체결 후 포트폴리오 상태 검증
    """
    
    async def check(
        self, 
        fill: 'Fill',
        portfolio_state: 'PortfolioState'
    ) -> PostTradeCheckResult:
        """
        사후 검사 실행
        """
        warnings = []
        alerts = []
        
        # 1. 포지션 불일치 검사
        if await self._check_position_mismatch(portfolio_state):
            alerts.append("포지션 불일치 감지")
        
        # 2. 마진 상태 검사 (바이낸스 선물)
        margin_ratio = await self._check_margin_status(portfolio_state)
        if margin_ratio < 0.3:
            alerts.append(f"마진 비율 위험: {margin_ratio:.1%}")
        elif margin_ratio < 0.5:
            warnings.append(f"마진 비율 주의: {margin_ratio:.1%}")
        
        # 3. 일일 손익 업데이트
        await self._update_daily_pnl(fill, portfolio_state)
        
        # 4. 익스포저 업데이트
        await self._update_exposure(fill, portfolio_state)
        
        return PostTradeCheckResult(
            all_ok=len(alerts) == 0,
            warnings=warnings,
            alerts=alerts
        )
    
    async def _check_position_mismatch(
        self, 
        state: 'PortfolioState'
    ) -> bool:
        """업비트-바이낸스 포지션 일치 여부"""
        upbit_btc = state.upbit_btc_balance
        binance_short = abs(state.binance_short_position)
        
        # 1% 이상 차이나면 불일치
        threshold = max(upbit_btc, binance_short) * Decimal("0.01")
        return abs(upbit_btc - binance_short) > threshold
    
    async def _check_margin_status(
        self, 
        state: 'PortfolioState'
    ) -> float:
        """바이낸스 마진 비율 확인"""
        return state.binance_margin_ratio
```

### 2.4 Risk Manager (통합)

```python
# src/risk/risk_manager.py

from typing import Optional
import logging

logger = logging.getLogger(__name__)

class RiskManager:
    """
    통합 리스크 관리자
    
    Order 레포에서 호출하는 메인 인터페이스
    """
    
    def __init__(
        self,
        pre_checker: PreTradeChecker,
        post_checker: PostTradeChecker,
        notifier: 'DiscordNotifier'
    ):
        self.pre_checker = pre_checker
        self.post_checker = post_checker
        self.notifier = notifier
    
    async def pre_trade_check(
        self, 
        order: 'Order'
    ) -> PreTradeCheckResult:
        """
        거래 전 검사
        
        Returns:
            PreTradeCheckResult: approved=True면 주문 진행
        """
        state = await self._get_portfolio_state()
        result = await self.pre_checker.check(order, state)
        
        if not result.approved:
            logger.warning(f"Order rejected: {result.rejection_reason}")
            await self.notifier.send(
                "⚠️ 주문 거부",
                f"사유: {result.rejection_reason}"
            )
        
        return result
    
    async def post_trade_check(
        self, 
        fill: 'Fill'
    ) -> PostTradeCheckResult:
        """
        거래 후 검사
        """
        state = await self._get_portfolio_state()
        result = await self.post_checker.check(fill, state)
        
        if result.alerts:
            await self.notifier.send_critical(
                "🚨 리스크 알림",
                "\n".join(result.alerts)
            )
        
        return result
    
    async def _get_portfolio_state(self) -> 'PortfolioState':
        """현재 포트폴리오 상태 조회"""
        # Storage에서 조회
        pass
```

---

## 3. VaR 계산기 (P1)

### 3.1 Historical VaR

```python
# src/risk/var_calculator.py

import numpy as np
import pandas as pd
from typing import Dict

class VaRCalculator:
    """
    VaR (Value at Risk) 계산기
    
    방법:
    1. Historical VaR: 과거 수익률 분포 기반
    2. Parametric VaR: 정규분포 가정
    """
    
    def historical_var(
        self, 
        returns: pd.Series,
        confidence: float = 0.95,
        holding_period: int = 1
    ) -> Dict[str, float]:
        """
        Historical VaR 계산
        
        Args:
            returns: 일별 수익률 시리즈
            confidence: 신뢰수준 (0.95 = 95%)
            holding_period: 보유 기간 (일)
            
        Returns:
            {'var': VaR값, 'cvar': CVaR값}
        """
        # 보유 기간 조정
        if holding_period > 1:
            returns = returns.rolling(holding_period).sum().dropna()
        
        # VaR: 하위 (1-confidence) 백분위수
        var_percentile = 1 - confidence  # 5%
        var = np.percentile(returns, var_percentile * 100)
        
        # CVaR (Expected Shortfall): VaR 이하 평균
        cvar = returns[returns <= var].mean()
        
        return {
            'var': abs(var),      # 양수로 표현
            'cvar': abs(cvar),
            'confidence': confidence,
            'holding_period': holding_period
        }
    
    def parametric_var(
        self,
        returns: pd.Series,
        confidence: float = 0.95,
        holding_period: int = 1
    ) -> Dict[str, float]:
        """
        Parametric VaR 계산 (정규분포 가정)
        
        공식: VaR = μ - z × σ × √t
        """
        from scipy import stats
        
        mu = returns.mean() * holding_period
        sigma = returns.std() * np.sqrt(holding_period)
        
        # Z-score for confidence level
        z = stats.norm.ppf(1 - confidence)  # 1.645 for 95%
        
        var = -(mu + z * sigma)
        
        return {
            'var': max(0, var),
            'mu': mu,
            'sigma': sigma,
            'z_score': z,
            'confidence': confidence
        }
```

### 3.2 VaR 해석

| VaR 값 | 의미 | 조치 |
|--------|------|------|
| < 2% | ✅ 낮은 위험 | 정상 운영 |
| 2% ~ 5% | ⚠️ 중간 위험 | 모니터링 강화 |
| > 5% | 🚨 높은 위험 | 포지션 축소 검토 |

---

## 4. Paper Trading 시스템 (P1)

### 4.1 개요

실거래 전 전략을 검증하는 **가상 거래 시스템**입니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Paper Trading 아키텍처                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Research 레포          Paper Trading           실거래         │
│   ────────────          ──────────────          ────────        │
│                                                                 │
│   백테스트 완료                                                  │
│        │                                                        │
│        ▼                                                        │
│   전략 검증 ─────────▶ 1주일 Paper Trading                      │
│                              │                                  │
│                              ▼                                  │
│                        성과 검증                                │
│                         (승률, 샤프)                            │
│                              │                                  │
│                   ┌─────────┴─────────┐                        │
│                   │ 기준 충족?         │                        │
│                   └─────────┬─────────┘                        │
│                             │                                   │
│            ┌────────────────┼────────────────┐                 │
│            │ YES            │                 │ NO              │
│            ▼                                  ▼                 │
│      실거래 전환                         전략 재검토            │
│      (10% → 25% → 50% → 100%)                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 가상 거래소

```python
# src/paper_trading/virtual_exchange.py

from dataclasses import dataclass, field
from decimal import Decimal
from datetime import datetime
from typing import Dict, List, Optional

@dataclass
class VirtualPosition:
    """가상 포지션"""
    symbol: str
    side: str              # LONG, SHORT
    quantity: Decimal
    entry_price: Decimal
    entry_time: datetime
    unrealized_pnl: Decimal = Decimal("0")


@dataclass
class VirtualFill:
    """가상 체결"""
    timestamp: datetime
    symbol: str
    side: str
    quantity: Decimal
    price: Decimal
    slippage: Decimal
    commission: Decimal


class VirtualExchange:
    """
    가상 거래소
    
    실제 시장 데이터를 사용하지만, 거래는 가상으로 실행
    """
    
    def __init__(
        self,
        initial_capital: Decimal = Decimal("20000000"),
        slippage_model: 'SlippageModel' = None
    ):
        self.cash = initial_capital
        self.initial_capital = initial_capital
        self.positions: Dict[str, VirtualPosition] = {}
        self.fills: List[VirtualFill] = []
        self.slippage_model = slippage_model
    
    async def market_buy(
        self, 
        symbol: str, 
        quantity: Decimal,
        market_price: Decimal,
        orderbook: Optional[Dict] = None
    ) -> VirtualFill:
        """가상 시장가 매수"""
        
        # 슬리피지 계산 (실제 호가 기반)
        if self.slippage_model and orderbook:
            exec_price, slippage = self.slippage_model.calculate(
                'BUY', quantity, orderbook
            )
        else:
            # 기본 슬리피지 0.05%
            slippage = market_price * Decimal("0.0005")
            exec_price = market_price + slippage
        
        # 수수료 (0.1%)
        commission = exec_price * quantity * Decimal("0.001")
        
        # 잔고 차감
        total_cost = exec_price * quantity + commission
        if total_cost > self.cash:
            raise InsufficientFundsError(f"잔고 부족: {self.cash} < {total_cost}")
        
        self.cash -= total_cost
        
        # 포지션 업데이트
        if symbol in self.positions:
            pos = self.positions[symbol]
            # 평균 단가 계산
            total_qty = pos.quantity + quantity
            avg_price = (pos.entry_price * pos.quantity + exec_price * quantity) / total_qty
            pos.quantity = total_qty
            pos.entry_price = avg_price
        else:
            self.positions[symbol] = VirtualPosition(
                symbol=symbol,
                side='LONG',
                quantity=quantity,
                entry_price=exec_price,
                entry_time=datetime.utcnow()
            )
        
        # 체결 기록
        fill = VirtualFill(
            timestamp=datetime.utcnow(),
            symbol=symbol,
            side='BUY',
            quantity=quantity,
            price=exec_price,
            slippage=slippage,
            commission=commission
        )
        self.fills.append(fill)
        
        return fill
    
    async def market_sell(
        self, 
        symbol: str, 
        quantity: Decimal,
        market_price: Decimal,
        orderbook: Optional[Dict] = None
    ) -> VirtualFill:
        """가상 시장가 매도"""
        # 구현 (market_buy와 유사)
        pass
    
    def get_equity(self, current_prices: Dict[str, Decimal]) -> Decimal:
        """현재 총 자산 계산"""
        equity = self.cash
        
        for symbol, pos in self.positions.items():
            current_price = current_prices.get(symbol, pos.entry_price)
            if pos.side == 'LONG':
                equity += pos.quantity * current_price
            else:
                equity += pos.quantity * (2 * pos.entry_price - current_price)
        
        return equity
    
    def get_performance(self, current_prices: Dict[str, Decimal]) -> Dict:
        """성과 지표 계산"""
        equity = self.get_equity(current_prices)
        total_return = (equity - self.initial_capital) / self.initial_capital
        
        # 일별 수익률 계산 (fills 기반)
        # ...
        
        return {
            'total_equity': equity,
            'total_return': total_return,
            'total_trades': len(self.fills),
            'total_commission': sum(f.commission for f in self.fills),
            'avg_slippage': sum(f.slippage for f in self.fills) / len(self.fills) if self.fills else 0
        }
```

### 4.3 Paper Trading 검증 기준

| 지표 | 기준 | 실거래 전환 조건 |
|------|------|-----------------|
| 기간 | 1주일 이상 | 최소 7일 운영 |
| 거래 수 | 10회 이상 | 통계적 유의성 |
| 승률 | > 50% | 백테스트와 유사 |
| 샤프 | > 0.5 | 양의 위험조정수익 |
| 슬리피지 | < 0.1% | 현실적 수준 |

---

## 5. 점진적 자본 확대 (Gradual Scale-in)

### 5.1 개요

Paper Trading 통과 후 **단계적으로 자본을 확대**합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    점진적 자본 확대 일정                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Paper Trading ──▶ Week 1 ──▶ Week 2 ──▶ Week 3 ──▶ Week 4   │
│                      10%       25%       50%       100%        │
│                                                                 │
│   각 단계 승격 조건:                                            │
│   • 손실 < 단계별 허용치                                        │
│   • 승률 유지 (백테스트 대비 -10% 이내)                         │
│   • 시스템 오류 없음                                            │
│                                                                 │
│   강등 조건:                                                    │
│   • 단계별 최대 손실 초과                                       │
│   • 연속 3회 손실                                               │
│   • 시스템 오류 발생                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 구현

```python
# src/capital/gradual_scaler.py

from dataclasses import dataclass
from decimal import Decimal
from enum import Enum
from typing import Optional

class ScaleStage(Enum):
    PAPER = 0        # Paper Trading
    STAGE_1 = 1      # 10%
    STAGE_2 = 2      # 25%
    STAGE_3 = 3      # 50%
    STAGE_4 = 4      # 100%


@dataclass
class StageConfig:
    """단계별 설정"""
    stage: ScaleStage
    capital_ratio: Decimal
    max_loss: Decimal        # 단계별 최대 허용 손실
    min_days: int           # 최소 운영 일수
    min_trades: int         # 최소 거래 수


class GradualScaler:
    """
    점진적 자본 확대 관리자
    """
    
    STAGE_CONFIGS = {
        ScaleStage.PAPER: StageConfig(
            stage=ScaleStage.PAPER,
            capital_ratio=Decimal("0"),
            max_loss=Decimal("0"),  # Paper는 손실 없음
            min_days=7,
            min_trades=10
        ),
        ScaleStage.STAGE_1: StageConfig(
            stage=ScaleStage.STAGE_1,
            capital_ratio=Decimal("0.10"),  # 10%
            max_loss=Decimal("0.02"),       # 최대 2% 손실
            min_days=7,
            min_trades=5
        ),
        ScaleStage.STAGE_2: StageConfig(
            stage=ScaleStage.STAGE_2,
            capital_ratio=Decimal("0.25"),  # 25%
            max_loss=Decimal("0.03"),       # 최대 3% 손실
            min_days=7,
            min_trades=5
        ),
        ScaleStage.STAGE_3: StageConfig(
            stage=ScaleStage.STAGE_3,
            capital_ratio=Decimal("0.50"),  # 50%
            max_loss=Decimal("0.04"),       # 최대 4% 손실
            min_days=7,
            min_trades=5
        ),
        ScaleStage.STAGE_4: StageConfig(
            stage=ScaleStage.STAGE_4,
            capital_ratio=Decimal("0.95"),  # 95% (5% 예비비)
            max_loss=Decimal("0.05"),       # 최대 5% 손실
            min_days=0,                     # 무기한
            min_trades=0
        ),
    }
    
    def __init__(self, total_capital: Decimal):
        self.total_capital = total_capital
        self.current_stage = ScaleStage.PAPER
        self.stage_start_date = None
        self.stage_trades = 0
        self.stage_pnl = Decimal("0")
    
    def get_available_capital(self) -> Decimal:
        """현재 단계의 사용 가능 자본"""
        config = self.STAGE_CONFIGS[self.current_stage]
        return self.total_capital * config.capital_ratio
    
    def can_promote(self, current_pnl: Decimal, days: int, trades: int) -> bool:
        """
        다음 단계 승격 가능 여부
        """
        if self.current_stage == ScaleStage.STAGE_4:
            return False  # 최종 단계
        
        config = self.STAGE_CONFIGS[self.current_stage]
        
        # 조건 확인
        loss_ok = current_pnl >= -config.max_loss * self.total_capital
        days_ok = days >= config.min_days
        trades_ok = trades >= config.min_trades
        
        return loss_ok and days_ok and trades_ok
    
    def promote(self) -> ScaleStage:
        """다음 단계로 승격"""
        if self.current_stage.value < ScaleStage.STAGE_4.value:
            self.current_stage = ScaleStage(self.current_stage.value + 1)
            self._reset_stage_stats()
        return self.current_stage
    
    def should_demote(self, current_pnl: Decimal) -> bool:
        """강등 여부 확인"""
        config = self.STAGE_CONFIGS[self.current_stage]
        return current_pnl < -config.max_loss * self.total_capital
    
    def demote(self) -> ScaleStage:
        """이전 단계로 강등"""
        if self.current_stage.value > ScaleStage.PAPER.value:
            self.current_stage = ScaleStage(self.current_stage.value - 1)
            self._reset_stage_stats()
        return self.current_stage
    
    def _reset_stage_stats(self):
        """단계 통계 초기화"""
        from datetime import datetime
        self.stage_start_date = datetime.utcnow()
        self.stage_trades = 0
        self.stage_pnl = Decimal("0")
```

---

## 6. 분석 기능 명세

### 6.1 리스크 지표

| 지표 | 공식 | 용도 |
|------|------|------|
| Sharpe Ratio | (Rp - Rf) / σp | 위험 조정 수익률 |
| Sortino Ratio | (Rp - Rf) / σd | 하방 위험만 고려 |
| MDD | max(Peak - Trough) / Peak | 최대 손실 위험 |
| VaR (95%) | Percentile(5%, returns) | 일별 예상 최대 손실 |
| CVaR (ES) | E[X \| X ≤ VaR] | 극단 손실 평균 |
| Calmar Ratio | CAGR / MDD | 수익 대비 낙폭 |

### 6.2 상관관계 분석

| 상관계수 | 해석 | 포트폴리오 영향 |
|----------|------|-----------------|
| 0.7 ~ 1.0 | 강한 양의 상관 | 다각화 효과 낮음 |
| 0.3 ~ 0.7 | 중간 양의 상관 | 일부 다각화 효과 |
| -0.3 ~ 0.3 | 낮은 상관 | **우수한 다각화** ✓ |
| -1.0 ~ -0.3 | 음의 상관 | **헤지 효과** ✓ |

---

## 7. 디렉토리 구조 (업데이트)

```
trading-platform-portfolio/
├── README.md
├── pyproject.toml
│
├── docs/
│   ├── README.md
│   ├── ANALYSIS_METHODS.md
│   ├── OPTIMIZATION_GUIDE.md
│   └── DETAILED_SPEC.md          # 이 문서
│
├── src/
│   ├── risk/                     # 🆕 리스크 관리
│   │   ├── __init__.py
│   │   ├── risk_manager.py       # 통합 관리자
│   │   ├── pre_trade_check.py    # 사전 검사
│   │   ├── post_trade_check.py   # 사후 검사
│   │   ├── var_calculator.py     # VaR 계산
│   │   └── exposure_tracker.py   # 익스포저 추적
│   │
│   ├── paper_trading/            # 🆕 Paper Trading
│   │   ├── __init__.py
│   │   ├── virtual_exchange.py   # 가상 거래소
│   │   ├── slippage_simulator.py # 슬리피지 시뮬레이션
│   │   └── execution_bridge.py   # Live/Paper 전환
│   │
│   ├── capital/                  # 🆕 자본 관리
│   │   ├── __init__.py
│   │   └── gradual_scaler.py     # 점진적 확대
│   │
│   ├── analyzer/                 # 분석 모듈
│   │   ├── __init__.py
│   │   ├── correlation.py
│   │   ├── risk.py
│   │   └── performance.py
│   │
│   └── optimizer/                # 최적화 모듈
│       ├── __init__.py
│       ├── kelly.py
│       ├── mean_variance.py
│       └── risk_parity.py
│
└── tests/
    ├── test_risk_manager.py
    ├── test_var_calculator.py
    └── test_gradual_scaler.py
```

---

## 8. 구현 로드맵 (업데이트)

| 우선순위 | 작업 | 산출물 | Phase |
|----------|------|--------|-------|
| **P0** | Risk Management 사전 검사 | PreTradeChecker | 3 |
| **P0** | Risk Management 사후 검사 | PostTradeChecker | 3 |
| **P1** | VaR 계산기 | VaRCalculator | 4 |
| **P1** | Paper Trading | VirtualExchange | 4 |
| **P1** | 점진적 자본 확대 | GradualScaler | 4 |
| P2 | 상관관계 분석 | CorrelationAnalyzer | 5 |
| P2 | 자본 배분 최적화 | Kelly, Mean-Variance | 5+ |

---

## 9. Order 레포 연동 예시

```python
# Order 레포에서 Portfolio의 RiskManager 사용

from portfolio.risk import RiskManager

class KimpExecutor:
    def __init__(self, risk_manager: RiskManager):
        self.risk_manager = risk_manager
    
    async def execute(self, signal: Signal):
        # 1. 사전 검사
        pre_check = await self.risk_manager.pre_trade_check(order)
        
        if not pre_check.approved:
            logger.warning(f"주문 거부: {pre_check.rejection_reason}")
            return None
        
        # 2. 주문 실행
        fill = await self._execute_order(order)
        
        # 3. 사후 검사
        post_check = await self.risk_manager.post_trade_check(fill)
        
        if post_check.alerts:
            # 긴급 알림 발송됨
            pass
        
        return fill
```

---

*— 문서 끝 —*
