# 🎨 Design System Reference

**Design System:** Kinetic Minimalism  
**Master Location:** [storage/docs/DESIGN_SYSTEM.md](https://github.com/vsun410/trading-platform-storage/blob/main/docs/DESIGN_SYSTEM.md)

> ⚠️ 이 문서는 요약본입니다. 전체 디자인 시스템은 **storage 레포**를 참조하세요.

---

## Portfolio 레포 UI 역할

| 기능 | 설명 |
|------|------|
| 리스크 대시보드 | Sharpe, MDD, VaR 지표 |
| 상관관계 히트맵 | 전략 간 상관계수 시각화 |
| 자본 배분 차트 | 최적 비중 파이/바 차트 |

---

## 핵심 디자인 토큰 (Quick Reference)

### 색상

```css
/* 중립 */
--bg-primary: #FFFFFF;
--bg-secondary: #F8F9FA;
--text-primary: #0A0A0B;
--text-secondary: #5F6368;

/* 액센트 */
--accent-primary: #0066FF;
--color-long: #00D4AA;    /* 양의 상관, 수익 */
--color-short: #FF3366;   /* 음의 상관, 손실 */
--color-warning: #FFB800; /* 경고 */
```

### 그림자 (방향성: 우하단)

```css
--shadow-md: 4px 8px 16px rgba(0, 0, 0, 0.08);
```

---

## Portfolio UI 컴포넌트

### 1. 리스크 지표 카드

```tsx
<div className="
  relative bg-white rounded-xl p-6
  shadow-[4px_8px_16px_rgba(0,0,0,0.08)]
">
  {/* Diagonal corner accent */}
  <div className="absolute top-0 right-0 w-16 h-16 
    bg-gradient-to-bl from-[#E6F0FF] to-transparent"
    style={{ clipPath: 'polygon(100% 0, 100% 100%, 0 0)' }}
  />
  
  <p className="text-xs text-[#9AA0A6] uppercase tracking-wider">Sharpe Ratio</p>
  <p className="text-3xl font-bold font-mono text-[#0066FF]">1.85</p>
  
  {/* Bottom accent bar */}
  <div className="absolute bottom-0 left-6 w-16 h-1 bg-[#0066FF] -skew-x-12" />
</div>
```

### 2. 상관관계 히트맵 셀

```tsx
{/* 양의 상관 (0.7+) */}
<div className="w-12 h-12 bg-[#FF3366] text-white text-xs font-mono 
  flex items-center justify-center">0.85</div>

{/* 낮은 상관 (-0.3 ~ 0.3) - 좋음 */}
<div className="w-12 h-12 bg-[#00D4AA] text-white text-xs font-mono 
  flex items-center justify-center">0.12</div>

{/* 음의 상관 - 헤지 효과 */}
<div className="w-12 h-12 bg-[#0066FF] text-white text-xs font-mono 
  flex items-center justify-center">-0.45</div>
```

### 3. 자본 배분 권고 카드

```tsx
<div className="
  flex items-center gap-4 p-4
  bg-white rounded-lg
  shadow-[4px_8px_16px_rgba(0,0,0,0.08)]
  border-l-4 border-[#0066FF]
">
  <div className="flex-1">
    <p className="font-semibold">김프 전략</p>
    <p className="text-sm text-[#5F6368]">현재 40% → 권장 55%</p>
  </div>
  <span className="
    px-3 py-1 bg-[#E6F0FF] text-[#0066FF]
    text-sm font-semibold rounded
    -skew-x-6
  ">
    +15%
  </span>
</div>
```

---

## 체크리스트

- [ ] 방향성 요소 1개 이상 포함
- [ ] 그림자 단일 방향 (우하단)
- [ ] 색상: 중립 90% + 액센트 10%
- [ ] 글래스/뉴모피즘 요소 없음
- [ ] 숫자는 Monospace 폰트

---

*전체 가이드: [storage/docs/DESIGN_SYSTEM.md](https://github.com/vsun410/trading-platform-storage/blob/main/docs/DESIGN_SYSTEM.md)*
