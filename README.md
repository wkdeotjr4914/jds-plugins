# jds-plugins

JDS의 개인 Claude Code 플러그인 마켓플레이스.

## 플러그인
- **flow-maker** — `ready` → `go` 흐름으로 의도를 구현까지 이끄는 스킬 모음.
  [dryforge](https://github.com/prekuter/dryforge)(Apache-2.0)를 기반으로 수정했습니다. 자세한 내용은 `flow-maker/NOTICE` 참고.

## 설치
```
/plugin marketplace add C:\AI-agent\Flow-Make\jds-plugins
/plugin install flow-maker@jds-plugins
```

## 스킬
- `/flow-maker:ready` — 의도 정리 및 3-doc(handoff/spec/plan) 작성
- `/flow-maker:go` — 승인된 3-doc 구현 및 검증
- `/flow-maker:migration` — 기존 프로젝트 문서를 하네스 구조로 이전
