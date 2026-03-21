# AI Dev OS Rules — Python

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](../../../LICENSE)

> Python 프로젝트(FastAPI, Flask, Typer CLI 등)를 위한 4계층 가이드라인.
> git submodule로 추가하고, AI 코딩 어시스턴트에서 참조합니다.

**[AI Dev OS](https://github.com/yunbow/ai-dev-os) 에코시스템의 일부입니다.**

## 왜 이 규칙인가?

AI Dev OS Rules는 AI 코딩 어시스턴트에게 모호한 지시 대신 **구체적이고 검증 가능한 기준**을 제공합니다:

- **13개 공통 규칙** — 명명, 에러 처리, 보안, 테스트, 로깅, i18n 등
- **프레임워크별 규칙** — FastAPI 패턴, Python CLI 모범 사례
- **충돌 해결 내장** — Specificity Cascade가 규칙 우선순위를 자동으로 해결
- **버전 관리 및 감사 가능** — 태그에 고정, 차이점 확인, PR에서 리뷰

## 빠른 시작

```bash
npx ai-dev-os init --rules python --plugin claude-code
```

> 모든 것을 자동으로 설정합니다. 자세한 내용은 [AI Dev OS CLI](https://github.com/yunbow/ai-dev-os-cli)를 참조하세요.

<details>
<summary>수동 설정</summary>

**submodule로 추가**

```bash
cd /path/to/your-project
git submodule add https://github.com/yunbow/ai-dev-os-rules-python.git docs/ai-dev-os
git submodule update --init
```

**템플릿으로 설정 (Python CLI)**

```bash
bash docs/ai-dev-os/templates/python-cli/submodule-setup.sh
```

**CLAUDE.md 편집**

`templates/python-cli/CLAUDE.md.template`을 `./CLAUDE.md`로 복사하고, 프로젝트명과 고유 가이드라인을 작성합니다.

**submodule 업데이트**

```bash
git submodule update --remote docs/ai-dev-os
```

</details>

## 포함 내용

| 레이어 | 경로 | 내용 |
|--------|------|------|
| L1 — 설계 철학 | `01_philosophy/` | 원칙, 멘탈 모델, 안티패턴 |
| L2 — 판단 기준 | `02_decision-criteria/` | 추상화, 기술 선정, 아키텍처, 에러, 보안 |
| L3 — 공통 가이드라인 | `03_guidelines/common/` | 13개 규칙: 코드, 명명, 유효성 검사, 에러, 로깅, 보안, 테스트 등 |
| L3 — FW 가이드라인 | `03_guidelines/frameworks/` | [FastAPI](03_guidelines/frameworks/fastapi/README.md), [Python CLI](03_guidelines/frameworks/python-cli/README.md) |
| 템플릿 | `templates/` | [Python CLI 스캐폴딩](templates/python-cli/README.md) |

## Specificity Cascade

규칙이 충돌할 때, **번호가 작을수록 우선**됩니다.

| 우선순위 | 레이어 | 예 |
|---------|--------|------|
| 1 (최우선) | 프레임워크별 가이드라인 | `03_guidelines/frameworks/python-cli/*` |
| 2 | 공통 가이드라인 | `03_guidelines/common/*` |
| 3 | 판단 기준 | `02_decision-criteria/*` |
| 4 | 설계 철학 | `01_philosophy/*` |

<details>
<summary>디렉토리 구조</summary>

```
ai-dev-os/
├── docs/
│   ├── operation-guide.md        # 운영 및 기여 가이드
│   └── i18n/                     # 다국어 가이드
│       ├── ja/                   #   日本語
│       ├── zh-CN/                #   简体中文
│       ├── ko/                   #   한국어
│       └── es/                   #   Español
│
├── 01_philosophy/                # 설계 철학 [샘플 - 모국어로 재작성]
│   ├── principles.md             #   세 가지 기둥: Correctness, Observability, Pragmatism
│   ├── mental-models.md          #   10가지 사고 프레임워크
│   └── anti-patterns.md          #   피해야 할 패턴 (코드 예제 포함)
│
├── 02_decision-criteria/         # 판단 기준 [샘플 - 모국어로 재작성]
│   ├── abstraction.md            #   추상화의 시기와 임계값
│   ├── technology-selection.md   #   기술 선정 판단 프레임워크
│   ├── architecture.md           #   API 설계, 상태 관리, 모듈 구성
│   ├── error-strategy.md         #   에러 분류, 재시도, Result 패턴
│   └── security-vs-ux.md        #   보안 조치 우선순위와 균형
│
├── 03_guidelines/                # 가이드라인 [영어]
│   ├── common/                   #   공통 (언어/FW 독립)
│   │   ├── code.md               #     코딩 규약
│   │   ├── naming.md             #     명명 규칙
│   │   ├── validation.md         #     유효성 검사
│   │   ├── error-handling.md     #     에러 처리
│   │   ├── logging.md            #     로깅
│   │   ├── security.md           #     보안
│   │   ├── rate-limiting.md      #     속도 제한
│   │   ├── testing.md            #     테스트
│   │   ├── performance.md        #     성능
│   │   ├── cors.md               #     CORS
│   │   ├── env.md                #     환경 변수
│   │   ├── cicd.md               #     CI/CD
│   │   └── i18n.md               #     국제화
│   │
│   └── frameworks/               #   프레임워크별 (각 README.md 참조)
│       ├── fastapi/              #     → [README.md](03_guidelines/frameworks/fastapi/README.md)
│       └── python-cli/           #     → [README.md](03_guidelines/frameworks/python-cli/README.md)
│
│
└── templates/                    # 프로젝트 템플릿 [영어]
    └── python-cli/               #     → [README.md](templates/python-cli/README.md)
```

</details>

<details>
<summary>운영 및 버전 관리</summary>

업데이트 정책, 프레임워크 추가 절차, 버전 관리 세부사항은 **[docs/operation-guide.md](../../../docs/operation-guide.md)**를 참조하세요.

**업데이트 빈도 가이드**

| 섹션 | 빈도 | 영향 범위 |
|------|------|---------|
| `01_philosophy/` | 매우 드뭄 | 모든 프로젝트 (MAJOR 변경) |
| `02_decision-criteria/` | 드뭄 | 모든 프로젝트 |
| `03_guidelines/common/` | 중간 | 모든 프로젝트 |
| `03_guidelines/frameworks/` | 높음 | 해당 FW 프로젝트만 |
| `templates/` | 중간 | 새 프로젝트만 |

**프레임워크 추가**

새 프레임워크 (예: FastAPI, Flask, Django)를 추가하는 경우:

1. `03_guidelines/frameworks/{framework}/`에 `overview.md`와 `project-structure.md` 생성
2. `common/`과의 책임 분리 철저 (공통 규칙 → common, FW 고유 패턴 → frameworks)
3. `templates/{framework}/`에 템플릿 준비
4. 본 README의 디렉토리 구조 업데이트

자세한 절차와 체크리스트는 [docs/operation-guide.md](../../../docs/operation-guide.md)를 참조.

**버전 관리** — 시맨틱 버전 관리 (git 태그)로 관리합니다.

| 변경 유형 | 버전 | 예 |
|---------|------|------|
| 철학/판단 기준의 대폭 변경 | MAJOR | v2.0.0 |
| 가이드라인 추가/개선 | MINOR | v1.1.0 |
| 오타 수정, 보충 추가 | PATCH | v1.0.1 |

submodule을 특정 태그에 고정:

```bash
cd docs/ai-dev-os
git checkout v1.2.0
cd ../..
git add docs/ai-dev-os
git commit -m "chore: pin ai-dev-os to v1.2.0"
```

</details>

## 언어 정책

- `01_philosophy/`와 `02_decision-criteria/`에는 **영어 샘플 콘텐츠**가 포함되어 있습니다. 클론 후 팀의 추상적 사고와 의사결정 프레임워크의 뉘앙스를 보존하기 위해 **모국어로 다시 작성하세요**.
- 그 외 모든 섹션은 AI 호환성과 국제적 접근성을 위해 **영어**로 작성합니다.
- 다국어 운영 가이드는 `docs/i18n/`에 있습니다.

## 관련

| 저장소 | 설명 |
|---|---|
| [ai-dev-os](https://github.com/yunbow/ai-dev-os) | 프레임워크 사양과 이론 |
| [rules-typescript](https://github.com/yunbow/ai-dev-os-rules-typescript) | TypeScript / Next.js / Node.js 가이드라인 |
| [plugin-claude-code](https://github.com/yunbow/ai-dev-os-plugin-claude-code) | Claude Code용 Skills, Hooks, Agents |
| [plugin-kiro](https://github.com/yunbow/ai-dev-os-plugin-kiro) | Kiro용 Steering Rules와 Hooks |
| [plugin-cursor](https://github.com/yunbow/ai-dev-os-plugin-cursor) | 가이드라인 기반 개발을 위한 Cursor Rules (.mdc) |
| [cli](https://github.com/yunbow/ai-dev-os-cli) | 설정 자동화 — `npx ai-dev-os init` |
| [benchmark](https://github.com/yunbow/ai-dev-os-benchmark) | 정량적 벤치마크 — 가이드라인 품질 영향 데이터 |

## 라이선스

[MIT](../../../LICENSE)

---

Languages: [English](../../../README.md) | [日本語](../ja/README.md) | [简体中文](../zh-CN/README.md) | 한국어 | [Español](../es/README.md)
