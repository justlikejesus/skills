<div align="center">

# ✝️ Just Like Jesus Skills

**예수님을 닮아가는 사역과 교육을 위한 Claude Code 스킬 마켓플레이스**

_Just Like Jesus Ministries · Just Like Jesus School_

[![Marketplace](https://img.shields.io/badge/Claude%20Code-Marketplace-7C3AED?style=for-the-badge)](https://docs.claude.com/en/docs/claude-code/plugin-marketplaces)
[![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)](#)
[![Made with Love](https://img.shields.io/badge/Made%20with-❤️-red?style=for-the-badge)](#)

</div>

---

## 📖 소개

**Just Like Jesus Skills**는 사역(Ministries)과 학교(School) 두 영역을 하나의 마켓플레이스에서 관리하는 통합 저장소입니다. 모든 플러그인은 `category`와 `tags`로 구분되어 있어, 필요한 영역의 도구를 쉽게 찾고 설치할 수 있습니다.

> _"내가 너희에게 행한 것 같이 너희도 행하게 하려 하여 본을 보였노라" — 요한복음 13:15_

---

## ✨ 제공 플러그인

### 🎓 School (학교)

#### [`school-seeders`](./plugins/school-seeders) — 쓰기 전용 시드

`school-monorepo`의 Neon 데이터베이스에 콘텐츠를 시드(seed)하는 도구입니다. 모든 스킬은 **두 단계 출력(데이터 파일 + 실행기)** + **단일 트랜잭션** + **재실행 가능(idempotent)** 패턴을 따릅니다.

| 스킬 | 설명 |
| :--- | :--- |
| [`seed-bible-translation`](./plugins/school-seeders/skills/seed-bible-translation) | OSIS XML, USFM, Zefania, JSON, CSV/TSV, SQLite 등 모든 일반적인 성경 포맷을 자동 인식하여 `bible_translations` / `bible_books` / `bible_verses` 테이블에 시드합니다 |
| [`seed-course-from-srt`](./plugins/school-seeders/skills/seed-course-from-srt) | SRT 자막으로부터 한국어 + 영어 이중 언어의 강의 초안(제목, 설명, 챕터, 레슨, 강사 소개, TipTap `detail_page_json`, 비디오 챕터, 성경 구절 마커)을 생성하고 시드합니다. 슬러그가 이미 존재하면 _evaluate-and-feedback_ 모드로 전환됩니다 |
| [`seed-quiz-from-srt`](./plugins/school-seeders/skills/seed-quiz-from-srt) | 기존 챕터에 부담 없는 시청 확인용(KO + EN) 퀴즈를 시드합니다. `multiple_choice` / `true_false` / `fill_blank`만 사용하며, 강조된 자막 구간을 근거로 합니다 |

#### [`school-evaluators`](./plugins/school-evaluators) — 읽기 전용 감사

기존 콘텐츠가 규약과 일치하는지 감사하는 **읽기 전용(read-only)** 도구입니다. **DB를 절대 변경하지 않습니다.**

| 스킬 | 설명 |
| :--- | :--- |
| [`course-content-lint`](./plugins/school-evaluators/skills/course-content-lint) | 기존 강의를 18개 등록 전환(enrollment-conversion) 규칙과 구조 규약에 대해 감사합니다. 6개 관점 점수표(전환·교수법·신학적 정합성·이중언어 균형·신뢰·일관성)와 표면별 위반 목록을 마크다운으로 출력합니다 |
| [`quiz-content-lint`](./plugins/school-evaluators/skills/quiz-content-lint) | 기존 챕터 퀴즈를 15개 기계적 규칙·문항 유형별 규칙·자문 평가표(advisor rubric)에 대해 감사합니다. 6개 관점 점수표(문항 메커니즘·공정성/Bloom·자막 근거·커버리지/순환·이중언어 균형·교수법적 가치)와 문항별 위반 목록을 마크다운으로 출력합니다 |

### ⛪ Ministries (사역)

> 곧 추가될 예정입니다. 기여를 환영합니다!

---

## 🚀 빠른 시작

### 1. 마켓플레이스 등록

Claude Code 안에서 다음 명령어를 실행하세요.

```shell
/plugin marketplace add justlikejesus/skills
```

### 2. 플러그인 설치

```shell
/plugin install school-seeders@jljm-skills
/plugin install school-evaluators@jljm-skills
```

### 3. 사용해 보기

```shell
/seed-bible-translation <bible-file-path> [--code KRV] [--lang ko]
/seed-course-from-srt <srt-path> [additional-srt-paths...]
/seed-quiz-from-srt <course-slug> <chapter-position-0indexed> <srt-path>
/course-content-lint <course-slug> [--rule <N>[,N,...]] [--locale ko|en] [--snapshot] [--severity error|warn|info]
/quiz-content-lint <course-slug> <chapter-position-0indexed> [--srt <path>] [--rule <ID>[,ID,...]] [--locale ko|en] [--snapshot] [--severity error|warn|info]
```

---

## 📂 디렉토리 구조

```
skills/
├── .claude-plugin/
│   └── marketplace.json                 # 마켓플레이스 카탈로그
└── plugins/
    ├── school-seeders/                  # 🎓 School (쓰기)
    │   ├── .claude-plugin/
    │   │   └── plugin.json
    │   └── skills/
    │       ├── seed-bible-translation/
    │       │   └── SKILL.md
    │       ├── seed-course-from-srt/
    │       │   └── SKILL.md
    │       └── seed-quiz-from-srt/
    │           └── SKILL.md
    └── school-evaluators/               # 🎓 School (읽기)
        ├── .claude-plugin/
        │   └── plugin.json
        └── skills/
            ├── course-content-lint/
            │   └── SKILL.md
            └── quiz-content-lint/
                └── SKILL.md
```

---

## 🛠️ 새 스킬 기여하기

1. 적절한 플러그인 디렉토리 (`plugins/<plugin-name>/skills/`)에 새 스킬 폴더를 만듭니다
2. `SKILL.md`에 frontmatter (`name`, `description`, `user-invocable`, 선택적으로 `argument-hint`)와 본문을 작성합니다
3. 새 플러그인을 만들 경우 `.claude-plugin/plugin.json`을 추가하고 루트의 `marketplace.json`에 항목을 등록합니다 (`category`는 `ministries` 또는 `school`)
4. 로컬에서 검증한 뒤 PR을 보내주세요

```shell
/plugin validate .
```

---

## 🙏 함께 만드는 분들

| 역할 | 이름 |
| :--- | :--- |
| Maintainer | **Jay Cho** ([@justlikejesus](https://github.com/justlikejesus)) |
| Organization | Just Like Jesus Ministries · Just Like Jesus School |

---

<div align="center">

**Soli Deo Gloria — 오직 하나님께 영광**

</div>
