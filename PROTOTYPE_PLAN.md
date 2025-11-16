# 토큰 게임 프로토타이핑 계획서

## 🎯 게임 컨셉 요약
프롬프트 엔지니어가 되어 토큰 카드를 조합하여 AI 파트너의 추론을 이끌어내는 덱빌딩 게임

---

## 🛠 기술 스택 제안

### Option A: 웹 기반 프로토타입 (추천)
- **프론트엔드**: React + TypeScript
- **상태 관리**: Zustand (가벼운 상태 관리)
- **UI**: Tailwind CSS + Framer Motion (카드 애니메이션)
- **빌드**: Vite
- **장점**: 빠른 프로토타이핑, 쉬운 배포, 크로스 플랫폼

### Option B: 게임 엔진
- **Unity** + C# 또는 **Godot** + GDScript
- **장점**: 풍부한 게임 기능, 애니메이션 도구
- **단점**: 초기 설정 복잡, 프로토타이핑 속도 느림

### 권장사항
**웹 기반 프로토타입**으로 시작하여 핵심 메커니즘 검증 후 필요시 게임 엔진으로 이전

---

## 📋 Phase별 구현 계획

### **Phase 1: 핵심 데이터 구조 설계** (1-2일)

#### 1.1 토큰 카드 시스템
```typescript
interface Token {
  id: string;
  type: 'target' | 'action' | 'modifier';
  grade: 'A' | 'B' | 'C';  // A: 명확, C: 모호
  text: string;            // 예: "적 1체", "피해"
  cost: number;            // 추론 필요량

  // C등급 전용
  ambiguity?: {
    interpretations: Array<{
      result: string;
      baseChance: number;  // 기본 확률
    }>;
  };
}
```

#### 1.2 프롬프트 & 추론 시스템
```typescript
interface Prompt {
  tokens: Token[];
  totalCost: number;
}

interface ReasoningResult {
  type: 'surge' | 'perfect' | 'cut';
  usedTokens: Token[];
  cutTokens?: Token[];      // Cut인 경우
  bonus?: string;            // Surge/Perfect 보너스
}
```

#### 1.3 덱 & 핸드 시스템
```typescript
interface GameState {
  library: Token[];         // 라이브러리 (덱)
  hand: Token[];            // 액티브 슬롯 (5장 고정)
  discardPile: Token[];     // 버린 덱
  reasoningPool: number;    // 매 턴 12 (고정)
}
```

---

### **Phase 2: 기본 UI 프레임워크** (2-3일)

#### 2.1 레이아웃 구성
```
┌─────────────────────────────────────┐
│  추론 풀: 12 / 12     턴: 1         │
├─────────────────────────────────────┤
│  [적 체력바] [적 체력바] [적 체력바] │
├─────────────────────────────────────┤
│  ┌─────┐ ┌─────┐ ┌─────┐           │
│  │슬롯1│ │슬롯2│ │슬롯3│  출력 슬롯│
│  └─────┘ └─────┘ └─────┘           │
├─────────────────────────────────────┤
│  ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐   │
│  │ 1 │ │ 2 │ │ 3 │ │ 4 │ │ 5 │   │
│  └───┘ └───┘ └───┘ └───┘ └───┘   │
│         액티브 핸드 (5장)           │
└─────────────────────────────────────┘
```

#### 2.2 카드 컴포넌트
- 드래그 앤 드롭 기능
- 등급별 시각적 구분 (A: 파랑, C: 주황)
- 비용(추론 필요량) 표시

---

### **Phase 3: 회전식 핸드 시스템** (2일)

#### 핵심 로직
```typescript
function endTurn(state: GameState, usedTokens: Token[]) {
  // 1. 사용한 카드 → 버린 덱
  state.discardPile.push(...usedTokens);

  // 2. 핸드에서 제거
  state.hand = state.hand.filter(t => !usedTokens.includes(t));

  // 3. 빈 슬롯만큼 보충
  const drawCount = 5 - state.hand.length;
  const drawn = state.library.splice(0, drawCount);

  // 4. 덱이 부족하면 버린 덱 셔플 후 재활용
  if (drawn.length < drawCount && state.discardPile.length > 0) {
    shuffle(state.discardPile);
    state.library.push(...state.discardPile);
    state.discardPile = [];

    const remaining = drawCount - drawn.length;
    drawn.push(...state.library.splice(0, remaining));
  }

  state.hand.push(...drawn);
}
```

#### 테스트 시나리오
- [ ] 5장 핸드에서 3장 사용 → 2장 유지 + 3장 보충
- [ ] 덱 소진 시 버린 덱 자동 셔플
- [ ] 연속 턴에서 핸드 상태 유지

---

### **Phase 4: 추론 시스템 (Surge/Perfect/Cut)** (3일)

#### 4.1 추론 판정 로직
```typescript
function evaluateReasoning(
  prompt: Prompt,
  reasoningPool: number
): ReasoningResult {
  const diff = reasoningPool - prompt.totalCost;

  if (diff > 0) {
    // 과추론 (Surge)
    return {
      type: 'surge',
      usedTokens: prompt.tokens,
      bonus: 'damage +30%'  // 서지 프로토콜에서 가져옴
    };
  } else if (diff === 0) {
    // 완전 추론 (Perfect)
    return {
      type: 'perfect',
      usedTokens: prompt.tokens,
      bonus: 'effect x2'
    };
  } else {
    // 추론 실패 (Cut)
    const cutCount = Math.ceil(Math.abs(diff) / 평균비용);
    return {
      type: 'cut',
      usedTokens: prompt.tokens.slice(0, -cutCount),
      cutTokens: prompt.tokens.slice(-cutCount)
    };
  }
}
```

#### 4.2 서지 프로토콜 시스템
- 플레이어가 게임 시작 시 선택
- 예시: "공격형", "방어형", "드로우형"

#### 테스트 시나리오
- [ ] 비용 9, 풀 12 → Surge (+30% 피해)
- [ ] 비용 12, 풀 12 → Perfect (효과 2회)
- [ ] 비용 15, 풀 12 → Cut (뒤에서 1~2장 제거)

---

### **Phase 5: 출력 슬롯 시스템 (Split/Merge)** (2일)

#### 5.1 분리 (Splitting)
```typescript
interface OutputSlot {
  id: number;
  tokens: Token[];
  executionOrder?: number;  // 플레이어가 지정
}

// 예: 슬롯1(방어) → 슬롯2(공격) 순서로 실행
```

#### 5.2 융합 (Merging)
```typescript
function mergeSlots(slots: OutputSlot[]): ComboCard {
  return {
    tokens: slots.flatMap(s => s.tokens),
    bonuses: ['effect x2'],  // Perfect과 시너지
    power: calculateComboPower(slots)
  };
}
```

#### UI 기능
- 슬롯별 드래그 앤 드롭
- 실행 순서 조정 버튼
- 분리/융합 모드 전환 토글

---

### **Phase 6: C등급 카드 & 파트너 인격** (3-4일)

#### 6.1 모호함 해석 시스템
```typescript
interface PartnerPersonality {
  name: string;
  modifiers: Array<{
    tokenGrade: 'C';
    tokenText: string;      // "뜨거움"
    interpretation: string; // "공격 버프"
    chanceBoost: number;    // +20% → 기본 75%에서 95%로
  }>;
}

// 해석 기록 추적
interface InterpretationLog {
  tokenId: string;
  history: Array<{
    timestamp: number;
    result: string;
  }>;
}
```

#### 6.2 UI - 관측 데이터 표시
```
┌─────────────────────────────┐
│ [C: 뜨거움]                 │
│ ━━━━━━━━━━━━━━━━━━━━━━━━  │
│ 🔥 공격 버프: 19/20 (95%)   │
│ ❄️  방어 버프: 1/20 (5%)    │
│                             │
│ 파트너: [보조적 해석]        │
└─────────────────────────────┘
```

#### 테스트 시나리오
- [ ] C등급 카드 사용 시 확률 기반 해석
- [ ] 파트너 인격 변경 시 확률 테이블 갱신
- [ ] 20회 이상 사용 후 통계 UI 표시

---

## 🧪 프로토타입 검증 체크리스트

### 핵심 메커니즘
- [ ] 회전식 핸드가 "피로도"와 "단조로움"을 동시에 해결하는가?
- [ ] Surge/Perfect/Cut 선택이 전략적 깊이를 만드는가?
- [ ] C등급 카드가 "연구의 재미"를 제공하는가?

### 밸런싱
- [ ] 추론 풀 12가 적절한 제약인가?
- [ ] A등급과 C등급의 리스크/리턴이 균형잡혀있는가?
- [ ] Split vs Merge 선택에 의미있는 차이가 있는가?

### 재미 요소
- [ ] 카드 조합의 발견이 즐거운가?
- [ ] AI 파트너의 "해석" 과정이 흥미로운가?
- [ ] 턴마다 의미있는 선택을 하는가?

---

## 📅 예상 일정

| Phase | 작업 | 예상 소요 |
|-------|------|----------|
| 1 | 데이터 구조 설계 | 1-2일 |
| 2 | UI 프레임워크 | 2-3일 |
| 3 | 회전식 핸드 | 2일 |
| 4 | 추론 시스템 | 3일 |
| 5 | 출력 슬롯 | 2일 |
| 6 | C등급 & 파트너 | 3-4일 |
| **총계** | | **13-16일** |

---

## 🎮 초기 카드 풀 (프로토타입용)

### A등급 (명확)
- `[A: 적 1체]` (비용 1)
- `[A: 적 전체]` (비용 3)
- `[A: 피해 3]` (비용 2)
- `[A: 방어 5]` (비용 2)

### C등급 (모호)
- `[C: 모두에게]` (비용 3)
  - 90%: 적 전체
  - 10%: 아군 포함 전체

- `[C: 뜨거움]` (비용 2)
  - 75%: 공격 버프 +2
  - 25%: 방어 버프 +2

- `[C: 강하게]` (비용 2)
  - 60%: 피해 +3
  - 30%: 방어 +3
  - 10%: 자신에게 피해 2

---

## 🚀 다음 단계

1. **기술 스택 선택** - React + TypeScript 웹 프로토타입으로 진행할까요?
2. **Phase 1 시작** - 데이터 구조부터 구현 시작
3. **간단한 전투 시스템** 추가 (적 1체, HP 시스템)

어떤 부분부터 시작하시겠습니까?
