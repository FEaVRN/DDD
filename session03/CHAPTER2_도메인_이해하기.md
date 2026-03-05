# CHAPTER 2: 도메인 이해하기 (Understanding the Domain)

본 문서는 "도메인 주도 설계를 위한 함수형 프로그래밍" CHAPTER 2 스터디 내용을 정리한 문서입니다.

> **주의:** 본 문서에 포함된 TypeScript 코드는 설명을 위한 예제이며, 완전한 실행 코드가 아닙니다.

## 1. 도메인 전문가 인터뷰하기
*   **보편적 언어(Ubiquitous Language):** 개발자와 도메인 전문가가 공통으로 사용하는 언어를 구축해야 합니다.
*   **질문의 기술:** 기술적인 솔루션이 아니라 비즈니스 프로세스의 목적과 결과에 집중하여 질문합니다.

## 2. 비기능적 요구사항 이해하기
*   시스템의 성능, 보안, 확장성 등 비즈니스 로직 외적인 요구사항을 파악합니다.
*   FP(Functional Programing)의 불변성과 순수 함수는 이러한 요구사항(특히 병렬 처리)을 해결하는 데 도움을 줍니다.

## 3. 작업 흐름(Workflow) 중심 사고
*   **입력과 출력:** 각 작업 흐름이 무엇을 입력받아 무엇을 출력하는지 명확히 정의합니다.
*   **데이터베이스 중심 디자인 지양:** 저장 방식(DB)을 먼저 고민하지 말고 비즈니스 로직 자체에 집중합니다.
*   **클래스 중심 디자인 지양:** 복잡한 상속 구조보다 데이터와 행위의 명확한 분리를 지향합니다.

## 4. 도메인 모델링의 핵심
*   **제약 사항 표현:** 비즈니스 규칙을 타입 시스템에 녹여내어 '유효하지 않은 상태'가 표현 불가능하도록 설계합니다. (Make Illegal States Unrepresentable)
*   **주문의 생애 주기:** 주문이 각 단계를 거치며 어떻게 상태가 변하는지(예: 견적 주문 -> 확정 주문 -> 배송 주문)를 추적합니다.

## 5. 주문 접수 작업 흐름 예시
*   **단계별 구체화:**
    1.  주문서 수신
    2.  유효성 검사 (Validation)
    3.  가격 계산 (Pricing)
    4.  결과 생성 및 이벤트 발생

## 6. 도메인 문서화 (Domain Documentation)

도메인을 이해한 후, 이를 어떻게 **문서화**할 것인가가 중요합니다. 전통적인 OOP 설계와 함수형 설계의 문서화 방식은 근본적으로 다릅니다.

### OOP 설계의 문서화: UML과 Flow Chart

**전통적인 OOP 접근:**

1. **UML 클래스 다이어그램으로 설계**
   ```
   ┌─────────────────────────────┐
   │       Order                 │
   ├─────────────────────────────┤
   │ - orderId: String           │
   │ - customer: Customer        │
   │ - items: List<OrderItem>    │
   │ - status: OrderStatus       │
   │ - totalPrice: Double        │
   ├─────────────────────────────┤
   │ + submitOrder()             │
   │ + validateOrder(): boolean  │
   │ + calculatePrice()          │
   │ + completeOrder()           │
   │ + cancelOrder()             │
   └─────────────────────────────┘
   ```

2. **상태 다이어그램으로 상태 전환 표현**
   ```
   [Created] ──submitOrder()──> [Submitted]
                                    │
                            validateOrder()
                                    │
                                    ▼
                              [Validated]
                                    │
                          calculatePrice()
                                    │
                                    ▼
                              [Priced] ──completeOrder()──> [Completed]
                                    │
                                    └──cancelOrder()──> [Cancelled]
   ```

3. **Flow Chart로 비즈니스 프로세스 표현**
   ```
   [Start]
      │
      ▼
   주문 수신
      │
      ▼
   검증 ◇ ──NO──> [Reject]
      │
     YES
      │
      ▼
   가격 계산
      │
      ▼
   결제 처리 ◇ ──FAIL──> [Payment Failed]
      │
    SUCCESS
      │
      ▼
   주문 확정
      │
      ▼
   [Complete]
   ```

**문제점:**
- 📊 다이어그램은 시각적으로 보기 좋지만, **실제 코드와 동기화되지 않음**
- 🔄 코드가 변경되면 다이어그램을 수동으로 업데이트해야 함
- 📝 상태 간 유효성 검사 규칙이 명확하지 않음
- ❓ 어떤 필드가 어떤 상태에서 유효한지 다이어그램에 표현하기 어려움

### 함수형 설계의 문서화: 타입 정의가 곧 문서

**함수형 접근:**

타입 정의 자체가 가장 정확한 문서입니다. 코드가 문서이고, 문서가 곧 검증 가능한 코드입니다.

```plaintext
/**
 * 주문의 생애주기
 * 
 * 주문은 다음과 같은 단계를 거칩니다:
 * 1. Created: 새로운 주문이 생성됨
 * 2. Submitted: 고객이 주문을 제출함
 * 3. Validated: 재고와 가격이 검증됨
 * 4. Confirmed: 결제가 완료되고 주문이 확정됨
 * 5. Shipped: 주문이 배송됨
 * 6. Cancelled: 주문이 취소됨
 */

// 1단계: 새로 생성된 주문 (최소 정보만 포함)
type CreatedOrder = {
  orderId: string;
  customerId: string;
  items: OrderItem[];      // 어떤 상품을 주문했는가?
  submittedAt: Date;
};

// 2단계: 제출된 주문 (유효성은 아직 확인 안 함)
type SubmittedOrder = CreatedOrder & {
  submittedAt: Date;
};

// 3단계: 검증된 주문 (재고 확인됨, 가격 계산됨)
type ValidatedOrder = SubmittedOrder & {
  validatedAt: Date;
  totalPrice: number;      // 가격이 계산됨
  availableInventory: true; // 재고가 충분함이 보장됨
};

// 4단계: 확정된 주문 (결제 완료)
type ConfirmedOrder = ValidatedOrder & {
  confirmedAt: Date;
  paymentId: string;       // 결제 ID가 있음
  paymentStatus: 'PAID';   // 결제 상태가 명확함
};

// 5단계: 배송된 주문
type ShippedOrder = ConfirmedOrder & {
  shippedAt: Date;
  trackingNumber: string;  // 배송 추적 번호가 있음
};

// 6단계: 취소된 주문
type CancelledOrder = CreatedOrder & {
  cancelledAt: Date;
  cancellationReason: string;
  refundAmount: number;
};

// 합타입으로 모든 가능한 상태를 표현
type Order = 
  | { status: 'Created'; data: CreatedOrder }
  | { status: 'Submitted'; data: SubmittedOrder }
  | { status: 'Validated'; data: ValidatedOrder }
  | { status: 'Confirmed'; data: ConfirmedOrder }
  | { status: 'Shipped'; data: ShippedOrder }
  | { status: 'Cancelled'; data: CancelledOrder };

// 상태 전환 함수들이 문서화와 동시에 검증을 제공
/**
 * 주문을 제출합니다.
 * 
 * 전제조건: Created 상태의 주문
 * 결과: Submitted 상태로 변환
 * 
 * @param order - Created 상태의 주문 (타입으로 강제됨)
 * @returns Submitted 상태의 주문
 */
const submitOrder = (order: CreatedOrder): SubmittedOrder => ({
  ...order,
  submittedAt: new Date()
});

/**
 * 주문을 검증합니다.
 * 
 * 검증 항목:
 * - 상품이 재고에 있는가?
 * - 가격 계산이 정확한가?
 * 
 * 전제조건: Submitted 상태의 주문
 * 결과: Validated 상태로 변환 또는 검증 실패
 * 
 * @param order - Submitted 상태의 주문
 * @param inventory - 재고 시스템
 * @returns 성공 시 Validated 상태, 실패 시 Error
 */
const validateOrder = (
  order: SubmittedOrder,
  inventory: InventoryService
): Promise<Result<ValidatedOrder, ValidationError>> => {
  // 재고 확인
  const hasInventory = inventory.check(order.items);
  if (!hasInventory) {
    return Err(new ValidationError('재고 부족'));
  }

  // 가격 계산
  const totalPrice = order.items.reduce((sum, item) => 
    sum + (item.price * item.quantity), 0
  );

  return Ok({
    ...order,
    validatedAt: new Date(),
    totalPrice,
    availableInventory: true
  });
};

/**
 * 주문을 확정합니다.
 * 
 * 결제를 처리하고 주문을 확정합니다.
 * 
 * 전제조건: Validated 상태의 주문
 * 결과: Confirmed 상태로 변환 또는 결제 실패
 */
const confirmOrder = async (
  order: ValidatedOrder,
  paymentService: PaymentService
): Promise<Result<ConfirmedOrder, PaymentError>> => {
  const paymentResult = await paymentService.charge(
    order.customerId,
    order.totalPrice
  );

  if (!paymentResult.success) {
    return Err(new PaymentError('결제 실패'));
  }

  return Ok({
    ...order,
    confirmedAt: new Date(),
    paymentId: paymentResult.transactionId,
    paymentStatus: 'PAID'
  });
};

/**
 * 주문을 배송합니다.
 */
const shipOrder = (
  order: ConfirmedOrder,
  shippingService: ShippingService
): Promise<ShippedOrder> => {
  const tracking = shippingService.create(order);
  
  return {
    ...order,
    shippedAt: new Date(),
    trackingNumber: tracking.number
  };
};

/**
 * 주문을 취소합니다.
 * 
 * Created 상태에서만 취소 가능합니다.
 * (이미 Submitted된 이후에는 취소 불가)
 */
const cancelOrder = (
  order: CreatedOrder,
  reason: string
): CancelledOrder => ({
  orderId: order.orderId,
  customerId: order.customerId,
  items: order.items,
  submittedAt: order.submittedAt,
  cancelledAt: new Date(),
  cancellationReason: reason,
  refundAmount: 0  // 아직 결제 안 했으므로 환불 없음
});
```

**함수형 문서화의 장점:**

- ✅ **코드 = 문서**: 타입 정의가 명확한 상태를 강제
- ✅ **자동 동기화**: 코드가 변경되면 타입도 변경되어야 함
- ✅ **유효성 보장**: 각 상태에서 유효한 필드만 존재
- ✅ **컴파일 시점 검증**: 잘못된 상태 전환은 컴파일 오류
- ✅ **함수 시그니처가 계약**: `validateOrder(SubmittedOrder)` → 입력은 Submitted, 출력은 Validated
- ✅ **다이어그램 불필요**: 타입 정의 자체가 흐름도

### OOP vs FP 문서화 비교

| 항목 | OOP (UML + Flow Chart) | FP (타입 정의) |
| :--- | :--- | :--- |
| **문서화 방식** | 다이어그램 + 코드 (분리됨) | 타입 정의 = 문서 (통합) |
| **동기화** | 수동 (개발자가 관리) | 자동 (컴파일러가 강제) |
| **상태 표현** | 텍스트 설명 | 타입 시스템 (명확함) |
| **유효한 필드** | 상태별로 nullable 필드 많음 | 각 상태가 정확한 필드만 보유 |
| **상태 전환 규칙** | 다이어그램에 표시 | 함수 시그니처로 표현 |
| **잘못된 상태** | 런타임 에러 | 컴파일 타임 에러 |
| **새로운 상태 추가** | 다이어그램 수정 + 코드 수정 | 타입 추가 → 자동으로 컴파일 에러 생성 |
| **도메인 전문가 이해도** | 다이어그램은 쉬움, 코드는 어려움 | 타입은 조금 어려움, 함수명은 명확함 |
| **시간 경과에 따른 정확도** | 점점 부정확해짐 | 항상 정확함 |

## 7. 실제 사례: 신약 화합물 합성 시스템

AI 신약 설계 플랫폼에서 화합물 합성 과정을 예로 들어봅시다.

### OOP 설계: UML + Flow Chart

**UML 클래스 다이어그램:**
```
┌──────────────────────────────────────┐
│          Compound                    │
├──────────────────────────────────────┤
│ - id: String                         │
│ - name: String                       │
│ - formula: String                    │
│ - status: SynthesisStatus            │
│ - weight: Double? (nullable)         │
│ - synthesisDate: Date? (nullable)    │
│ - testResults: TestResult? (nullable)│
│ - failureReason: String? (nullable)  │
├──────────────────────────────────────┤
│ + startSynthesis()                   │
│ + completeSynthesis(weight)          │
│ + startTesting(testResult)           │
│ + failSynthesis(reason)              │
│ + getWeight(): Double                │
│ + getTestResults(): TestResult       │
└──────────────────────────────────────┘

SynthesisStatus: PLANNING, IN_SYNTHESIS, SYNTHESIZED, TESTING, TESTED, FAILED
```

**상태 전환 다이어그램:**
```
[PLANNING]
    │
    ▼
startSynthesis()
    │
    ▼
[IN_SYNTHESIS]
    │
    ├──completeSynthesis()──> [SYNTHESIZED]
    │                              │
    │                         startTesting()
    │                              │
    │                              ▼
    │                         [TESTING]
    │                              │
    │                    ┌─────────┴──────────┐
    │                    │                    │
    │          completeTesting()      failTesting()
    │                    │                    │
    │                    ▼                    ▼
    │               [TESTED]            [FAILED]
    │
    └──failSynthesis()──> [FAILED]
```

**코드 예제:**
```java
public class Compound {
    private String id;
    private String name;
    private String formula;
    private SynthesisStatus status;
    private Double weight;
    private Date synthesisDate;
    private TestResult testResults;
    private String failureReason;
    
    public void startSynthesis() {
        if (status != SynthesisStatus.PLANNING) {
            throw new IllegalStateException("PLANNING 상태에서만 시작 가능");
        }
        this.status = SynthesisStatus.IN_SYNTHESIS;
    }
    
    public void completeSynthesis(Double weight) {
        if (status != SynthesisStatus.IN_SYNTHESIS) {
            throw new IllegalStateException("IN_SYNTHESIS 상태여야 함");
        }
        if (weight <= 0) {
            throw new IllegalArgumentException("무게는 0보다 커야 함");
        }
        this.status = SynthesisStatus.SYNTHESIZED;
        this.weight = weight;
        this.synthesisDate = new Date();
    }
    
    public Double getWeight() {
        // 문제: SYNTHESIZED 이후에만 weight가 유효한데
        // 이것을 코드 리뷰로만 확인 가능
        if (weight == null) {
            throw new IllegalStateException("아직 합성되지 않음");
        }
        return weight;
    }
    
    // ... 더 많은 메서드들
}
```

**문제점:**
- 🔄 다이어그램과 코드가 다르면 어느 것을 믿나?
- ❓ `getWeight()` 호출 전에 상태를 확인해야 하는데, 이것이 강제되지 않음
- 📝 새로운 상태가 추가되면 모든 메서드를 검토해야 함
- 🐛 테스트 시나리오가 수십 개가 됨 (모든 상태 조합)

### 함수형 설계: 타입 정의가 곧 문서

```plaintext
/**
 * 신약 화합물 합성 시스템
 * 
 * 화합물은 다음 단계를 거칩니다:
 * 1. Planning: 합성 계획 단계
 * 2. InSynthesis: 합성 진행 중
 * 3. Synthesized: 합성 완료
 * 4. Testing: 테스트 진행 중
 * 5. Tested: 테스트 완료
 * 6. Failed: 합성 또는 테스트 실패
 */

// 단계 1: 계획 단계
type PlanningCompound = {
  id: string;
  name: string;
  formula: string;
  createdAt: Date;
};

// 단계 2: 합성 진행 중
type InSynthesisCompound = PlanningCompound & {
  synthesisStartedAt: Date;
};

// 단계 3: 합성 완료 (무게 정보가 이제 존재)
type SynthesizedCompound = InSynthesisCompound & {
  weight: number;  // nullable이 아님! 이 단계에서는 반드시 있음
  synthesisCompletedAt: Date;
};

// 단계 4: 테스트 진행 중
type TestingCompound = SynthesizedCompound & {
  testStartedAt: Date;
};

// 단계 5: 테스트 완료
type TestedCompound = TestingCompound & {
  testResults: {
    solubility: number;
    toxicity: number;
    efficacy: number;
    notes: string;
  };
  testCompletedAt: Date;
};

// 단계 6: 실패한 화합물 (weight가 없을 수 있음)
type FailedCompound = PlanningCompound & {
  failureStage: 'SYNTHESIS' | 'TESTING';
  failureReason: string;
  failedAt: Date;
};

// 합타입으로 모든 가능한 상태를 표현
type Compound = 
  | { status: 'Planning'; data: PlanningCompound }
  | { status: 'InSynthesis'; data: InSynthesisCompound }
  | { status: 'Synthesized'; data: SynthesizedCompound }
  | { status: 'Testing'; data: TestingCompound }
  | { status: 'Tested'; data: TestedCompound }
  | { status: 'Failed'; data: FailedCompound };

// 상태 전환 함수들
/**
 * 합성을 시작합니다.
 * 
 * @param compound - Planning 단계의 화합물
 * @returns InSynthesis 단계의 화합물
 */
const startSynthesis = (compound: PlanningCompound): InSynthesisCompound => ({
  ...compound,
  synthesisStartedAt: new Date()
});

/**
 * 합성을 완료합니다.
 * 
 * 유효성 검사:
 * - 무게는 0보다 커야 함
 * - 화학식 검증
 * 
 * @param compound - InSynthesis 단계의 화합물
 * @param weight - 합성된 화합물의 무게
 * @returns 성공 시 SynthesizedCompound, 실패 시 Error
 */
const completeSynthesis = (
  compound: InSynthesisCompound,
  weight: number
): Result<SynthesizedCompound, SynthesisError> => {
  // 유효성 검사
  if (weight <= 0) {
    return Err(new SynthesisError('무게는 0보다 커야 합니다'));
  }

  // 성공
  return Ok({
    ...compound,
    weight,
    synthesisCompletedAt: new Date()
  });
};

/**
 * 합성 실패를 기록합니다.
 * 
 * @param compound - InSynthesis 단계의 화합물
 * @param reason - 실패 사유
 * @returns FailedCompound
 */
const failSynthesis = (
  compound: InSynthesisCompound,
  reason: string
): FailedCompound => ({
  id: compound.id,
  name: compound.name,
  formula: compound.formula,
  createdAt: compound.createdAt,
  failureStage: 'SYNTHESIS',
  failureReason: reason,
  failedAt: new Date()
});

/**
 * 테스트를 시작합니다.
 * 
 * @param compound - SynthesizedCompound (합성이 완료된 상태만 가능)
 * @returns TestingCompound
 */
const startTesting = (
  compound: SynthesizedCompound
): TestingCompound => ({
  ...compound,
  testStartedAt: new Date()
});

/**
 * 테스트를 완료합니다.
 * 
 * @param compound - TestingCompound
 * @param results - 테스트 결과
 * @returns TestedCompound
 */
const completeTesting = (
  compound: TestingCompound,
  results: {
    solubility: number;
    toxicity: number;
    efficacy: number;
    notes: string;
  }
): TestedCompound => ({
  ...compound,
  testResults: results,
  testCompletedAt: new Date()
});

/**
 * 테스트 실패를 기록합니다.
 * 
 * @param compound - TestingCompound
 * @param reason - 실패 사유
 * @returns FailedCompound
 */
const failTesting = (
  compound: TestingCompound,
  reason: string
): FailedCompound => ({
  id: compound.id,
  name: compound.name,
  formula: compound.formula,
  createdAt: compound.createdAt,
  weight: compound.weight,
  synthesisStartedAt: compound.synthesisStartedAt,
  synthesisCompletedAt: compound.synthesisCompletedAt,
  failureStage: 'TESTING',
  failureReason: reason,
  failedAt: new Date()
});

/**
 * 합성된 화합물의 무게를 가져옵니다.
 * 
 * 이 함수는 SynthesizedCompound 이상의 상태에서만 호출 가능합니다.
 * (Planning, InSynthesis 상태에서는 컴파일 에러 발생)
 * 
 * @param compound - SynthesizedCompound | TestingCompound | TestedCompound
 * @returns 무게
 */
const getWeight = (
  compound: SynthesizedCompound | TestingCompound | TestedCompound
): number => compound.weight;

/**
 * 테스트 결과를 가져옵니다.
 * 
 * 이 함수는 TestedCompound 상태에서만 호출 가능합니다.
 * 
 * @param compound - TestedCompound (정확하게 tested 상태만 받음)
 * @returns 테스트 결과
 */
const getTestResults = (compound: TestedCompound) => compound.testResults;

// 사용 예제
const workflow = async () => {
  // 1. 화합물 계획
  const planning: Compound = {
    status: 'Planning',
    data: {
      id: 'CMPD-001',
      name: 'Compound A',
      formula: 'C20H30N4O5',
      createdAt: new Date()
    }
  };

  // 2. 합성 시작
  const inSynthesis = startSynthesis(planning.data);
  // ❌ 이런 실수는 컴파일 타임에 감지됨:
  // const weight = getWeight(inSynthesis);  // Type Error!
  // getTestResults(inSynthesis);            // Type Error!

  // 3. 합성 완료
  const synthesized = await completeSynthesis(inSynthesis, 45.2);
  if (synthesized.isErr()) {
    console.error('합성 실패:', synthesized.error);
    return;
  }

  const compound = synthesized.value;

  // ✅ 이제 getWeight() 호출 가능
  const weight = getWeight(compound);
  console.log(`합성 완료, 무게: ${weight}g`);

  // 4. 테스트 시작
  const testing = startTesting(compound);

  // 5. 테스트 완료
  const tested = completeTesting(testing, {
    solubility: 85.3,
    toxicity: 2.1,
    efficacy: 92.5,
    notes: '우수한 결과'
  });

  // ✅ 이제 getTestResults() 호출 가능
  const results = getTestResults(tested);
  console.log('테스트 결과:', results);
};
```

**함수형 문서화의 이점:**

- ✅ **타입이 문서**: 각 단계의 필드가 명확하게 정의됨
- ✅ **컴파일 검증**: `getWeight(planningCompound)` 호출은 컴파일 타임에 에러
- ✅ **자동 동기화**: 타입을 수정하면 함수도 함께 에러 발생
- ✅ **함수 시그니처가 계약**: `completeSynthesis(InSynthesis): Result<Synthesized>` → 입출력이 명확
- ✅ **상태 불가능 원천 차단**: Synthesized 상태의 화합물은 항상 weight를 가짐
- ✅ **테스트 시나리오 감소**: 잘못된 상태 조합을 만들 수 없음

## 8. 마무리

도메인 문서화의 관점에서:

- **OOP (UML + Flow Chart)**: 시각적으로 이해하기 쉽지만, 코드와 동기화되지 않고 시간이 지날수록 부정확해짐
- **FP (타입 정의)**: 처음에는 추상적으로 보이지만, 코드와 100% 동기화되고 컴파일러가 검증함

**도메인을 깊이 이해하는 것**이 좋은 소프트웨어 설계의 시작이며, **FP의 타입 시스템**은 이를 명확하게 문서화하고 검증하는 강력한 도구입니다. 특히 복잡한 상태 전환이 있는 도메인(신약 설계)에서는 함수형이 진정한 가치를 발휘합니다.
