# Daylog — 기술 심층 분석 및 엔지니어링 케이스스터디 (Technical Deep Dive)

본 문서는 Daylog(실시간 일일 마이크로 브리핑 보드)를 개발하며 직면했던 핵심 엔지니어링 문제와, 이를 공학적 근거와 수치로 해결한 과정을 정리한 심층 기술 케이스스터디입니다.

---

## 📌 목차
1. [Deep Dive 1] TBT 93% 단축: WebGL 렌더링 병목 제거와 Map 기반 O(n log n) 알고리즘 개선
2. [Deep Dive 2] 프론트엔드 복원력(Resilience): 장애 5종 분류 모델과 RFC 5861 Stale-If-Error 게이트웨이
3. [Deep Dive 3] 3D 라이브러리(Three.js ~590KB) 폐기와 데이터 가독성 중심의 UX 피벗 (C-P-A-D-I)
4. [Deep Dive 4] SSR-CSR Hydration Mismatch 해결을 위한 `useSyncExternalStore` 동기화 패턴

---

## 🚀 Deep Dive 1. TBT 93% 단축: WebGL 렌더링 병목 제거와 Map 기반 O(n log n) 알고리즘 개선

### 1.1 배경 및 문제 상황 (Context & Problem)
Daylog는 사용자가 매일 접속하지 않더라도 과거 8월 1일부터의 데이터를 끊김 없이 조회할 수 있도록 **과거 원자료 백필(Backfill)** 기능을 제공합니다.
하지만 프로덕션 빌드 후 모바일 저사양 환경(Pixel 2 에뮬레이션)에서 Lighthouse 측정을 수행했을 때 심각한 메인 스레드 프리징이 관측되었습니다:
- **Lighthouse Performance Score:** 69점
- **Total Blocking Time (TBT):** **2,130ms** (적색 경고, 50ms 초과 Long Task 다수 발생)
- **First Contentful Paint (FCP):** 2.4s

### 1.2 원인 분석: 1차 렌더링 병목과 2차 알고리즘 병목의 결합
Chrome DevTools Performance 탭으로 CPU 화염도(Flamegraph)를 프로파일링한 결과, 메인 스레드 2초 점유는 두 가지 요인이 중첩되어 있었습니다:

1. **1차 병목 (초기 렌더링 블로킹):**  
   초기 프로토타입에 포함된 Three.js 라이브러리의 WebGL 컨텍스트 초기화, 지형 셰이더 컴파일, 3D 캔버스 렌더링이 모바일 메인 스레드를 1.5초 이상 독점.
2. **2차 병목 (시계열 데이터 처리 병목):**  
   8월 1일부터 누적된 수백 건의 과거 원자료를 기존 일일 기록과 병합하는 과정에서, 매 항목마다 배열 전체를 `findIndex()`로 순회하고 루프 내부에서 `sort()`를 반복 호출하는 **O(n²)** 연산으로 인해 V8 GC(Garbage Collection) 스파이크 발생:
   ```typescript
   // ❌ 기존 병목 코드: O(n²) 알고리즘 + 반복 정렬
   export function mergeDailyRecords(existing: DailyRecord[], incoming: DailyRecord[]): DailyRecord[] {
     let result = [...existing];
     for (const item of incoming) {
       const index = result.findIndex(r => r.date === item.date && r.widgetId === item.widgetId);
       if (index !== -1) result[index] = item;
       else result.push(item);
       result.sort((a, b) => a.date.localeCompare(b.date)); // ⚠️ 루프마다 O(n log n) 정렬 반복
     }
     return result;
   }
   ```

### 1.3 해결책 및 최적화 파이프라인 (Solution)
1. **Three.js 전격 폐기 및 경량 SVG 전환:**  
   초기 로드 시 WebGL 컨텍스트 생성 비용을 0으로 만들어 1.5초 이상의 초기 메인 스레드 블로킹 해소.
2. **해시 맵(Map) 기반 O(n log n) 파이프라인 개편:**  
   복합키(`date:widgetId`)를 활용해 삽입과 수정을 **O(1)**로 단축하고, 정렬은 데이터 정리가 완전히 끝난 후 단 한 번만 실행:
   ```typescript
   // ✅ 개선된 코드: O(n log n) 최적화
   export function mergeDailyRecordsOptimized(
     existing: DailyRecord[],
     incoming: DailyRecord[]
   ): DailyRecord[] {
     const recordMap = new Map<string, DailyRecord>();
     for (let i = 0; i < existing.length; i++) {
       recordMap.set(`${existing[i].date}:${existing[i].widgetId}`, existing[i]);
     }
     for (let j = 0; j < incoming.length; j++) {
       recordMap.set(`${incoming[j].date}:${incoming[j].widgetId}`, incoming[j]);
     }
     return Array.from(recordMap.values()).sort((a, b) => a.date.localeCompare(b.date));
   }
   ```

### 1.4 결과 및 정량적 성과 (Results & Impact)
| 지표 | 개선 전 | 개선 후 | 개선율 |
|---|---|---|---|
| **Total Blocking Time (TBT)** | 2,130ms | **140ms** | **93.4% 단축** ⚡ |
| **Lighthouse Performance** | 69점 | **98점** | **+29점 상승** 🚀 |
| **3,000건 대용량 데이터 연산** | ~850ms | **14ms** | **98.3% 속도 향상** |
| **Cumulative Layout Shift (CLS)**| 0.09 | **0.01** | 레이아웃 시프트 소멸 |

*회귀 방지:* 3,000건 대용량 병합이 250ms 이내에 완료되는지 검증하는 단위 벤치마크 테스트 코드를 Vitest에 영구 반영했습니다.

---

## 🛡️ Deep Dive 2. 프론트엔드 복원력(Resilience): 장애 5종 분류 모델과 RFC 5861 Stale-If-Error 게이트웨이

### 2.1 배경 및 문제 상황 (Context & Problem)
대시보드 애플리케이션의 숙명은 불안정한 다수의 서드파티 API에 종속된다는 점입니다:
- 기상청·에어코리아: 정기 서버 점검 시 500/503 반환
- 업비트: 빈번한 네트워크 지연 및 Rate Limit(429)
- 한국수출입은행: 도메인 만료 및 주말/공휴일 고시 부재로 인한 빈 응답
- 만약 하나의 위젯이라도 예외 처리가 미비하면, React 렌더링 트리가 언마운트되거나 대시보드 전체가 빈 화면(White-out)이 되는 치명적인 문제가 발생합니다.

### 2.2 해결 아키텍처: Resilience Gateway & RFC 5861 Stale-If-Error 전략
마틴 파울러의 무거운 서킷 브레이커 상태 머신을 서버리스 환경에 억지로 도입하는 대신, **RFC 5861 기반의 Stale-If-Error 전략**과 **Next.js BFF 라우트 핸들러를 결합한 경량 복원력 게이트웨이**를 구축했습니다.

#### 1. 장애 5종 정밀 분류 (Failure Taxonomy)
오류를 뭉뚱그려 "알 수 없는 에러"로 표기하지 않고 5가지 구체적 도메인 상태로 분류:
1. `timeout`: 3,500ms 제한 + 1회 백오프(최대 7.2초 이내) 초과 시 Abort 및 시간 초과 안내
2. `auth_failed`: 401/403 응답 시 API 키 만료 안내 및 오픈소스 환경에서의 대체 공급자 안내
3. `rate_limited`: 429 응답 시 일일 호출 제한 도달 안내 및 리셋 시각 표기
4. `offline`: DNS 실패 또는 네트워크 단절 안내
5. `schema_changed`: 필수 필드가 결손된 비정상 JSON 수신 시 파싱 방어

#### 2. Stale-While-Revalidate & Last Known Good 보존
- 외부 API가 에러를 반환하더라도 이전에 수신했던 마지막 정상 데이터(`lastGoodValue`)가 존재한다면 카드를 삭제하지 않습니다.
- 시각적으로 노란색 점과 함께 **"오래된 데이터 (14:30 기준)"** 라벨을 부착하여 사용자가 과거 정보임을 인지한 채로 계속 참고할 수 있도록 합니다.

#### 3. 런타임 Fallback Chain (동적 장애 극복)
- 수출입은행 공식 환율 API가 실패하거나 인증키가 없는 경우, 애플리케이션은 즉시 유럽중앙은행 기반의 `Frankfurter` 공개 API로 백업 전환합니다.
- 전환 사실을 사용자에게 숨기지 않고, 출처 뱃지에 **"(대체 공급자) ECB 참조환율"**임을 명확히 표시하여 정보의 투명성을 유지합니다.

---

## 🎨 Deep Dive 3. 3D 라이브러리(Three.js ~590KB) 폐기와 데이터 가독성 중심의 UX 피벗 (C-P-A-D-I)

### 3.1 Context & Problem (문제 정의)
초기 프로토타입 단계에서는 포트폴리오의 시각적 화려함을 극대화하기 위해 `Three.js`를 사용한 3차원 입체 바 차트(3D Bar Chart)를 구현했습니다.
그러나 실제 사용자 테스트 결과:
1. **오클루전(Occlusion)과 왜곡:** 앞 열의 막대가 뒤 열의 막대를 가려 특정 날짜의 데이터를 확인하려면 마우스로 3D 궤도를 계속 회전시켜야 하는 인지 피로 발생.
2. **원근 왜곡:** 동일한 수치임에도 뒤쪽 막대가 더 작아 보이는 치명적인 데이터 왜곡 발생.
3. **Fitts's Law 관점의 조작성 악화:** 모바일 터치 환경에서 좁은 3D 오브젝트를 터치하기가 매우 어렵고 오작동 빈발.
4. **번들 및 GPU 오버헤드:** Three.js 추가로 번들이 ~590KB 증가하고, 모바일 스크롤 시 프레임 드랍(Jank) 발생.

### 3.2 Alternatives & Decision (대안 비교 및 의사결정)
- *대안 A (Three.js 인스턴싱 최적화):* 렌더링 성능은 일부 개선되나 조작 피로도와 원근 왜곡은 근본적으로 해결 불가.
- *대안 B (경량 2D HTML Canvas):* 렌더링은 빠르나 DOM 요소가 아니므로 스크린 리더 접근성(a11y) 트리 접근 불가.
- *대안 C (경량 SVG 기반 달력 히트맵 & 밴드 차트):* 5KB 미만의 초경량, 완전한 DOM 기반 웹 접근성 충족, 한눈에 들어오는 2D 직관적 패턴.

**Decision:** 대안 C 채택. "정보 전달 도구는 예술품이 아니라 신뢰할 수 있는 계측기여야 한다"는 원칙 확립.

---

## ⚡ Deep Dive 4. SSR-CSR Hydration Mismatch 해결을 위한 `useSyncExternalStore` 동기화

### 4.1 문제 상황과 "57년 후" 안티 패턴 방어
Daylog의 위젯 카드는 "3분 전", "방금 전"과 같은 상대 시각 표기를 제공합니다.
서버 SSR 시점에 상대 시각을 계산할 때 기준 시각을 0(Unix Epoch)으로 잘못 넘기면 `dayjs("2026-11-10").from(0)`이 연산되어 서버 HTML에 **"57년 후"**라는 기괴한 텍스트가 박혀 내려오거나, React 418 Hydration 불일치 오류가 발생합니다.

### 4.2 해결: React 18의 `useSyncExternalStore` 도입 및 절대시각 렌더링
단순히 `useEffect`로 깜빡임을 감추는 편법 대신, React 18의 `useSyncExternalStore`를 채택하여 서버 스냅샷과 클라이언트 스토어를 엄격하게 분리했습니다:

```typescript
// 서버 SSR 시점에는 '14:30' 절대 시각을 안전하게 렌더링하고,
// 클라이언트 마운트 완료 후에만 타이머 스토어를 구독하여 상대 시각('3분 전')으로 전환
function TimeValue({ label, value, now }: { label: string; value: string; now: number }) {
  const isHydrated = now > 0;
  const displayText = !value 
    ? "API 미제공" 
    : isHydrated 
      ? formatRelativeTime(value, now) 
      : formatTimeOnly(value); // SSR 시점 안전한 절대 시각

  return <dd title={formatAbsoluteTime(value)}>{displayText}</dd>;
}
```

### 4.3 효과
- 초기 SSR 마운트 시 서버와 클라이언트의 DOM 트리가 100% 일치하여 Hydration Mismatch 경고 **0건** 달성.
- 화면 깜빡임(FOUC) 없이, 마운트 직후 클라이언트 구독에 의해 자연스럽고 일관된 상대 시각 갱신 보장.
