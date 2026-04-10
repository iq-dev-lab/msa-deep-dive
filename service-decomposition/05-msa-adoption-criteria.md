# 05. MSA 도입 판단 기준 — Conway's Law와 팀 구조

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Conway's Law란 무엇이며, 왜 아키텍처가 조직 구조를 따라가는가?
- 팀 규모별로 어떤 아키텍처가 권장되는가?
- Strangler Fig 패턴으로 모놀리스를 MSA로 어떻게 점진적으로 전환하는가?
- "모놀리스 우선" 전략이 왜 유효한가?
- MSA 전환을 시작할 적절한 시점을 판단하는 기준은 무엇인가?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

MSA 전환은 기술 결정이 아니라 조직 결정이다. 아무리 좋은 기술 아키텍처도 조직 구조와 맞지 않으면 실패한다. Conway's Law는 이것을 역설한다: "시스템 설계는 그것을 만든 조직의 커뮤니케이션 구조를 닮는다." 아키텍처를 바꾸려면 조직도 함께 바꿔야 하며, 역으로 조직 구조가 아키텍처를 강제한다.

---

## 😱 흔한 실수 (Before — Conway's Law를 무시할 때)

```
상황: CTO의 MSA 전환 결정

조직 구조: 기능별 팀 (프론트팀, 백엔드팀, DB팀, 인프라팀)
  모든 백엔드 개발자가 하나의 팀에서 함께 작업

MSA 전환 시도:
  백엔드팀이 주문/결제/배송 서비스를 나눔

  그런데:
  - 주문과 결제 코드를 같은 팀원이 수정
  - 서비스 경계를 자유롭게 넘나드는 코드 변경
  - API 계약 없이 내부적으로 직접 호출
  - "어차피 우리 팀이 다 하니까" 동시 배포

결과: 코드만 분리됐고 팀은 분리되지 않음 → 분산 모놀리스
```

---

## ✨ 올바른 접근 (After — Inverse Conway Maneuver)

```
Inverse Conway Maneuver:
  원하는 시스템 아키텍처에 맞게 먼저 팀을 재구성한다

목표 아키텍처: 주문 서비스 / 결제 서비스 / 배송 서비스

팀 재구성:
  주문팀 (4명): 주문 서비스 전담
  결제팀 (3명): 결제 서비스 전담
  배송팀 (3명): 배송 서비스 전담

  각 팀이 자신의 서비스를 독립적으로 설계, 개발, 배포, 운영
  팀 간 커뮤니케이션 = API 계약

결과: 팀 경계 = 서비스 경계 → 자연스럽게 독립 배포 달성
```

---

## 🔬 내부 동작 원리 — Conway's Law 심층 분석

### Conway's Law

```
Melvin Conway (1967):
"Any organization that designs a system will produce a design
 whose structure is a copy of the organization's communication structure."

실제 예시:

조직 A (기능별 팀):
  UI팀, 백엔드팀, DB팀, 인프라팀
  → 3계층 아키텍처 (Presentation → Business Logic → Data)

조직 B (도메인별 팀):
  주문팀, 결제팀, 배송팀, 상품팀
  → MSA (팀과 서비스가 1:1 대응)

함의:
  기술 아키텍처를 먼저 설계해도 조직이 바뀌지 않으면 원래로 돌아감
```

### 팀 규모별 권장 아키텍처

```
1~10명: 모놀리스
  이유: MSA 운영 인프라를 전담할 인력 없음
       빠른 피벗이 필요한 시기
       서비스 경계가 아직 불명확

10~50명: 모듈러 모놀리스
  이유: 팀이 커지면서 코드 충돌 증가
       모듈 경계로 팀 간 작업 영역 분리
       아직 MSA 운영 복잡도 감당 어려움

50명 이상 + 도메인별 팀: MSA 검토
  전제 조건:
  - 도메인 경계가 명확히 정립됨
  - CI/CD 자동화 성숙
  - 인프라 팀 (또는 플랫폼 팀) 존재
  - 분산 시스템 경험 보유
```

---

## 💻 Strangler Fig 패턴 — 점진적 MSA 전환

```
1단계: 라우팅 레이어 도입 (Façade)

클라이언트 → API Gateway
              └── 모든 요청 → 모놀리스 (기존)

2단계: 기능 단위로 서비스 추출 (독립성 높은 것부터)

클라이언트 → API Gateway
              ├── /api/products/** → 상품 서비스 (신규) ✅
              └── 나머지 → 모놀리스 (유지)

추출 순서 원칙:
  - 다른 도메인 의존성이 가장 낮은 것부터
  - 변경 빈도가 높은 것부터

3단계: 점진적으로 모놀리스 기능 제거

클라이언트 → API Gateway
              ├── /api/products/**  → 상품 서비스 ✅
              ├── /api/orders/**    → 주문 서비스 ✅
              ├── /api/payments/**  → 결제 서비스 ✅
              └── (모놀리스 종료)
```

### 이중 쓰기(Dual Write)로 데이터 이관

```java
@Service
public class ProductMigrationService {

    private final LegacyProductRepository legacyRepo;
    private final NewProductRepository newRepo;

    @Transactional
    public Product createProduct(CreateProductCommand cmd) {
        // 모놀리스 DB에 쓰기 (기존)
        LegacyProduct legacy = legacyRepo.save(LegacyProduct.from(cmd));

        // 새 상품 서비스 DB에도 쓰기 (신규)
        try {
            newRepo.save(NewProduct.from(cmd));
        } catch (Exception e) {
            log.error("New product DB write failed", e);
            // 배치로 동기화 보완
        }

        return Product.from(legacy);
    }
    // 이중 쓰기 안정화 → 모놀리스 쓰기 제거 → 신규 서비스만 남음
}
```

---

## 📊 패턴 비교 — MSA 전환 전략

| 전략 | 설명 | 리스크 | 권장 상황 |
|------|------|--------|---------|
| Big Bang | 모놀리스를 한 번에 MSA로 교체 | 매우 높음 | 비권장 |
| Strangler Fig | 기능을 하나씩 추출 | 낮음 | 대부분의 경우 |
| Modular Monolith First | 모듈화 후 필요시 추출 | 낮음 | 도메인 불명확 시 |

---

## ⚖️ 트레이드오프

```
모놀리스 우선 전략:
  ✅ 도메인 경계를 충분히 탐색 후 분리 → 경계 실수 비용 낮음
  ✅ 초기 운영 복잡도 낮음
  ❌ 모놀리스가 너무 커지기 전에 분리 시작점을 놓칠 수 있음

MSA 우선 전략:
  ✅ 처음부터 독립 배포 경험
  ❌ 잘못된 경계 설정 비용 큼 (재조정 시 분산 데이터 마이그레이션)
  ❌ 작은 팀이 운영 복잡도에 압도됨

결론: 대부분의 경우 "모놀리스 시작 → Strangler Fig 전환"이 안전
```

---

## 📌 핵심 정리

```
✅ Conway's Law
   "조직의 커뮤니케이션 구조가 시스템 구조를 결정한다"
   → 팀 구조를 바꾸지 않고 아키텍처만 바꾸면 원래로 돌아감

✅ Inverse Conway Maneuver
   원하는 아키텍처에 맞게 먼저 팀을 재구성

✅ 팀 규모별 권장
   1~10명: 모놀리스
   10~50명: 모듈러 모놀리스
   50명 이상 + 도메인팀: MSA 검토

✅ Strangler Fig 패턴
   1. API Gateway로 라우팅 레이어 도입
   2. 독립성 높은 서비스부터 순차 추출
   3. 이중 쓰기로 데이터 이관
   4. 모놀리스 점진적 제거
```

---

## 🤔 생각해볼 문제

**Q1.** "우리 팀 5명인데 MSA가 필요하다"는 주장에 어떻게 반응하겠는가?

<details>
<summary>해설 보기</summary>

MSA가 필요하다고 느끼는 실제 문제가 무엇인지 파악한다. 팀이 5명이라면 MSA 운영 복잡도를 전담할 인력이 없다. 대신 모듈러 모놀리스를 권장한다. 패키지를 도메인 경계로 명확히 나누고, 모듈 간 직접 의존을 금지하며, 인터페이스로만 통신하면 나중에 서비스로 추출하기 쉬운 구조가 된다.

</details>

**Q2.** 기능별 팀(프론트팀/백엔드팀/DB팀) 구조에서 MSA로 전환하려면 무엇을 먼저 해야 하는가?

<details>
<summary>해설 보기</summary>

기술 전환보다 팀 구조 전환이 먼저다. Inverse Conway Maneuver를 적용한다. 도메인별 팀(주문팀, 결제팀, 상품팀)으로 재편하고, 각 팀이 자신의 서비스 전체(백엔드 + DB)를 소유하게 한다. 팀 재편 없이 기술만 MSA로 바꾸면 기능별 팀이 서비스 경계를 넘나들어 분산 모놀리스가 된다.

</details>

**Q3.** Strangler Fig 전환 중 모놀리스와 신규 서비스 사이에서 데이터를 어떻게 동기화하는가?

<details>
<summary>해설 보기</summary>

이중 쓰기(Dual Write) 전략을 사용한다. 모든 쓰기 작업을 모놀리스 DB와 신규 서비스 DB 양쪽에 동시에 수행한다. 신규 DB 쓰기 실패는 로깅 후 배치 동기화로 보완한다. 두 DB의 데이터가 일치함을 확인하면, 모놀리스 쓰기를 제거하고 신규 서비스만 남긴다. Kafka Outbox Pattern으로 이벤트 기반 동기화도 가능하다.

</details>

---

<div align="center">

**[⬅️ 이전: 서비스 분해 실전](./04-decomposition-in-practice.md)** | **[홈으로 🏠](../README.md)** | **[다음: 서비스 자율성 원칙 ➡️](./06-service-autonomy-principles.md)**

</div>
