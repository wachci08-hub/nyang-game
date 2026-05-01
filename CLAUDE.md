# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

바나나 고양이 다마고치 게임. 순수 HTML/CSS/JavaScript 단일 파일(`index.html`)로 구성되며, 외부 의존성이 없다. GitHub Pages에 배포 중.

- **라이브**: https://wachci08-hub.github.io/nyang-game/
- **저장소**: https://github.com/wachci08-hub/nyang-game

## 개발 및 배포

브라우저에서 `index.html`을 직접 열면 된다. 빌드 도구나 서버가 필요없다.

```bash
# 변경 후 배포
git add index.html
git commit -m "설명"
git push
```

## 아키텍처

`index.html` 한 파일 안에 CSS → HTML → JS 순서로 전부 포함되어 있다.

### 전역 상태 객체 `G`

모든 게임 상태는 단일 객체 `G`에서 관리한다.

```js
G = {
  hunger, happy, affection,   // 0–100 스탯 (시간이 지날수록 감소)
  level, xp,                  // 레벨/경험치
  x, y, vx, vy, tx, ty,      // 고양이 위치 및 이동 목표
  walkTimer,                  // 다음 목표 변경까지 남은 ms
  mood,                       // 'normal' | 'happy' | 'hungry' | 'scared'
  isEating, eatTarget,        // 장식 먹는 중 여부 및 대상
  catScale,                   // CSS scale 값 (먹이마다 +0.35, 상한 없음)
  runUntil,                   // 달리기 종료 타임스탬프
  isFlying, flyUntil,         // 날개 아이템 비행 상태
  isBeingDragged,             // 유저가 고양이를 드래그 중
  pageEls,                    // { el, name, ex, ey }[] 화면 장식 목록
}
```

### 게임 루프

`requestAnimationFrame` 기반 루프가 매 프레임 세 함수를 호출한다.

- `updateMovement(dt)` — 고양이 위치 계산. `G.isBeingDragged`이면 건너뜀. `G.runUntil` 기준으로 일반(55px/s) vs 달리기(280px/s) 속도 분기.
- `updateStats(dt)` — 스탯 감소, 바 UI 갱신, `applyMood()` 호출.
- `tryEatEl(now)` — 배고픔 < 30%일 때 가장 가까운 `pageEls` 항목으로 이동하여 먹음.
- `checkFlyExpiry()` — `G.flyUntil` 만료 시 비행 해제.

### 드래그 시스템

전역 변수 `drag`(음식/장식 드래그)와 `catDrag`(고양이 드래그)를 분리해 관리한다. `document` 레벨 `mousemove`에서 어느 쪽이 활성화됐는지 확인해 처리하며, `onPointerUp`에서 두 드래그를 모두 종료한다.

- `drag.kind === 'food'` → 마우스업 시 고양이와 겹치면 `feedCat()` 호출
- `drag.kind === 'page'` → 마우스업 시 `pageObj.ex/ey`를 새 위치로 업데이트
- `catDrag` + `catDragMoved` — 6px 이상 이동해야 드래그로 판정(그 이하는 클릭 처리)

### 고양이 시각 구조

```
.cat (position: fixed, 68×108px — 히트박스)
  └── .cat-scale (CSS scale 적용, transform-origin: bottom center)
       ├── .cat-wings (비행 시 opacity: 1)
       └── .cat-inner (waddle / runBob / flyFloat 애니메이션)
            └── CSS 그림 고양이 (stem, body, face, ears, eyes, nose, mouth, paws)
  └── .bubble (말풍선)
```

`catScale.style.transform = scale(N)` 으로 크기를 키운다. `cat.getBoundingClientRect()`가 아닌 **`catScale.getBoundingClientRect()`** 로 음식 드롭 히트 판정을 해야 실제 시각 크기와 일치한다.

### 특수 아이템

- **일반 음식** (`FOOD_ICONS`): 5초마다 스폰, 25초 후 자동 제거. 드래그해서 고양이에 놓으면 hunger +30, catScale +0.35.
- **날개 아이템** (`wing-item` 클래스): 60초마다 스폰. 먹이면 50초간 비행(`G.isFlying`), `newTarget()`이 화면 전체 Y 범위로 목표를 설정.

### 레이아웃 기준값

```js
const groundY = () => window.innerHeight - CHAT_H - CAT_H - 2; // 고양이가 서는 Y
const minY    = () => HUD_H + 10;   // 상단 허드 아래
const maxY    = () => groundY();
```

페이지 장식(`.pageel`)은 `window.innerHeight - CHAT_H - 68`에 고정 배치된다.

## 스타일 컨벤션

- CSS 색상/간격은 `:root` 변수(`--ui-bg`, `--border`, `--text` 등) 사용.
- 기분(mood) 클래스(`.cat.happy`, `.cat.hungry`, `.cat.scared`)는 CSS만으로 눈·입 형태를 바꾼다.
- `:hover` 위치는 mood 클래스 이후에 선언해 specificity로 항상 우선 적용된다.
- 모든 애니메이션은 `@keyframes`로 정의하고 클래스 토글로 적용한다.
