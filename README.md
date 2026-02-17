# Skills

> Cursor / Claude용 스킬 모음

## 개요

이 저장소는 AI 코딩 에이전트가 프로젝트 작업 시 참고할 수 있는 **스킬(Skill)** 정의를 담고 있습니다.

## 포함 스킬

| 스킬 | 설명 |
| :--- | :--- |
| [Feature-Sliced Design](./skills/feature-sliced-design/) | Next.js 앱에서 FSD 아키텍처 구현 가이드 |

## 사용 방법

스킬 디렉터리를 Cursor 스킬 경로에 복사하여 사용합니다:

```bash
npx skills add https://github.com/unerue/skills --skill feature-sliced-design
pnpm skills add https://github.com/unerue/skills --skill feature-sliced-design
bunx skills add https://github.com/unerue/skills --skill feature-sliced-design
bunx skills add https://github.com/unerue/skills --skill feature-sliced-design --agent claude-code cursor
```

또는 프로젝트에 직접 추가: `.claude/skills/feature-sliced-design/`
