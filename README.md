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

**Just Like Jesus Skills**는 사역(Ministries)과 학교(School) 두 영역을 하나의 마켓플레이스에서 관리하는 통합 저장소입니다. 모든 스킬은 `category`와 `tags`로 구분되어 있어, 필요한 영역의 도구를 쉽게 찾고 설치할 수 있습니다.

> _"내가 너희에게 행한 것 같이 너희도 행하게 하려 하여 본을 보였노라" — 요한복음 13:15_

---

## ✨ 제공 스킬

### ⛪ Ministries (사역)

| 플러그인 | 설명 |
| :--- | :--- |
| [`sermon-helper`](./plugins/sermon-helper) | 설교 개요, 묵상글, 소그룹 가이드를 작성하고 다듬어 줍니다 |

### 🎓 School (학교)

| 플러그인 | 설명 |
| :--- | :--- |
| [`lesson-planner`](./plugins/lesson-planner) | 수업 계획, 학습 목표, 교실 활동을 생성합니다 |

---

## 🚀 빠른 시작

### 1. 마켓플레이스 등록

Claude Code 안에서 다음 명령어를 실행하세요.

```shell
/plugin marketplace add justlikejesus/skills
```

### 2. 원하는 플러그인 설치

```shell
/plugin install sermon-helper@jljm-skills
/plugin install lesson-planner@jljm-skills
```

### 3. 사용해 보기

```shell
/sermon-outline
/lesson-plan
```

---

## 📂 디렉토리 구조

```
skills/
├── .claude-plugin/
│   └── marketplace.json          # 마켓플레이스 카탈로그
└── plugins/
    ├── sermon-helper/             # ⛪ Ministries
    │   ├── .claude-plugin/
    │   │   └── plugin.json
    │   └── skills/
    │       └── sermon-outline/
    │           └── SKILL.md
    └── lesson-planner/            # 🎓 School
        ├── .claude-plugin/
        │   └── plugin.json
        └── skills/
            └── lesson-plan/
                └── SKILL.md
```

---

## 🛠️ 새 스킬 기여하기

1. `plugins/<plugin-name>/` 디렉토리를 새로 만듭니다
2. `.claude-plugin/plugin.json`에 메타데이터를 작성합니다
3. `skills/<skill-name>/SKILL.md`에 스킬 내용을 작성합니다
4. 루트의 `.claude-plugin/marketplace.json`에 항목을 추가합니다 (`category`는 `ministries` 또는 `school`)
5. 로컬에서 검증한 뒤 PR을 보내주세요

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
