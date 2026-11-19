# 핵심 객체 역할 정의서 — 회사 연차관리 시스템

## 0. 설계 결정 요약
| 항목 | 결정 |
| --- | --- |
| 연차 부여 기준 | 회계연도(매년 1월 1일) 일괄 부여 |
| 출근율 반영 | 반영함 (결근 기록 기반, 개근 가정하지 않음) |
| 연차 단위 | double, 0.5 단위로만 사용 |
| 입사 다음 해 비례 부여 | 0.5 단위 올림 (예: 6.2일 → 6.5일) |
| 병가 | 신청·승인 절차는 거치되 잔여 연차에서 차감하지 않음 |
| 신청 시작일 | 오늘 이전 불가, 오늘은 허용 |
| 직급 | Position enum (사원, 팀장, 관리자) |

## 1. 객체 도출 과정

### 1-1. 요구사항에서 명사·동사 추출
| FR | 요구사항 문장 | 명사 (객체/속성 후보) | 동사 (행위 후보) |
| --- | --- | --- | --- |
| FR-01 | 관리자가 사번, 이름, 부서, 직급, 입사일, 상사를 입력해 직원을 등록한다 | 관리자, 직원, 사번, 이름, 부서, 직급, 입사일, 상사 | 입력한다, 등록한다 |
| FR-02 | 사번으로 직원 정보를 조회한다 | 사번, 직원 정보 | 조회한다 |
| FR-03 | 입사일과 기준일로 해당 연도 연차를 계산해 부여한다 | 입사일, 기준일, 연도, 연차 | 계산한다, 부여한다 |
| FR-04 | 직원이 종류, 시작일, 종료일, 사유를 입력해 연차를 신청한다 | 직원, 종류, 시작일, 종료일, 사유, 연차 신청 | 입력한다, 신청한다 |
| FR-05 | 직원이 대기 또는 승인 상태의 본인 신청을 취소한다 | 직원, 상태, 신청 | 취소한다 |
| FR-06 | 팀장이 본인 팀원의 대기 중 신청을 조회한다 | 팀장, 팀원, 신청 | 조회한다 |
| FR-07 | 팀장이 대기 중 신청을 승인 또는 반려하고, 승인되면 잔여 연차가 차감된다 | 팀장, 신청, 잔여 연차 | 승인한다, 반려한다, 차감한다 |
| FR-08 | 직원이 총 연차, 사용 연차, 잔여 연차를 조회한다 | 총 연차, 사용 연차, 잔여 연차 | 조회한다 |
| FR-09 | 직원이 본인 신청 목록과 상태를 조회한다 | 신청 목록, 상태 | 조회한다 |
| FR-10 | 오전/오후 반차를 0.5일로 처리한다 | 반차, 일수 | 처리한다 |
| FR-11 | 기간이 겹치는 신청을 막는다 | 기간, 신청 | 검사한다 |
| FR-12 | 사용 일수 계산 시 주말을 제외한다 | 사용 일수, 주말 | 계산한다, 제외한다 |
| FR-13 | 관리자가 부서별 연차 사용률을 조회한다 | 부서, 사용률 | 조회한다, 집계한다 |
| FR-14 | 관리자가 직원별 결근일을 날짜로 등록한다 | 결근일, 날짜 | 등록한다 |
| FR-15 | 매년 1월 1일에 출근율과 근속연수로 전 직원 연차를 일괄 부여한다 | 회계연도, 출근율, 근속연수, 연차 | 계산한다, 부여한다 |

### 1-2. 후보 분류

#### 명사 분류
| 후보 | 분류 | 결과 | 이유 |
| --- | --- | --- | --- |
| 직원 | 객체 | Employee | 정보를 가지고 근속연수 계산·권한 판단을 한다 |
| 결근일 | 객체 | AttendanceRecord | 날짜를 모아 출근율·개근 월 수를 계산한다 |
| 잔여·총·사용 연차 | 객체 | LeaveBalance | 세 값이 함께 움직이고 차감·복구 규칙을 지켜야 한다 |
| 연차 신청 | 객체 | LeaveRequest | 상태가 바뀌고 전이 규칙을 지켜야 한다 |
| 연차 계산 규칙 | 정책 인터페이스 | LeavePolicy | 회사마다 바뀔 수 있는 규칙이다 |
| 종류, 반차 | enum | LeaveType | 정해진 4개 중 하나이고 차감 단위를 가진다 |
| 상태 | enum | RequestStatus | 정해진 4개 중 하나 |
| 직급 | enum | Position | 사원/팀장/관리자 중 하나 |
| 관리자, 팀장, 팀원 | Employee의 직급·상사 관계 | — | 행위가 같고 권한만 다르다 |
| 사번, 이름, 부서, 입사일, 상사 | Employee 속성 | — | 단순 값 또는 참조 |
| 시작일, 종료일, 사유 | LeaveRequest 속성 | — | 단순 값 |
| 기준일, 연도 | 파라미터 / LeaveBalance 속성 | — | 계산할 때 넘기는 값 |
| 사용 일수 | LeaveRequest가 계산 | — | 기간과 종류로 계산된다 |
| 부서별 사용률, 신청 목록 | 조회 결과 | — | 저장하지 않고 조회 시 계산한다 |

#### 동사 분류
| 동사 | 담당 객체 | 이유 |
| --- | --- | --- |
| 근속연수를 계산한다 | Employee | 입사일을 가진 객체가 계산한다 |
| 출근율·개근 월 수를 계산한다 | AttendanceRecord | 결근일을 가진 객체가 계산한다 |
| 연차 일수를 계산한다 | LeavePolicy | 여러 객체 정보를 조합하는 규칙이다 |
| 차감한다 / 복구한다 | LeaveBalance | 잔액을 가진 객체만 바꿀 수 있다 |
| 승인한다 / 반려한다 / 취소한다 | LeaveRequest | 자기 상태는 스스로 바꾼다 |
| 사용 일수를 계산한다 | LeaveRequest | 기간과 종류를 가진 객체가 계산한다 |
| 기간 겹침을 검사한다 | LeaveRequest | 자기 기간과 다른 기간을 비교한다 (확장) |
| 등록한다 / 조회한다 | Repository | 저장과 조회만 담당한다 |
| 부여한다 / 신청한다 (흐름 전체) | LeaveService | 여러 객체를 순서대로 호출한다 |
| 집계한다 | LeaveService | 여러 직원 데이터를 모은다 (확장) |

#### 도출 결과
- 핵심 객체: **Employee, LeaveRequest, LeaveBalance**
- 보조 객체: AttendanceRecord, LeavePolicy
- enum: LeaveType, RequestStatus, Position
- 조율·저장: LeaveService, Repository 4종

## 2. 객체별 역할 정의

### 2-1. Employee (직원) — 핵심 객체
| 항목 | 내용 |
| --- | --- |
| 책임 | 자신의 신원과 입사일을 알고, 기준일 시점의 근속연수와 결재 권한을 판단한다 |
| 주요 속성 | 사번(String), 이름(String), 부서(String), 직급(Position), 입사일(LocalDate), 상사(Employee) |
| 주요 행위 | getYearsOfService(기준일): 근속연수 / isApprover(): 팀장 이상인지 / isManagerOf(직원): 해당 직원의 상사인지 |
| 협력 객체 | LeavePolicy가 근속연수를 묻는다. LeaveRequest가 신청자·결재자로 참조한다. LeaveService가 결재 권한을 확인한다 |
| 제약 및 규칙 | 이름은 비어 있을 수 없다. 입사일은 미래일 수 없다. 사번은 변경할 수 없다. 사원은 결재할 수 없다 |
| 요구사항 연결 | FR-01, FR-02 |

### 2-2. AttendanceRecord (근태 기록)
| 항목 | 내용 |
| --- | --- |
| 책임 | 한 직원의 결근일을 보관하고, 기간별 출근율과 개근 월 수를 계산한다 |
| 주요 속성 | 직원(Employee), 결근일 목록(Set<LocalDate>) |
| 주요 행위 | addAbsence(날짜): 결근 등록 / getAttendanceRate(시작일, 종료일): 출근율 / countPerfectMonths(시작일, 종료일): 개근한 달 수 |
| 협력 객체 | LeavePolicy가 출근율과 개근 월 수를 묻는다. LeaveService가 결근 등록을 요청한다 |
| 제약 및 규칙 | 결근일은 입사일 이전이거나 미래일 수 없다. 같은 날짜는 한 번만 저장된다. 소정근로일은 평일만 센다. 출근율 = (소정근로일 − 결근일) / 소정근로일 |
| 요구사항 연결 | FR-14 |

### 2-3. LeaveBalance (연차 잔액) — 핵심 객체
| 항목 | 내용 |
| --- | --- |
| 책임 | 한 직원의 특정 연도 연차 총량과 사용량을 정확하게 유지한다 |
| 주요 속성 | 직원(Employee), 연도(int), 총 연차(double), 사용 연차(double) |
| 주요 행위 | deduct(일수): 차감 / restore(일수): 복구 / getRemaining(): 잔여 / canUse(일수): 사용 가능 여부 / addMonthlyLeave(): 입사 첫해 월 1일 추가 |
| 협력 객체 | LeavePolicy의 계산 결과로 생성된다. LeaveService가 차감·복구를 요청한다 |
| 제약 및 규칙 | 모든 일수는 0.5의 배수다. 잔여보다 많이 차감할 수 없다. 사용량보다 많이 복구할 수 없다. 0 이하 값은 받지 않는다. setter가 없다. 입사 첫해 월 발생분은 11일을 넘지 않는다 |
| 요구사항 연결 | FR-03, FR-08 |

### 2-4. LeaveRequest (연차 신청) — 핵심 객체
| 항목 | 내용 |
| --- | --- |
| 책임 | 신청 한 건의 내용을 보관하고, 허용된 상태 전이만 일어나도록 스스로 관리한다 |
| 주요 속성 | 신청번호(long), 신청자(Employee), 종류(LeaveType), 시작일(LocalDate), 종료일(LocalDate), 사유(String), 상태(RequestStatus), 결재자(Employee) |
| 주요 행위 | approve(결재자): 승인 / reject(결재자): 반려 / cancel(요청자): 취소 / getUsedDays(): 사용 일수 / isDeductible(): 잔여 차감 대상인지 (병가는 false) |
| 협력 객체 | LeaveService가 생성하고 상태 변경을 요청한다. LeaveType에 차감 단위와 차감 여부를 묻는다. Employee에 결재 권한을 묻는다 |
| 제약 및 규칙 | 생성 시 상태는 항상 대기. 시작일은 오늘 이전일 수 없다(오늘은 허용). 시작일은 종료일보다 뒤일 수 없다. 반차는 시작일과 종료일이 같다. 대기 상태에서만 승인·반려할 수 있다. 대기·승인 상태에서만 취소할 수 있다. 본인 신청은 본인이 승인할 수 없다. 결재자는 신청자의 상사여야 한다 |
| 요구사항 연결 | FR-04, FR-05, FR-07, FR-09 |

### 2-5. LeavePolicy (연차 정책) — 인터페이스
| 항목 | 내용 |
| --- | --- |
| 책임 | 직원, 근태 기록, 부여 기준일(1월 1일)을 받아 그해 부여할 연차 일수를 결정한다 |
| 주요 행위 | calculateAnnualLeave(직원, 근태 기록, 기준일): 부여 일수(double) |
| 구현체 | FiscalYearLeavePolicy: 회계연도 기준 / (선택) FixedLeavePolicy: 모두 15일 고정, 다형성 시연용 |
| 계산 규칙 | 입사 다음 해 1월 1일: 전년도 출근율 80% 이상이면 15 × (전년도 재직일수 / 365)를 0.5 단위 올림 / 그 이후: 전년도 출근율 80% 이상이면 15 + (근속연수 − 1) / 2, 최대 25일 / 출근율 80% 미만이면 전년도 개근 월 수 × 1일 |
| 협력 객체 | Employee에 근속연수를, AttendanceRecord에 출근율·개근 월 수를 묻는다. LeaveService가 호출한다 |
| 제약 및 규칙 | 결과는 0 이상 25 이하이고 0.5의 배수다 |
| 요구사항 연결 | FR-03, FR-15 |

### 2-6. LeaveType (연차 종류) — enum
| 상수 | 표시 이름 | 차감 단위 | 차감 여부 |
| --- | --- | --- | --- |
| ANNUAL | 연차 | 1.0일 | 차감 |
| HALF_AM | 오전 반차 | 0.5일 | 차감 |
| HALF_PM | 오후 반차 | 0.5일 | 차감 |
| SICK | 병가 | 1.0일 | 미차감 |

- 책임: 종류별 차감 단위와 차감 여부를 안다.
- 요구사항 연결: FR-04, FR-10

### 2-7. RequestStatus (신청 상태) — enum
| 상수 | 의미 | 다음으로 갈 수 있는 상태 |
| --- | --- | --- |
| PENDING | 대기 | APPROVED, REJECTED, CANCELED |
| APPROVED | 승인 | CANCELED |
| REJECTED | 반려 | 없음 |
| CANCELED | 취소 | 없음 |

- 책임: 상태 목록과 허용된 다음 상태를 정의한다.
- 행위: canChangeTo(다음 상태): 전이 가능 여부

### 2-8. Position (직급) — enum
| 상수 | 의미 | 결재 권한 |
| --- | --- | --- |
| STAFF | 사원 | 없음 |
| TEAM_LEADER | 팀장 | 소속 팀원 신청 결재 |
| ADMIN | 관리자 | 직원 등록, 결근 등록, 연차 부여, 통계 조회 |

- 책임: 직급별 권한 여부를 안다.

### 2-9. LeaveService (연차 서비스)
| 항목 | 내용 |
| --- | --- |
| 책임 | 유스케이스 단위로 객체를 순서대로 호출해 흐름을 조율한다. 직접 계산하지 않는다 |
| 주요 속성 | 각 Repository, LeavePolicy (생성자 주입) |
| 주요 행위 | registerEmployee / registerAbsence / grantAnnualLeave(연도) / apply / cancel / approve / reject / getPendingRequests(팀장) / getMyBalance / getDepartmentStats |
| 협력 객체 | 모든 도메인 객체와 Repository |
| 제약 및 규칙 | 검증과 계산은 도메인 객체에 위임한다. 메서드는 "조회 → 도메인에 시키기 → 저장" 흐름만 가진다. 병가는 차감·복구 단계를 건너뛴다 |
| 요구사항 연결 | 전체 FR의 진입점 |

### 2-10. Repository (저장소)
| 인터페이스 | 책임 | 주요 행위 |
| --- | --- | --- |
| EmployeeRepository | 직원 저장·조회 | save, findById, findAll, findByManager |
| LeaveRequestRepository | 신청 저장·조회 | save, findById, findByApplicant, findPendingByManager |
| LeaveBalanceRepository | 잔액 저장·조회 | save, findByEmployeeAndYear |
| AttendanceRepository | 근태 기록 저장·조회 | save, findByEmployee |

- 구현체: 각각 메모리(Map) 기반 구현 1개씩
- 이유: 저장 방식을 파일·DB로 바꿔도 Service가 바뀌지 않도록 한다

## 3. 객체 간 협력 요약
| 호출하는 객체 | 호출받는 객체 | 메시지 |
| --- | --- | --- |
| LeaveService | LeavePolicy | 연차 일수 계산해 줘 |
| LeavePolicy | Employee | 근속연수 알려 줘 |
| LeavePolicy | AttendanceRecord | 출근율·개근 월 수 알려 줘 |
| LeaveService | LeaveBalance | 사용 가능해? / 차감해 / 복구해 |
| LeaveService | LeaveRequest | 승인해 / 반려해 / 취소해 / 차감 대상이야? / 사용 일수 알려 줘 |
| LeaveRequest | LeaveType | 차감 단위와 차감 여부 알려 줘 |
| LeaveRequest | RequestStatus | 이 상태로 바꿀 수 있어? |
| LeaveRequest | Employee | 결재 권한 있어? 신청자의 상사야? |
| Employee | Position | 결재 권한 있는 직급이야? |
| LeaveService | 각 Repository | 저장해 / 찾아 줘 |

## 4. 요구사항 추적표
| FR | 담당 객체 |
| --- | --- |
| FR-01, FR-02 | Employee, Position, EmployeeRepository |
| FR-03, FR-15 | LeavePolicy, LeaveBalance |
| FR-04, FR-05 | LeaveRequest, LeaveType |
| FR-06, FR-07 | LeaveRequest, RequestStatus, LeaveService |
| FR-08 | LeaveBalance |
| FR-09 | LeaveRequestRepository |
| FR-10 | LeaveType |
| FR-11, FR-12 | LeaveRequest (확장) |
| FR-13 | LeaveService (확장) |
| FR-14 | AttendanceRecord |