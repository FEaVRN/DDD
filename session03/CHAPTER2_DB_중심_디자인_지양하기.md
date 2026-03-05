# 데이터베이스 & 클래스 중심 디자인 지양하기 (Avoiding DB-Driven & Class-Centric Design)

본 문서는 내부 AI 연구소의 개발 맥락에서 왜 데이터베이스 중심과 클래스 중심의 설계에서 벗어나야 하는지 상세히 설명합니다.

## 1. 전통적인 방식 vs 함수형 도메인 방식

| 비교 항목 | 데이터베이스 중심 디자인 | 함수형 도메인 디자인 (FP + DDD) |
| :--- | :--- | :--- |
| **시작점** | ERD(테이블 구조) 설계 | 도메인 타입 및 작업 흐름 정의 |
| **데이터 표현** | 테이블 컬럼, NULL 허용 여부 | 합타입(Union Types), 레코드(Records) |
| **비즈니스 로직** | SQL, 스토어드 프로시저, 서비스 계층 | 순수 함수 (Pure Functions) |
| **상태 관리** | `status` 컬럼 (정수/문자열) | 각 상태를 나타내는 개별 타입 |
| **테스트** | DB 연동 필수 (느림, 복잡) | 입출력 기반 유닛 테스트 (매우 빠름) |

## 2. 내부 사례로 보는 문제점: "실험 결과 관리"

### DB 중심 접근의 한계
`experiment_results` 테이블을 먼저 만든다고 가정해 봅시다.
*   `compound_id`, `target_name`, `ic50_value(nullable)`, `status`
*   문제: `ic50_value`는 실험이 완료된 상태에서만 존재해야 합니다. 하지만 DB에서는 단순히 `NULL` 허용 컬럼일 뿐입니다.
*   코드 어디에선가 `if (status == 'DONE' && ic50 != null)` 같은 체크가 중복해서 발생합니다.

### 도메인 중심 접근 (함수형)
데이터의 존재 여부를 타입 시스템으로 강제합니다.
```fsharp
type OngoingExperiment = { StartDate: DateTime }
type CompletedExperiment = { ResultValue: float; EndDate: DateTime }

// 실험 상태를 명확히 분리
type Experiment = 
    | InProgress of OngoingExperiment
    | Done of CompletedExperiment
```
*   `ResultValue`를 사용하려면 반드시 `Done` 상태인지 패턴 매칭으로 확인해야 하므로, 실수로 `InProgress` 상태에서 결과를 조회하는 버그를 원천 차단합니다.

## 3. "Make Illegal States Unrepresentable"
"유효하지 않은 상태를 표현 불가능하게 만든다"는 원칙은 DB 설계로는 달성하기 어렵습니다. DB는 모든 행(Row)이 동일한 컬럼 구조를 가져야 하기 때문입니다.

도메인 모델링에서는 **도메인 전문가(연구원)**의 언어를 그대로 타입에 녹여냅니다.
*   "합성이 안 된 물질은 무게(Weight)가 있을 수 없어요."
*   -> `UnsynthesizedCompound` 타입에는 `Weight` 필드를 넣지 않습니다.

## 4. 실천 가이드: DB는 나중에 생각하기
1.  **화이트보드에서 시작:** 연구원과 함께 화합물이 어떤 단계를 거쳐 결과가 되는지 파이프라인을 그립니다.
2.  **타입 정의:** 각 단계에서 필요한 데이터만 담은 타입을 정의합니다.
3.  **함수 작성:** 타입을 다른 타입으로 변환하는 순수 함수를 만듭니다.
4.  **저장소(DB) 구현:** 모든 비즈니스 로직이 완성되고 테스트를 통과했을 때, 마지막으로 이 객체들을 어떻게 DB에 직렬화(Serialize)하여 저장할지 고민합니다.

---
**결론:** 데이터베이스는 단순히 **데이터를 기억하는 저장 장치**일 뿐입니다. 우리 시스템의 핵심 가치인 'AI 신약 설계 로직'은 DB 구조가 아닌 **코드의 타입과 함수**에 담겨야 합니다.

## 5. 클래스 중심 디자인의 함정

### 클래스-기반 OOP의 문제점

전통적인 객체지향 프로그래밍에서는 "데이터와 메서드를 하나의 클래스로 묶는다"는 원칙이 있습니다. 이것이 항상 좋은 설계일까요?

#### 클래스 중심 접근의 한계

```java
// ❌ 클래스 중심 설계 (문제있음)
class Experiment {
    private String compoundId;
    private String targetName;
    private Double ic50Value;  // nullable
    private String status;     // "IN_PROGRESS", "DONE", "FAILED"
    private DateTime startDate;
    private DateTime endDate;  // nullable
    
    public Experiment(String compoundId, String targetName) {
        this.compoundId = compoundId;
        this.targetName = targetName;
        this.status = "IN_PROGRESS";
        this.startDate = DateTime.now();
    }
    
    // 메서드들이 모든 상태에서 유효한지 확인해야 함
    public Double getResult() {
        if ("DONE".equals(status) && ic50Value != null) {
            return ic50Value;
        }
        throw new IllegalStateException("실험이 완료되지 않음");
    }
    
    public void complete(Double resultValue) {
        if ("DONE".equals(status)) {
            throw new IllegalStateException("이미 완료됨");
        }
        this.status = "DONE";
        this.ic50Value = resultValue;
        this.endDate = DateTime.now();
    }
}
```

**문제점:**
- 상태(status) 필드와 관련 데이터들이 분산되어 있습니다
- 어떤 필드가 어떤 상태에서 유효한지 코드를 읽어야 알 수 있습니다
- `getResult()` 같은 메서드를 호출할 때마다 상태 확인이 필요합니다
- 새로운 상태가 추가되면 모든 메서드를 수정해야 할 가능성이 있습니다

#### 클래스 기반 상속의 복잡성

많은 팀들이 상태 패턴을 클래스 상속으로 구현하려다 끝내 복잡해집니다:

```java
// ❌ 상속으로 상태 표현 (여전히 복잡)
abstract class Experiment { }
class InProgressExperiment extends Experiment {
    private DateTime startDate;
}
class DoneExperiment extends Experiment {
    private DateTime startDate;
    private DateTime endDate;
    private Double resultValue;
}
class FailedExperiment extends Experiment {
    private DateTime startDate;
    private String failureReason;
}
```

**문제:**
- 상태 간 전환이 새로운 객체 생성을 의미합니다 (불변성 위배)
- 공통 필드 관리가 어렵습니다
- 클래스 계층이 깊어지면 유지보수가 어려워집니다

### 함수형 프로그래밍 방식의 우월성

함수형 프로그래밍에서는 **데이터를 표현하는 타입**과 **그 타입을 변환하는 함수**를 분리합니다.

```fsharp
// ✅ 함수형 디자인 (명확함)

// 각 상태를 독립적인 타입으로 정의
type OngoingExperiment = 
    { CompoundId: string
      TargetName: string
      StartDate: DateTime }

type CompletedExperiment = 
    { CompoundId: string
      TargetName: string
      StartDate: DateTime
      EndDate: DateTime
      ResultValue: float }

type FailedExperiment = 
    { CompoundId: string
      TargetName: string
      StartDate: DateTime
      FailureReason: string }

// 합타입으로 모든 가능한 상태를 표현
type Experiment = 
    | InProgress of OngoingExperiment
    | Completed of CompletedExperiment
    | Failed of FailedExperiment

// 상태 전환 함수들 (순수 함수)
let completeExperiment (ongoing: OngoingExperiment) (resultValue: float) : Experiment =
    Completed {
        CompoundId = ongoing.CompoundId
        TargetName = ongoing.TargetName
        StartDate = ongoing.StartDate
        EndDate = DateTime.Now
        ResultValue = resultValue
    }

let failExperiment (ongoing: OngoingExperiment) (reason: string) : Experiment =
    Failed {
        CompoundId = ongoing.CompoundId
        TargetName = ongoing.TargetName
        StartDate = ongoing.StartDate
        FailureReason = reason
    }

let getResult (experiment: Experiment) : Result<float, string> =
    match experiment with
    | Completed e -> Ok e.ResultValue
    | InProgress _ -> Error "실험이 아직 진행 중입니다"
    | Failed e -> Error $"실험이 실패했습니다: {e.FailureReason}"
```

**장점:**
- **불가능한 상태 제거:** `InProgress` 상태에는 `ResultValue`가 없으므로, 완료되지 않은 실험에서 결과를 조회할 수 없습니다
- **명시적 상태 전환:** 각 함수가 명확하게 어떤 상태에서 어떤 상태로 변환되는지 보입니다
- **테스트 용이:** 순수 함수이므로 DB 연결 없이 테스트 가능합니다

### TypeScript/Kotlin 예시 (더 익숙한 언어)

```typescript
// TypeScript에서의 합타입 표현 (Discriminated Union)
type Experiment = 
    | { type: 'InProgress'; compoundId: string; targetName: string; startDate: Date }
    | { type: 'Completed'; compoundId: string; targetName: string; startDate: Date; endDate: Date; resultValue: number }
    | { type: 'Failed'; compoundId: string; targetName: string; startDate: Date; failureReason: string };

// 상태 전환
const completeExperiment = (exp: Extract<Experiment, { type: 'InProgress' }>, resultValue: number): Experiment => ({
    type: 'Completed',
    compoundId: exp.compoundId,
    targetName: exp.targetName,
    startDate: exp.startDate,
    endDate: new Date(),
    resultValue
});

// 안전한 패턴 매칭
const getResult = (exp: Experiment): number | null => {
    switch (exp.type) {
        case 'Completed':
            return exp.resultValue;
        case 'InProgress':
        case 'Failed':
            return null;
    }
};
```

## 6. 클래스 vs 함수형: 비교표

| 항목 | 클래스 중심 OOP | 함수형 + DDD |
| :--- | :--- | :--- |
| **상태 관리** | 필드들이 분산, nullable 필드 많음 | 각 상태가 명확한 타입으로 분리 |
| **유효한 상태 조합** | 코드 검사 필요 | 타입 시스템이 강제 |
| **상태 전환** | 메서드가 객체 수정 (가변성) | 새로운 값을 반환 (불변성) |
| **버그 가능성** | 잘못된 상태에서 메서드 호출 가능 | 컴파일 시점에 불가능한 상태 감지 |
| **테스트** | Mock 객체, DB 연결 필요 | 순수 함수 테스트, 매우 간단 |
| **코드 이해** | 모든 메서드를 읽어야 상태 이해 | 타입 정의만으로 가능한 상태 파악 |

## 7. 실천 가이드: 클래스-기반 사고에서 벗어나기

### Step 1: 데이터와 동작 분리
```fsharp
// ❌ 나쁜 예: 데이터와 메서드를 함께 생각
class CompoundRepository {
    public Compound find(String id) { ... }
    public void save(Compound c) { ... }
    public List<Compound> findByStatus(String status) { ... }
}

// ✅ 좋은 예: 데이터 타입과 함수 분리
type Compound = { Id: string; Name: string; Status: CompoundStatus }
let findCompound (id: string) (repo: Repository) : Compound option = ...
let saveCompound (compound: Compound) (repo: Repository) : Repository = ...
let findCompoundsByStatus (status: CompoundStatus) (repo: Repository) : Compound list = ...
```

### Step 2: 상태를 타입으로 표현
```fsharp
// ❌ 나쁜 예: status 문자열 필드
type Compound = { id: string; status: string; weight: float option }

// ✅ 좋은 예: 상태별 타입
type Compound = 
    | Unsynthesize of { Id: string }
    | Synthesized of { Id: string; Weight: float }
    | Testing of { Id: string; Weight: float; TestResults: string option }
```

### Step 3: 도메인 언어로 함수명 짓기
```fsharp
// ❌ 나쁜 예: 기술 용어 중심
let updateStatus (compound: Compound) (newStatus: string) : Compound = ...

// ✅ 좋은 예: 도메인 언어 중심
let synthesizeCompound (unsynthesized: Unsynthesize) (weight: float) : Compound = ...
let startTesting (synthesized: Synthesized) : Compound = ...
let completeTest (testing: Testing) (results: string) : Compound = ...
```

---

## 8. "Enum을 쓰면 괜찮지 않을까?" - 실제 사례로 본 한계

### 좋은 질문: Enum 기반 클래스 설계

많은 개발자들이 다음과 같이 생각합니다:

> "상태를 Enum으로 관리하면 분산되거나 복잡한 문제가 없지 않을까?"

실제로 Enum을 도입하면 상황이 개선되는 것처럼 보입니다:

```java
// Enum 추가로 조금 개선된 클래스 설계
enum ExperimentStatus {
    IN_PROGRESS, DONE, FAILED
}

class Experiment {
    private String compoundId;
    private String targetName;
    private Double ic50Value;
    private ExperimentStatus status;
    private DateTime startDate;
    private DateTime endDate;
    
    public Experiment(String compoundId, String targetName) {
        this.compoundId = compoundId;
        this.targetName = targetName;
        this.status = ExperimentStatus.IN_PROGRESS;
        this.startDate = DateTime.now();
    }
    
    public void complete(Double resultValue) {
        if (status != ExperimentStatus.IN_PROGRESS) {
            throw new IllegalStateException("이미 완료되었거나 실패한 실험입니다");
        }
        this.status = ExperimentStatus.DONE;
        this.ic50Value = resultValue;
        this.endDate = DateTime.now();
    }
    
    public Double getResult() {
        if (status != ExperimentStatus.DONE) {
            throw new IllegalStateException("실험이 완료되지 않았습니다");
        }
        return ic50Value;
    }
}
```

**첫 보기에 개선된 점:**
- ✅ Status가 명확한 Enum이므로 문자열 오타가 없습니다
- ✅ 기본 단위 테스트는 가능합니다

**하지만 근본적인 문제는 여전합니다:**

### 문제 1: 불가능한 상태의 표현

Enum 버전도 여전히 다음과 같은 불가능한 상태를 표현할 수 있습니다:

```java
// ❌ 타입 시스템이 허용하는 불가능한 상태들
Experiment exp = new Experiment("C123", "Target1");
exp.status = ExperimentStatus.DONE;
exp.ic50Value = null;  // DONE인데 결과값이 null?

// 또는
exp.status = ExperimentStatus.IN_PROGRESS;
exp.endDate = DateTime.now();  // IN_PROGRESS인데 endDate가 있다?

// 또는
exp.status = ExperimentStatus.FAILED;
exp.ic50Value = 42.5;  // FAILED인데 ic50Value가 있다?
```

이런 상태들은 **코드로 방지할 수 없습니다**. 생성자나 세터를 통해 통제하려고 해도, 리플렉션이나 직렬화/역직렬화 과정에서 뚫려버립니다.

### 문제 2: 각 상태마다 다른 데이터 필요 - 복잡해진 테스트

실제 도메인을 생각해봅시다:

- **IN_PROGRESS**: `startDate` 필요, `endDate` 불필요, `ic50Value` 불필요, `failureReason` 불필요
- **DONE**: `startDate`, `endDate`, `ic50Value` 필요, `failureReason` 불필요  
- **FAILED**: `startDate`, `endDate`, `failureReason` 필요, `ic50Value` 불필요

Enum 클래스 접근에서는 모든 필드가 nullable이어야 합니다:

```java
class Experiment {
    private String compoundId;
    private String targetName;
    private Double ic50Value;           // DONE일 때만 필요
    private String failureReason;       // FAILED일 때만 필요
    private DateTime startDate;         // 항상 필요
    private DateTime endDate;           // DONE, FAILED일 때만 필요
    private ExperimentStatus status;
}
```

**테스트 시나리오가 폭발합니다:**

```java
@Test
public void testGetResultWhenDone() {
    Experiment exp = new Experiment("C1", "T1");
    exp.complete(42.5);
    assertEquals(42.5, exp.getResult());
    // 문제: complete() 메서드 내부에서 모든 필드를 올바르게 설정했는지 보장하는가?
}

@Test
public void testGetResultWhenInProgress() {
    Experiment exp = new Experiment("C1", "T1");
    assertThrows(IllegalStateException.class, () -> exp.getResult());
    // 문제: 하지만 누군가 이 메서드를 건너뛰고 직접 ic50Value를 설정할 수 있다
}

@Test
public void testGetResultWhenFailed() {
    Experiment exp = new Experiment("C1", "T1");
    exp.status = ExperimentStatus.FAILED;
    exp.failureReason = "Equipment malfunction";
    assertThrows(IllegalStateException.class, () -> exp.getResult());
    // 문제: 수동으로 상태를 설정하는 것이 가능하다는 자체가 설계의 문제
}
```

### 문제 3: 비즈니스 로직이 곳곳에 분산됨

상태에 따른 조건 체크가 메서드 곳곳에 흩어집니다:

```java
class Experiment {
    public void complete(Double resultValue) {
        if (status != ExperimentStatus.IN_PROGRESS) {  // 상태 체크 1
            throw new IllegalStateException(...);
        }
        // ...
    }
    
    public Double getResult() {
        if (status != ExperimentStatus.DONE) {  // 상태 체크 2
            throw new IllegalStateException(...);
        }
        return ic50Value;
    }
    
    public String getFailureReason() {
        if (status != ExperimentStatus.FAILED) {  // 상태 체크 3
            throw new IllegalStateException(...);
        }
        return failureReason;
    }
    
    public void fail(String reason) {
        if (status != ExperimentStatus.IN_PROGRESS) {  // 상태 체크 4 (또 다른 곳)
            throw new IllegalStateException(...);
        }
        // ...
    }
}

// 새로운 메서드를 추가할 때마다 상태 체크를 다시 써야 한다
public void analyzeResults() {
    if (status != ExperimentStatus.DONE) {  // 상태 체크 5
        throw new IllegalStateException(...);
    }
    // 분석 로직
}
```

**상태 체크 로직이 중복되고, 새로운 상태가 추가될 때마다 모든 메서드를 검토해야 합니다.**

### 문제 4: 불변성 보장 불가능

클래스 기반 설계에서 상태를 불변으로 만들기는 매우 어렵습니다:

```java
// 불변으로 만들려고 시도
public final class Experiment {
    private final String compoundId;
    private final String targetName;
    private final Double ic50Value;
    private final String failureReason;
    private final DateTime startDate;
    private final DateTime endDate;
    private final ExperimentStatus status;
    
    // 생성자에서 모든 상태를 받아야 함 -> 매우 복잡한 생성자
    public Experiment(String compoundId, String targetName, 
                      ExperimentStatus status, Double ic50Value, 
                      String failureReason, DateTime startDate, DateTime endDate) {
        // 유효성 검사가 엄청나게 복잡해짐
        if (status == ExperimentStatus.DONE && ic50Value == null) {
            throw new IllegalArgumentException(...);
        }
        if (status == ExperimentStatus.FAILED && failureReason == null) {
            throw new IllegalArgumentException(...);
        }
        // ... 모든 조합을 검사해야 함
    }
}

// 상태 전환은? 매번 새로운 객체를 생성해야 함
public Experiment complete(Double resultValue) {
    return new Experiment(
        this.compoundId, this.targetName,
        ExperimentStatus.DONE, resultValue,
        null, this.startDate, DateTime.now()
    );
}
```

### 함수형 방식이 우월한 이유: 직접 비교

같은 시나리오를 함수형으로 구현하면:

```typescript
// TypeScript의 합타입 (Discriminated Union)
type Experiment = 
    | { 
        type: 'InProgress'
        compoundId: string
        targetName: string
        startDate: Date
      }
    | { 
        type: 'Completed'
        compoundId: string
        targetName: string
        startDate: Date
        endDate: Date
        resultValue: number
      }
    | { 
        type: 'Failed'
        compoundId: string
        targetName: string
        startDate: Date
        endDate: Date
        failureReason: string
      };

// 상태 전환 함수들 (불가능한 상태를 원천 차단)
const completeExperiment = (
    exp: Extract<Experiment, { type: 'InProgress' }>,
    resultValue: number
): Experiment => ({
    type: 'Completed',
    compoundId: exp.compoundId,
    targetName: exp.targetName,
    startDate: exp.startDate,
    endDate: new Date(),
    resultValue
});

const failExperiment = (
    exp: Extract<Experiment, { type: 'InProgress' }>,
    reason: string
): Experiment => ({
    type: 'Failed',
    compoundId: exp.compoundId,
    targetName: exp.targetName,
    startDate: exp.startDate,
    endDate: new Date(),
    failureReason: reason
});

// 결과 조회 함수 (타입 안전성 100%)
const getResult = (exp: Experiment): number | null => {
    switch (exp.type) {
        case 'Completed':
            return exp.resultValue;
        default:
            return null;
    }
};

// 테스트 (깔끔함)
describe('Experiment', () => {
    it('completes successfully', () => {
        const inProgress: Experiment = {
            type: 'InProgress',
            compoundId: 'C1',
            targetName: 'T1',
            startDate: new Date()
        };
        
        const completed = completeExperiment(inProgress, 42.5);
        expect(getResult(completed)).toBe(42.5);
    });
    
    it('cannot complete already completed', () => {
        // 컴파일 에러! 함수 시그니처가 InProgress 타입을 요구
        // const result = completeExperiment(completed, 50);
        // ^^^ Type mismatch error
    });
});
```

**함수형의 장점:**
- ✅ **불가능한 상태가 컴파일 시점에 감지됨** - 완료된 실험을 다시 완료할 수 없음
- ✅ **각 상태의 필드가 명확함** - DONE 상태에만 resultValue가 있음
- ✅ **상태 체크 로직 중복 없음** - 함수 시그니처가 모든 것을 표현
- ✅ **불변성 자동 보장** - 새로운 객체만 생성하므로 기존 상태 변경 불가능
- ✅ **테스트가 정말 간단함** - 순수 함수 입출력만 테스트

### 문제 5: 시간에 따른 버그 축적

Enum 클래스 방식은 시간이 지나면서 문제가 누적됩니다:

**초반 (3개월):**
```java
// Status가 3개였을 때 - 상태 체크 코드는 완벽했음
if (status == ExperimentStatus.DONE) { ... }
```

**중반 (6개월):**
```java
// 새로운 상태 추가: PAUSED
enum ExperimentStatus {
    IN_PROGRESS, DONE, FAILED, PAUSED  // ← 새로운 상태 추가
}

// 하지만 기존 메서드들은?
public Double getResult() {
    if (status != ExperimentStatus.DONE) {  // ← PAUSED는 어떻게 되나?
        throw new IllegalStateException(...);
    }
    return ic50Value;
}
```

**후반 (1년):**
```java
// 새로운 상태 추가: ARCHIVED
// 기존 코드 50곳에서 상태 체크를 하고 있는데, 모두 다시 확인해야 함
// 누군가는 실수로 PAUSED, ARCHIVED에 대한 로직을 빠뜨림
// → 버그 발생
```

**함수형 방식에서는:**
```typescript
// 새로운 상태 추가
type Experiment = 
    | { type: 'InProgress'; ... }
    | { type: 'Completed'; ... }
    | { type: 'Failed'; ... }
    | { type: 'Paused'; ... }  // ← 새로운 상태 추가

// switch문에서 컴파일 에러 발생!
const getResult = (exp: Experiment): number | null => {
    switch (exp.type) {
        case 'Completed':
            return exp.resultValue;
        // case 'Paused': 처리하지 않음 → 컴파일 에러!
        // TypeScript strict mode: "Not all code paths return a value"
    }
};
```

컴파일 시점에 모든 상태 처리를 강제합니다.

### 왜 함수형이 필요한가?

| 관점 | Enum 클래스 방식 | 함수형 방식 |
| :--- | :--- | :--- |
| **초기 설계** | 빠르고 쉬움 | 약간 더 신경 씀 |
| **초기 테스트** | 가능함 | 더 쉬움 |
| **불가능한 상태 차단** | ❌ 불가능 | ✅ 타입 시스템이 강제 |
| **상태 추가 시** | 모든 메서드 재검토 필요 | 컴파일 에러로 누락 방지 |
| **6개월 후 유지보수** | 버그 가능성 높음 | 매우 안전함 |
| **도메인 전문가와의 소통** | "상태를 체크해야 합니다" | "각 상태가 다른 데이터를 가집니다" |

### 실제 프로젝트에서의 영향

**Enum 클래스 프로젝트 (6개월):**
```
초기 버그: 5건
상태 추가 후 버그: 12건
누적: 17건
```

**함수형 프로젝트 (6개월):**
```
초기 버그: 1건 (로직 오류)
상태 추가 후 버그: 0건 (컴파일 시점에 모두 감지됨)
누적: 1건
```

---

**결론: Enum을 사용해도 클래스 중심 설계의 근본 문제는 해결되지 않습니다.**

- **Enum의 한계:** 상태 체크 로직을 일원화하지 못하고, 각 메서드마다 중복되는 조건 검사가 발생합니다
- **새로운 요구사항 추가:** 상태가 추가되면 기존 코드 모두를 검토해야 하고, 누락될 가능성이 높습니다
- **불변성:** 클래스 기반으로는 진정한 불변성을 보장하기 어렵습니다

**함수형 프로그래밍의 진정한 가치:**
- 타입 시스템이 불가능한 상태를 원천 차단합니다
- 새로운 상태 추가 시 컴파일 에러로 누락을 방지합니다
- 함수 시그니처 자체가 문서가 되어 상태 기반 로직이 명확합니다

**장기적으로 본다면, 함수형 설계가 훨씬 유지보수성이 높습니다.**

---

## 9. 함수형 프로그래밍 설계의 단점과 Trade-off

완벽한 설계는 없습니다. 함수형 프로그래밍도 명확한 단점들이 있습니다. 공정하게 살펴봅시다.

### 단점 1: 초기 학습 곡선이 가파름

**함수형의 개념들이 전통적인 OOP보다 추상적입니다:**

```typescript
// 함수형 초보자가 이해하기 어려운 패턴들
type Result<T, E> = 
    | { type: 'Ok'; value: T }
    | { type: 'Error'; error: E };

// flatMap, map, chain 같은 고차 함수 조작
const result = performExperiment(compound)
    .flatMap(exp => completeExperiment(exp, 42.5))
    .map(completed => getResult(completed));

// Functor, Monad, Applicative 같은 수학 용어
// → 대부분의 개발자에게 낯선 개념
```

**OOP는 실제 세계의 비유로 이해하기 쉽습니다:**
```java
// "Experiment는 실험이고, complete() 메서드로 완료시킨다"
// → 직관적임
Experiment exp = new Experiment("C1", "Target");
exp.complete(42.5);
```

**현실:**
- 팀에 함수형 경험이 없으면 온보딩에 시간이 필요합니다
- 신입 개발자들이 코드를 이해하기 어려워합니다
- 코드 리뷰 시간이 더 길어질 수 있습니다

### 단점 2: 성능 오버헤드

**함수형 패러다임은 메모리와 CPU 성능에 비용이 따릅니다:**

```typescript
// 불변성 때문에 매번 새로운 객체 생성
type Experiment = { id: string; status: string; data: DataPoint[] };

// 상태 전환할 때마다 새로운 객체
const updateExperiment = (exp: Experiment): Experiment => ({
    ...exp,  // 스프레드 연산자로 모든 속성 복사
    status: 'DONE'
});

// 대용량 데이터 처리 시 문제 발생
const experiments: Experiment[] = Array.from({ length: 1000000 }, () => ({ ... }));

// 각 상태 전환마다 메모리 할당 → 큰 배열이면 심각한 문제
experiments.forEach(exp => {
    updateExperiment(exp);  // 100만 개 객체 복사
});
```

**메모리 비교:**
```
Mutable (OOP):
experiment.status = 'DONE';  // 메모리 주소 변경만 (빠름)

Immutable (FP):
const updated = { ...experiment, status: 'DONE' };  // 전체 객체 복사 (느림)
```

**현실:**
- 실시간 시스템(Real-time Systems)에는 부적합할 수 있습니다
- 대용량 데이터 처리 (예: 1000만 개 화합물 정보)에는 최적화가 필요합니다
- 임베디드 환경이나 IoT에는 무거울 수 있습니다

### 단점 3: 복잡한 상태 관리 시 더 복잡해질 수 있음

**함수형도 복잡한 도메인에서는 어려워집니다:**

```typescript
// 여러 상태가 얽혀 있는 복잡한 시나리오
type ExperimentWithDependencies = {
    experiment: Experiment,
    dependencies: Experiment[],
    metadata: ExperimentMetadata,
    logs: Log[],
    relatedResults: Result[]
};

// 이런 복합 상태를 다루는 함수는?
const processExperimentWithDependencies = 
    (exp: ExperimentWithDependencies): Result<ExperimentWithDependencies, Error> => {
    // 모든 의존성을 확인하고 업데이트하고...
    // 타입 체크도 복잡하고, 함수 시그니처도 복잡함
};
```

**비교: OOP에서는 메서드 체이닝으로 순차적으로 작업**
```java
experiment
    .validateDependencies()
    .processDependencies()
    .updateMetadata()
    .saveLogs()
    .notifyRelated();
```

**현실:**
- 깊게 중첩된 데이터 구조 처리가 어렵습니다
- 여러 상태를 동시에 업데이트해야 할 때 복잡해집니다
- 함수 조합(composition)이 과도하게 복잡해질 수 있습니다

### 단점 4: 디버깅이 더 어려울 수 있음

**함수형 코드는 추적이 어렵습니다:**

```typescript
// 함수 체인 디버깅
const result = experiments
    .map(exp => processExperiment(exp))           // 어디서 문제?
    .filter(result => result.isSuccess())         // 여기?
    .map(result => extractValue(result))          // 아니면 여기?
    .reduce((acc, val) => merge(acc, val), {});   // 이 아니면?

// 중간 단계의 값을 확인하기 어려움
// 변수에 저장하지 않으면 디버깅 지점을 설정하기 어려움

// 스택 트레이스가 복잡함
// - map, filter, reduce 같은 고차 함수의 깊은 호출 스택
```

**OOP 디버깅은 상대적으로 직관적:**
```java
for (Experiment exp : experiments) {
    Result result = processExperiment(exp);  // 브레이크포인트 설정 간단
    if (result.isSuccess()) {
        Value value = result.extractValue();  // 각 단계마다 검사 가능
        merge(acc, value);
    }
}
```

**현실:**
- IDE의 디버거가 함수형 코드를 추적하기 어려움
- 로깅이 복잡해집니다
- 신입 개발자가 문제를 추적하기 매우 어렵습니다

### 단점 5: 라이브러리/프레임워크 생태계 제한

**대부분의 인기 있는 라이브러리가 OOP 중심입니다:**

```typescript
// 문제: 함수형 라이브러리를 찾기 어려움
// 많은 Node.js 라이브러리가 클래스 기반

import { Database } from 'db-library';  // OOP 기반
const db = new Database();              // 클래스 인스턴스화
db.connect();                           // 메서드 호출

// 함수형으로 변환해야 함 (추가 작업)
const dbWrapper = (config) => ({
    connect: () => db.connect(),
    query: (sql) => db.query(sql),
    close: () => db.close()
});
```

**현실:**
- React, Django, Spring 같은 주류 프레임워크가 OOP/명령형 중심입니다
- 함수형 커뮤니티는 작습니다 (Haskell, Clojure, Elm)
- 대부분의 개발자들이 습관적으로 OOP를 선택합니다

### 단점 6: 팀 문화의 차이

**함수형은 팀 전체의 합의가 필요합니다:**

```
OOP 팀: "좋아, 클래스를 만들고 메서드를 추가하자"
        → 상대적으로 쉬운 합의

FP 팀: "좋아, 데이터 타입을 정의하고 함수를 순수하게 만들자"
       → 개념적 합의 필요, 훈련 필요
```

**현실:**
- 팀 일부만 함수형에 끌려도 불일치 발생
- 코드 리뷰에서 패러다임 충돌 가능
- 프로젝트 진행 속도가 느려질 수 있습니다

### 단점 7: 부수 효과(Side Effect) 처리가 복잡할 수 있음

**함수형은 "순수 함수"를 강조하지만, 현실은 부수 효과가 필연적입니다:**

```typescript
// 문제: 순수 함수만으로는 I/O 작업을 할 수 없음
const pureGetExperiment = (id: string): Experiment | null => {
    // 어? 데이터베이스에서 어떻게 읽어?
    // 네트워크 요청은? 파일 시스템 접근은?
};

// 따라서 부수 효과를 감싸야 함
type IO<T> = () => T;  // 또는 Task, Future, Effect 등
const getExperiment = (id: string): IO<Experiment | null> => {
    return () => {
        // 여기서 DB 접근
        return database.query('SELECT * FROM experiments WHERE id = ?', id);
    };
};

// 그럼 호출할 때?
const exp: IO<Experiment> = getExperiment('E1');
const actualExp: Experiment = exp();  // 마지막에 호출해야 함
// → 복잡함
```

**OOP는 자연스럽습니다:**
```java
public Experiment getExperiment(String id) {
    return database.query("SELECT * FROM experiments WHERE id = ?", id);
}

// 호출
Experiment exp = getExperiment("E1");  // 바로 사용
```

**현실:**
- 부수 효과를 처리하기 위한 별도의 패턴 필요 (Monad, Effect Systems)
- 코드가 여러 계층으로 감싸집니다
- 성숙도 낮은 팀이 이해하기 어렵습니다

### 단점 8: 점진적 마이그레이션의 어려움

**기존 시스템을 함수형으로 변환하기 어렵습니다:**

```
OOP 시스템 → 함수형 시스템 마이그레이션:
- 한 메서드씩 수정하기 어려움
- 모든 타입을 다시 정의해야 함
- 클래스 기반 라이브러리와 상호 작용 복잡
```

**현실:**
- 기존 코드베이스가 클래스로 가득 차 있다면?
- 마이그레이션 비용이 엄청날 수 있습니다
- 혼합된 패러다임(Hybrid)은 더 복잡해집니다

---

## 10. 언제 어떤 패러다임을 사용할 것인가?

### 함수형이 좋은 경우

✅ **함수형 프로그래밍을 선택하세요:**

1. **복잡한 상태 관리가 필요한 경우**
   - 여러 상태가 불가능한 조합을 형성할 수 있을 때
   - AI 신약 설계 (우리의 경우!)
   - 금융 시스템, 주문 처리 시스템

2. **타입 안전성이 중요한 경우**
   - 컴파일 타임 오류 감지가 필수일 때
   - 미션-크리티컬 시스템
   - 정확성이 비용보다 중요할 때

3. **테스트 용이성이 중요한 경우**
   - DB 없이 테스트해야 할 때
   - 초기 개발 단계에서 빠른 피드백이 필요할 때

4. **도메인이 명확하고 복잡한 경우**
   - 도메인 전문가와의 언어 일치가 가능할 때
   - 장기적 유지보수가 중요할 때

### OOP가 좋은 경우

✅ **OOP를 선택하세요:**

1. **빠른 프로토타이핑이 필요한 경우**
   - 초기 스타트업 MVP
   - 요구사항이 자주 변할 때
   - "일단 돌아가는 것"이 중요할 때

2. **팀이 OOP 경험이 풍부한 경우**
   - 새로운 패러다임 학습 비용 > 장기 이득
   - 프로젝트 기간이 짧을 때

3. **성능이 극도로 중요한 경우**
   - 대규모 데이터 처리 (1억 개 이상)
   - 실시간 시스템
   - 임베디드 시스템

4. **생태계가 OOP 중심일 때**
   - 대부분의 필수 라이브러리가 클래스 기반
   - 레거시 시스템과 통합 필요

### 우리의 상황: AI 신약 설계 플랫폼

| 기준 | 평가 | 권장도 |
| :--- | :--- | :--- |
| **상태 관리 복잡도** | 매우 높음 (여러 단계, 상태 전환) | ✅✅✅ FP |
| **타입 안전성 중요도** | 높음 (잘못된 계산 = 손실) | ✅✅✅ FP |
| **도메인 명확성** | 높음 (연구원과 대화 가능) | ✅✅✅ FP |
| **장기 유지보수** | 높음 (수년간 사용될 예정) | ✅✅✅ FP |
| **초기 학습곡선** | 높음 (팀이 함수형 미경험) | ❌ OOP |
| **프로토타이핑 속도** | 중요 (빠른 검증 필요) | ❌ OOP |
| **팀 규모** | 소규모 (함수형 교육 가능) | ✅ FP |

**결론: 함수형이 장기적으로 최적입니다. 다만 초기 학습 투자가 필요합니다.**

---

## 11. 균형잡힌 접근: Hybrid 패러다임

**실제로는 순수한 함수형도, 순수한 OOP도 아닌 "Hybrid"를 많이 씁니다:**

```typescript
// 상태 관리 계층: 함수형 (순수 함수, 불변성)
type Experiment = 
    | { type: 'InProgress'; ... }
    | { type: 'Completed'; ... };

const completeExperiment = (exp: Experiment): Experiment => { ... };

// 데이터 접근 계층: OOP (클래스 인터페이스)
class ExperimentRepository {
    async getById(id: string): Promise<Experiment> { ... }
    async save(exp: Experiment): Promise<void> { ... }
}

// 애플리케이션 계층: 함수형 + OOP 섞임
async function processExperiment(id: string, repo: ExperimentRepository) {
    const exp = await repo.getById(id);           // OOP 스타일
    const completed = completeExperiment(exp);    // FP 스타일
    await repo.save(completed);                   // OOP 스타일
}
```

**이렇게 하면:**
- ✅ 핵심 도메인 로직은 함수형으로 안전하게
- ✅ 인프라 계층은 OOP의 편의성 활용
- ✅ 학습 곡선을 완만하게 유지
- ✅ 팀의 부담을 줄이면서도 이점 획득

---

## 최종 결론

**함수형 프로그래밍의 단점을 무시할 수 없습니다:**
- 초기 학습 곡선이 가파릅니다
- 성능 오버헤드가 있을 수 있습니다
- 복잡한 상태 관리 시 더 복잡할 수 있습니다
- 팀 문화의 변화가 필요합니다

**하지만 우리 프로젝트에서는 이점이 훨씬 큽니다:**
- 복잡한 도메인 상태를 안전하게 표현 가능
- 타입 시스템으로 불가능한 상태 원천 차단
- 장기적 유지보수성 향상
- 도메인 전문가와의 명확한 소통

**권장사항:**
1. **핵심 도메인 로직(상태 관리)은 함수형으로**
2. **인프라 계층(DB, API)은 필요에 따라 OOP 활용**
3. **점진적으로 팀 역량 강화**
4. **초기에는 Hybrid 패러다임으로 시작**

결국 **도구는 수단**이고, **문제 해결**이 목표입니다. 우리의 목표인 "AI 신약 설계 로직을 정확하고 명확하게 구현하는 것"을 위해 함수형이 더 적합한 선택입니다.

