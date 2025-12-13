# 🛡️ 김치프리미엄 리스크 관리 프레임워크 (Ver 3.0)

## 1. 리스크 관리 개요

### 1.1 핵심 철학

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Risk Management Philosophy                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│   🎯 핵심 원칙: "절대 손절하지 않는다 (No Stop Loss)"                │
│                                                                       │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │  왜 손절하지 않는가?                                         │   │
│   │                                                               │   │
│   │  1. 델타 뉴트럴 = 방향성 리스크 제거                         │   │
│   │  2. 김프는 결국 평균 회귀 (Mean Reversion)                   │   │
│   │  3. 손절 = 수수료 손실 + 재진입 비용 발생                    │   │
│   │  4. 'L자형 장세'도 Breakout Rescue로 탈출 가능 (Ver 3.0)     │   │
│   │                                                               │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
│   대신 관리하는 리스크:                                              │
│   ├─ 환율 리스크 → 환율 필터 (진입 차단)                            │
│   ├─ 장기 보유 리스크 → Breakout Rescue (탈출구 마련)               │
│   ├─ 실행 리스크 → 동시 주문 + 롤백 메커니즘                        │
│   └─ 자금 묶임 리스크 → 예비비 5% 확보                              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 Ver 3.0 리스크 관리 변경사항

| 리스크 | 기존 대응 | Ver 3.0 대응 |
|:---|:---|:---|
| 환율 급등 | 없음 | **환율 필터 (진입 차단)** |
| L자형 장세 | 무기한 대기 | **Breakout Rescue (탈출)** |
| 장기 보유 | 타임컷 (폐기됨) | **BB 돌파 시 탈출** |

---

## 2. 환율 리스크 관리 (Ver 3.0 신규)

### 2.1 환율과 김프의 상관관계

```
환율 급등 → 김프 구조적 하락

┌────────────────────────────────────────────────────────────┐
│                                                              │
│   환율 ↑↑       김프 ↓↓                                     │
│     │             │                                          │
│     │             └─ 해외 가격 KRW 환산값 상승               │
│     │                → 국내 대비 프리미엄 감소               │
│     │                                                        │
│     └─ 달러 강세 시 외국인 자금 유출 경향                    │
│        → 국내 거래소 매수세 약화                             │
│                                                              │
│   결론: 환율 급등 구간에서 김프 롱 진입 = 역풍 맞음          │
│                                                              │
└────────────────────────────────────────────────────────────┘
```

### 2.2 환율 필터 로직

```python
class FXRiskManager:
    """환율 리스크 관리자"""
    
    # 파라미터
    MA_PERIOD = 720           # 12시간 (분 단위)
    SURGE_THRESHOLD = 1.001   # +0.1% 급등 기준
    
    def __init__(self, fx_data_feed):
        self.fx_feed = fx_data_feed
        self.blocked_count = 0
        self.last_check_result = None
    
    def is_entry_allowed(self) -> bool:
        """진입 허용 여부 체크"""
        current_rate = self.fx_feed.get_current_rate()
        ma_12h = self.fx_feed.get_ma(self.MA_PERIOD)
        
        threshold = ma_12h * self.SURGE_THRESHOLD
        is_blocked = current_rate > threshold
        
        if is_blocked:
            self.blocked_count += 1
            surge_pct = ((current_rate / ma_12h) - 1) * 100
            print(f"⛔ FX Filter: {current_rate:.2f} > {threshold:.2f} (+{surge_pct:.2f}%)")
        
        self.last_check_result = {
            'timestamp': datetime.now(),
            'current_rate': current_rate,
            'ma_12h': ma_12h,
            'threshold': threshold,
            'is_blocked': is_blocked
        }
        
        return not is_blocked
    
    def get_risk_level(self) -> str:
        """현재 환율 리스크 레벨"""
        if not self.last_check_result:
            return "UNKNOWN"
        
        surge_pct = (
            (self.last_check_result['current_rate'] / 
             self.last_check_result['ma_12h']) - 1
        ) * 100
        
        if surge_pct > 0.3:
            return "HIGH"      # 0.3% 이상 급등
        elif surge_pct > 0.1:
            return "MEDIUM"    # 0.1~0.3% 급등
        elif surge_pct > 0:
            return "LOW"       # 0~0.1% 상승
        else:
            return "SAFE"      # 하락 또는 안정
```

### 2.3 환율 필터 통계

```sql
-- 환율 필터 효과 분석 쿼리
SELECT 
    DATE(timestamp) AS date,
    COUNT(*) AS total_signals,
    SUM(CASE WHEN is_blocked THEN 1 ELSE 0 END) AS blocked_count,
    ROUND(
        SUM(CASE WHEN is_blocked THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 
        2
    ) AS block_rate_pct
FROM fx_filter_log
GROUP BY DATE(timestamp)
ORDER BY date DESC;
```

---

## 3. 포지션 리스크 관리

### 3.1 Dual Track 청산 시스템

```
┌─────────────────────────────────────────────────────────────────────┐
│                  Position Exit Risk Management                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│   Track A: 정상 익절 (Target Hit)                                    │
│   ├─ 조건: 수익률 ≥ 0.7%                                            │
│   ├─ 리스크: 없음 (정상 수익 실현)                                   │
│   └─ 예상 비율: ~44% (백테스트 기준)                                 │
│                                                                       │
│   Track B: Breakout Rescue (Ver 3.0)                                 │
│   ├─ 조건: 수익률 ≥ 0.48% AND 김프 > BB 상단                        │
│   ├─ 리스크: 약수익 (순익 0.1%)                                      │
│   ├─ 의미: "L자형 장세 탈출구"                                       │
│   └─ 예상 비율: ~56% (백테스트 기준)                                 │
│                                                                       │
│   ※ 손절 없음 → 100% 수익 청산 (Target 또는 Breakout)               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 장기 보유 리스크 모니터링

```python
class PositionRiskMonitor:
    """포지션 리스크 모니터"""
    
    # 경고 기준
    HOLDING_WARNING_HOURS = 24      # 24시간 보유 시 경고
    HOLDING_CRITICAL_HOURS = 72     # 72시간 보유 시 위험
    
    def __init__(self, position):
        self.position = position
        self.alerts_sent = set()
    
    def check_holding_risk(self) -> dict:
        """보유 시간 리스크 체크"""
        holding_hours = self._get_holding_hours()
        
        status = {
            'holding_hours': holding_hours,
            'risk_level': 'NORMAL',
            'action': None
        }
        
        if holding_hours >= self.HOLDING_CRITICAL_HOURS:
            status['risk_level'] = 'CRITICAL'
            status['action'] = 'Breakout 조건 완화 검토'
            self._send_alert('CRITICAL', holding_hours)
        
        elif holding_hours >= self.HOLDING_WARNING_HOURS:
            status['risk_level'] = 'WARNING'
            status['action'] = '모니터링 강화'
            self._send_alert('WARNING', holding_hours)
        
        return status
    
    def check_drawdown(self, current_kimp: float) -> dict:
        """손실폭(Drawdown) 체크"""
        entry_kimp = self.position.entry_kimp
        drawdown = entry_kimp - current_kimp
        drawdown_pct = drawdown  # 김프 자체가 %
        
        # 손절 없지만 모니터링은 필요
        status = {
            'drawdown_pct': drawdown_pct,
            'risk_level': 'NORMAL'
        }
        
        if drawdown_pct > 1.0:  # 1% 이상 역김프
            status['risk_level'] = 'ELEVATED'
        
        if drawdown_pct > 2.0:  # 2% 이상 역김프
            status['risk_level'] = 'HIGH'
        
        return status
    
    def _get_holding_hours(self) -> float:
        elapsed = datetime.now() - self.position.entry_timestamp
        return elapsed.total_seconds() / 3600
    
    def _send_alert(self, level: str, hours: float):
        alert_key = f"{level}_{int(hours // 24)}"
        if alert_key not in self.alerts_sent:
            # Telegram 알림 전송
            self.alerts_sent.add(alert_key)
```

---

## 4. 실행 리스크 관리

### 4.1 동시 주문 실패 시나리오

```
┌─────────────────────────────────────────────────────────────────────┐
│                  Execution Risk Scenarios                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│   시나리오 1: Upbit만 체결                                           │
│   ├─ 리스크: 델타 노출 (현물만 보유)                                 │
│   └─ 대응: 즉시 Upbit 반대매매 (손실 최소화)                         │
│                                                                       │
│   시나리오 2: Binance만 체결                                         │
│   ├─ 리스크: 숏 포지션만 보유 (방향성 리스크)                        │
│   └─ 대응: 즉시 Binance 포지션 청산                                  │
│                                                                       │
│   시나리오 3: 양쪽 부분 체결                                         │
│   ├─ 리스크: 불균형 델타                                             │
│   └─ 대응: 작은 쪽 기준으로 조정 후 나머지 청산                      │
│                                                                       │
│   시나리오 4: 청산 중 한쪽 실패                                      │
│   ├─ 리스크: 불완전 청산 (한쪽만 포지션 남음)                        │
│   └─ 대응: 재시도 (최대 3회) → 수동 개입 알림                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 롤백 메커니즘

```python
class ExecutionRiskManager:
    """실행 리스크 관리자"""
    
    MAX_RETRIES = 3
    RETRY_DELAY = 1.0
    
    async def safe_execute_entry(self, signal) -> dict:
        """안전한 진입 실행"""
        
        # 동시 실행
        results = await asyncio.gather(
            self._execute_upbit(signal),
            self._execute_binance(signal),
            return_exceptions=True
        )
        
        upbit_result, binance_result = results
        
        # 양쪽 성공
        if not isinstance(upbit_result, Exception) and \
           not isinstance(binance_result, Exception):
            return {'success': True, 'results': results}
        
        # 롤백 필요
        await self._rollback(upbit_result, binance_result)
        
        return {
            'success': False,
            'error': f"Upbit: {upbit_result}, Binance: {binance_result}"
        }
    
    async def _rollback(self, upbit_result, binance_result):
        """롤백 실행"""
        
        if not isinstance(upbit_result, Exception):
            # Upbit만 체결 → 반대매매
            for attempt in range(self.MAX_RETRIES):
                try:
                    await self.upbit.create_market_sell_order(
                        symbol='BTC/KRW',
                        amount=upbit_result['filled']
                    )
                    print(f"✅ Upbit rollback success")
                    break
                except Exception as e:
                    print(f"⚠️ Upbit rollback attempt {attempt + 1} failed: {e}")
                    await asyncio.sleep(self.RETRY_DELAY)
        
        if not isinstance(binance_result, Exception):
            # Binance만 체결 → 포지션 청산
            for attempt in range(self.MAX_RETRIES):
                try:
                    await self.binance.create_market_buy_order(
                        symbol='BTC/USDT:USDT',
                        amount=binance_result['filled'],
                        params={'reduceOnly': True}
                    )
                    print(f"✅ Binance rollback success")
                    break
                except Exception as e:
                    print(f"⚠️ Binance rollback attempt {attempt + 1} failed: {e}")
                    await asyncio.sleep(self.RETRY_DELAY)
```

---

## 5. 자금 관리 리스크

### 5.1 자본 구조

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Capital Structure                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│   총 자본: 4,000만원                                                 │
│   ├── 예비비: 200만원 (5%)                                          │
│   │   └─ 용도: 수수료, 슬리피지, 긴급 자금                          │
│   │                                                                   │
│   └── 운용 자본: 3,800만원 (95%)                                    │
│       ├── Upbit (현물): 1,900만원 (50%)                             │
│       └── Binance (선물): 1,900만원 (50%)                           │
│                                                                       │
│   포지션 최대 자금:                                                  │
│   ├── Level 1: 1,520만원 (40%)                                      │
│   ├── Level 2: 2,280만원 (60%)                                      │
│   └── Combined: 3,800만원 (100%)                                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 자금 묶임 시나리오

```python
class CapitalRiskManager:
    """자금 리스크 관리자"""
    
    def calculate_capital_at_risk(self, position) -> dict:
        """묶인 자금 계산"""
        
        total_capital = 40_000_000
        invested = position.total_invested if position else 0
        available = total_capital - invested
        
        return {
            'total': total_capital,
            'invested': invested,
            'available': available,
            'invested_pct': (invested / total_capital) * 100,
            'available_pct': (available / total_capital) * 100
        }
    
    def estimate_holding_cost(self, position) -> dict:
        """보유 비용 추정"""
        
        holding_hours = self._get_holding_hours(position)
        
        # 펀딩비 (Binance 선물)
        # 8시간마다 발생, 평균 ±0.01%
        funding_periods = holding_hours / 8
        estimated_funding = position.binance_amount * 0.0001 * funding_periods
        
        # 기회비용 (연 5% 기준)
        opportunity_cost = position.total_invested * 0.05 * (holding_hours / 8760)
        
        return {
            'holding_hours': holding_hours,
            'estimated_funding_cost': estimated_funding,
            'opportunity_cost': opportunity_cost,
            'total_holding_cost': estimated_funding + opportunity_cost
        }
```

---

## 6. 리스크 대시보드

### 6.1 실시간 리스크 모니터링

```python
class RiskDashboard:
    """리스크 대시보드"""
    
    def __init__(self, fx_manager, position_monitor, capital_manager):
        self.fx = fx_manager
        self.position = position_monitor
        self.capital = capital_manager
    
    def get_overall_risk_status(self) -> dict:
        """종합 리스크 상태"""
        
        return {
            'timestamp': datetime.now().isoformat(),
            
            # 환율 리스크
            'fx_risk': {
                'level': self.fx.get_risk_level(),
                'is_entry_allowed': self.fx.is_entry_allowed(),
                'blocked_count_today': self.fx.blocked_count
            },
            
            # 포지션 리스크
            'position_risk': self.position.check_holding_risk() if self.position.position else None,
            
            # 자금 리스크
            'capital_risk': self.capital.calculate_capital_at_risk(
                self.position.position
            ),
            
            # 종합 상태
            'overall': self._calculate_overall_risk()
        }
    
    def _calculate_overall_risk(self) -> str:
        """종합 리스크 레벨 계산"""
        
        fx_level = self.fx.get_risk_level()
        
        if fx_level == 'HIGH':
            return 'HIGH'
        
        if self.position.position:
            pos_risk = self.position.check_holding_risk()
            if pos_risk['risk_level'] == 'CRITICAL':
                return 'HIGH'
            elif pos_risk['risk_level'] == 'WARNING':
                return 'MEDIUM'
        
        return 'LOW'
```

### 6.2 Telegram 리스크 알림

```python
async def send_risk_alert(bot, chat_id, risk_status: dict):
    """리스크 알림 전송"""
    
    overall = risk_status['overall']
    
    if overall == 'HIGH':
        emoji = "🔴"
    elif overall == 'MEDIUM':
        emoji = "🟡"
    else:
        emoji = "🟢"
    
    text = f"{emoji} *리스크 상태: {overall}*\n\n"
    
    # 환율 리스크
    fx = risk_status['fx_risk']
    text += f"📊 *환율*: {fx['level']}\n"
    text += f"   진입 가능: {'✅' if fx['is_entry_allowed'] else '⛔'}\n"
    
    # 포지션 리스크
    if risk_status['position_risk']:
        pos = risk_status['position_risk']
        text += f"\n📈 *포지션*: {pos['risk_level']}\n"
        text += f"   보유 시간: {pos['holding_hours']:.1f}시간\n"
    
    # 자금 리스크
    cap = risk_status['capital_risk']
    text += f"\n💰 *자금*\n"
    text += f"   투입: {cap['invested']:,.0f}원 ({cap['invested_pct']:.1f}%)\n"
    text += f"   가용: {cap['available']:,.0f}원 ({cap['available_pct']:.1f}%)\n"
    
    await bot.send_message(chat_id, text, parse_mode='Markdown')
```

---

## 7. 리스크 파라미터 설정

```yaml
# config/risk_config.yaml
risk_management:
  version: "3.0"
  
  # 환율 필터
  fx_filter:
    enabled: true
    ma_period_minutes: 720  # 12시간
    surge_threshold: 1.001  # +0.1%
  
  # 포지션 리스크
  position:
    holding_warning_hours: 24
    holding_critical_hours: 72
    drawdown_warning_pct: 1.0
    drawdown_critical_pct: 2.0
  
  # 실행 리스크
  execution:
    max_retries: 3
    retry_delay_seconds: 1.0
    rollback_enabled: true
  
  # 자금 리스크
  capital:
    reserve_ratio: 0.05
    max_position_ratio: 0.95
  
  # 알림
  alerts:
    telegram_enabled: true
    risk_check_interval_seconds: 60
```

---

**버전**: 3.0  
**작성일**: 2025-12-14  
**레포**: trading-platform-portfolio
