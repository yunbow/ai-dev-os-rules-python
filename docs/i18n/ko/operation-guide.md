# 운영 및 기여 가이드

이 문서는 ai-dev-os의 업데이트 및 확장에 대한 운영 규칙을 정의합니다.

---

## 1. 섹션별 업데이트 정책

### 1.1 업데이트 빈도와 영향 범위

| 섹션 | 업데이트 빈도 | 영향 범위 | 주의사항 |
|------|-------------|---------|---------|
| `01_philosophy/` | 매우 드뭄 | 모든 프로젝트 | 핵심 사상 변경은 MAJOR 버전업 필요 |
| `02_decision-criteria/` | 드뭄 | 모든 프로젝트 | 판단 기준 변경 시 하위 가이드라인과의 정합성 확인 필요 |
| `03_guidelines/common/` | 중간 | 모든 프로젝트 | 공통 기준 변경 시 모든 프레임워크에 대한 영향 검증 필요 |
| `03_guidelines/frameworks/` | 높음 | 해당 FW 프로젝트 | 프레임워크 업데이트 추적 |
| `04_ai-prompts/` | 중간 | AI 어시스턴트 사용자 | 프롬프트 개선은 자유롭게 가능 |
| `templates/` | 중간 | 새 프로젝트만 | 기존 프로젝트에 영향 없음 |

### 1.2 업데이트 규칙

**모든 섹션 공통:**
- 프로젝트 고유 기술을 포함하지 않음 (특정 서비스명, 특정 도메인 용어)
- 구체적인 예가 필요한 경우 `{domain}`, `{Payment Service}` 등의 플레이스홀더를 사용한 범용 예시 사용
- 변경 후 `README.md`의 디렉토리 구조를 최신 상태로 업데이트

**언어 정책:**
- `01_philosophy/`와 `02_decision-criteria/`에는 **영어 샘플 콘텐츠**가 포함되어 있습니다 — 클론 후 **모국어로 다시 작성하세요** (추상적 사고와 의사결정 프레임워크는 뉘앙스를 보존하기 위해 모국어로 작성하는 것이 최선)
- 그 외 모든 섹션(`03_guidelines/`, `04_ai-prompts/`, `templates/`)은 **영어**로 작성 필수 — AI 호환성 및 국제적 접근성을 위해
- 다국어 운영 가이드는 `docs/i18n/`에서 유지 (JA, ZH, KO, ES)
- 콘텐츠를 추가하거나 업데이트할 때 항상 이 언어 정책을 준수

**`01_philosophy/` 업데이트:**
- 샘플 콘텐츠(영어)가 포함되어 있습니다. 클론 후 **모국어로 다시 작성하세요**
- 새로운 원칙을 추가할 때 기존 원칙과 모순이 없는지 확인
- 삭제 및 대폭 변경은 MAJOR 버전으로 취급
- `02_decision-criteria/` 이하와의 정합성도 확인

**`02_decision-criteria/` 업데이트:**
- 샘플 콘텐츠(영어)가 포함되어 있습니다. 클론 후 **모국어로 다시 작성하세요**
- 판단 기준 변경 시 `03_guidelines/`의 해당 섹션도 업데이트가 필요한지 확인
- 새로운 판단 축을 추가할 때 대응하는 가이드라인이 있는지 확인

**`03_guidelines/` 업데이트:**
- `common/`과 `frameworks/` 간의 콘텐츠 중복에 주의
- 공유해야 할 콘텐츠가 프레임워크별 파일에만 작성되어 있지 않은지 정기적으로 검토
- 자세한 내용은 "2. 프레임워크 가이드라인 추가" 참조

**`04_ai-prompts/` 업데이트:**
- 프롬프트에서 참조하는 가이드라인 경로가 실제로 존재하는지 확인
- `skills/`는 Claude Code SKILL.md 형식(frontmatter + procedures)을 유지해야 함

**`templates/` 업데이트:**
- 템플릿은 새 프로젝트용. 기존 프로젝트에 자동 적용하지 않음
- 설정 파일(pyproject.toml, ruff 설정 등)은 `03_guidelines/` 기준과 일치해야 함

---

## 2. 프레임워크 가이드라인 추가

### 2.1 추가 기준

새 프레임워크 가이드라인 추가 조건:

| 조건 | 필수/권장 |
|------|----------|
| 2개 이상의 프로젝트에서 사용 | 필수 |
| `common/`으로 표현할 수 없는 프레임워크 고유 패턴 | 필수 |
| 프레임워크가 안정 릴리스(v1.0+) | 권장 |
| 장기 유지보수가 예상됨 | 권장 |

### 2.2 디렉토리 구조

```
03_guidelines/frameworks/{framework-name}/
├── overview.md            # 기술 스택 정의 (필수)
├── project-structure.md   # 디렉토리 구조 (필수)
└── ...                    # 프레임워크별 가이드라인
```

### 2.3 추가 단계

#### Step 1: overview.md 생성

모든 프레임워크 가이드라인의 필수 진입점. 포함 내용:

```markdown
# {Framework} Technology Stack

## Core
- Framework: {Name} v{Version}
- Language: Python (타입 힌트 사용, mypy/pyright strict 모드)
- Package Manager: {uv / pip / poetry}

## Recommended Libraries
{카테고리별 권장 라이브러리 목록}

## Prerequisites
{common/ 가이드라인과의 관계 명시}
```

#### Step 2: project-structure.md 생성

디렉토리 구조 가이드라인. 다음 원칙을 따름:

- `common/`에서 이미 정의된 개념(수직 슬라이스, 의존성 규칙 등)을 그대로 적용
- 프레임워크 고유의 디렉토리 규칙만 기술
- 구체적인 예시에는 `{domain}` 등의 플레이스홀더 사용

#### Step 3: 프레임워크별 가이드라인 추가

`common/` 대응 가이드라인과의 관계:

| 패턴 | 접근 방법 |
|------|----------|
| 공통 내용을 그대로 적용 | 프레임워크 측에 파일을 생성하지 않음 |
| 공통을 확장하는 내용 | 프레임워크 측에 생성, 공통을 참조하고 차이만 기술 |
| 프레임워크 고유 개념 | 프레임워크 측에만 생성 |

**파일 명명 규칙:**
- `common/`과 대응할 때 동일한 이름 사용 (예: `common/security.md` → `python-cli/security.md`)
- 프레임워크 고유 개념은 설명적인 이름 사용 (예: `dependency-injection.md`, `cli-commands.md`)

#### Step 4: 템플릿 생성

```
templates/{framework-name}/
├── CLAUDE.md.template     # CLAUDE.md 템플릿 (필수)
├── submodule-setup.sh     # 설정 스크립트 (권장)
├── .claude/skills/        # Claude Code skills (권장)
└── {configuration files}  # pyproject.toml, ruff 설정 등
```

#### Step 5: README.md 업데이트

- 디렉토리 구조에 새 프레임워크 추가
- 필요한 경우 소개 섹션에 프레임워크별 지침 추가

### 2.4 common/과의 책임 분리 원칙

```
common/          → "무엇을 할 것인가" (언어/FW에 독립적인 규칙)
frameworks/xxx/  → "어떻게 구현할 것인가" (FW별 구현 패턴)
```

**예시: 유효성 검증**
- `common/validation.md` → "서버 측 유효성 검증 필수" "Pydantic 권장"
- `frameworks/python-cli/validation.md` → "Pydantic 모델과 Typer 인수 파싱을 사용한 유효성 검증 패턴"
- `frameworks/fastapi/validation.md` → "Pydantic 모델과 의존성 주입을 사용한 요청 본문 유효성 검증"

**예시: 에러 처리**
- `common/error-handling.md` → "에러 분류" "사용자 표시 원칙"
- `frameworks/python-cli/error-handling.md` → "커스텀 예외 계층과 Typer 종료 코드"

### 2.5 추가 체크리스트

- [ ] overview.md와 project-structure.md가 생성됨
- [ ] 프로젝트 고유 기술이 포함되지 않음
- [ ] `common/`과 콘텐츠 중복 없음
- [ ] 해당되는 곳에 `common/`에 대한 적절한 참조 포함
- [ ] `templates/{framework}/CLAUDE.md.template`이 생성됨
- [ ] `README.md`의 디렉토리 구조가 업데이트됨
- [ ] MINOR 버전 이상으로 릴리스

---

## 3. 버전 관리 및 변경 관리

### 3.1 변경 유형 결정

| 변경 | 버전 | 예시 |
|------|------|------|
| 설계 철학의 근본적 변경 | MAJOR | 3대 원칙 변경, 의존성 규칙 반전 |
| 판단 기준의 대폭 변경 | MAJOR | 추상화 임계값의 근본적 변경 |
| 프레임워크 가이드라인 추가 | MINOR | `frameworks/fastapi/` 생성 |
| 기존 가이드라인 개선/확장 | MINOR | 보안 가이드라인에 새 패턴 추가 |
| 프롬프트/skills 추가 | MINOR | 새로운 skill 정의 추가 |
| 오타 수정/표현 개선 | PATCH | 오타 수정, 예시 교체 |
| 템플릿만 변경 | PATCH | 설정 파일 미세 조정 |

### 3.2 호환성 깨뜨리는 변경 처리

다음은 MAJOR 버전이 필요한 호환성 깨뜨리는 변경으로 취급됩니다:

- 파일 삭제 또는 이름 변경 (CLAUDE.md 경로 참조가 깨짐)
- 디렉토리 구조 변경
- 기존 규칙의 폐지 또는 반전

**파일 이동 시:**
1. 이전 경로에 한 줄짜리 리다이렉트 파일을 남김: "Moved to: `new-path`"
2. 다음 MAJOR 버전에서 리다이렉트 파일 제거

---

Languages: [English](../../operation-guide.md) | [日本語](../ja/operation-guide.md) | [简体中文](../zh-CN/operation-guide.md) | 한국어 | [Español](../es/operation-guide.md)
