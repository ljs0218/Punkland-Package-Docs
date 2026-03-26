# Packages 시스템 사용 가이드

## 개요

### 기존 스크립트 방식의 문제점

기존 스크립트 로딩 방식은 다음과 같은 한계가 있었습니다:

| 문제점 | 설명 |
|--------|------|
| **순서 미보장** | 파일 이름 알파벳순으로 불러와 실행 순서를 제어할 수 없음 |
| **재사용 불가** | 게임마다 같은 기능의 스크립트를 복사·붙여넣기 해야 함 |
| **중복 배포** | 비슷한 기능이 여러 파일에 흩어져 관리가 어려움 |
| **전역 오염** | 모든 스크립트가 같은 전역 네임스페이스를 공유 |
| **리소스 충돌** | 이미지나 사운드 파일명이 겹치면 덮어쓰기 발생 |

### Packages 시스템의 해결책

```
✓ 의존성 기반 위상정렬 → 실행 순서 보장
✓ 모듈 방식 격리 → 패키지 단위로 스크립트·리소스 분리
✓ require() 기반 참조 → 명시적 의존성 선언
✓ root: 접두사 → 루트 리소스 명시적 접근
✓ @패키지명/모듈 → 다른 패키지 참조
```

---

## 핵심 개념

### 1. 패키지 격리 (Isolation)

패키지 내부의 스크립트와 리소스는 자신만의 네임스페이스를 가집니다.

```
MyGame/
├── Pictures/
│   └── bg.png              ← 루트 리소스
└── Packages/
    └── UIKit/
        └── Pictures/
            └── bg.png      ← UIKit 패키지 리소스 (별개의 파일)
```

**같은 파일명이라도 충돌하지 않습니다.** 패키지 스크립트에서 `Pictures/bg.png`를 호출하면 자신의 패키지 폴더에서 먼저 찾습니다.

### 2. `root:` 접두사 - 루트 명시적 참조

패키지 내부에서 루트(게임 최상위) 리소스에 접근하려면 `root:` 접두사를 사용합니다.

```lua
-- UIKit/Scripts/HUD.lua 에서

-- 패키지 내부 리소스 (기본 동작)
image.SetImage("Pictures/hud_icon.png")
-- → Packages/UIKit/Pictures/hud_icon.png

-- 루트 리소스 (명시적)
image.SetImage("root:Pictures/background.png")
-- → (루트) Pictures/background.png
```

**스크립트에서도 동일:**

```lua
-- 패키지 내부 모듈
local Config = require("Config")       -- 패키지 내 Config.lua

-- 루트 모듈 (명시적)
local Util = require("root:Utility")   -- 루트 Scripts/ 또는 Shared/Utility.lua
```

### 3. `@패키지명/모듈` - 다른 패키지 참조

다른 패키지의 모듈을 사용하려면 `@` 접두사를 사용합니다.

```lua
-- UIKit/ServerScripts/ScoreManager.lua 에서
local Math = require("@MathLib/MathHelper")

print(Math.add(1, 2))  -- 3
```

**중요:** `@패키지명/모듈` 형식을 사용하려면 반드시 `package.json`의 `dependencies`에 해당 패키지를 선언해야 합니다.

```json
{
    "name": "UIKit",
    "dependencies": ["MathLib"]
}
```

선언하지 않고 참조하면 오류가 발생합니다:
```
Package 'UIKit' requires '@MathLib' but it is not declared in dependencies
```

---

## 디렉터리 구조

```
MyGame/
├── Scripts/              ← 루트 클라이언트 스크립트
├── ServerScripts/        ← 루트 서버 스크립트
├── Shared/               ← 루트 공통 스크립트
├── Pictures/             ← 루트 이미지
├── BGM/                  ← 루트 배경음악
├── SE/                   ← 루트 효과음
└── Packages/
    ├── MathLib/
    │   ├── package.json         ← 필수: 패키지 설정
    │   ├── ServerScripts/
    │   │   └── MathHelper.lua
    │   └── Shared/
    │       └── Constants.lua
    └── UIKit/
        ├── package.json
        ├── Scripts/             ← 패키지 클라이언트 스크립트
        │   └── HUD.lua
        ├── ServerScripts/       ← 패키지 서버 스크립트
        │   └── ScoreManager.lua
        ├── Shared/              ← 패키지 공통 스크립트
        │   └── Config.lua
        ├── Pictures/            ← 패키지 전용 이미지
        │   └── hud_bg.png
        └── SE/                  ← 패키지 전용 효과음
            └── click.ogg
```

---

## package.json 설정

```json
{
    "name": "UIKit",
    "main": "Init",
    "dependencies": ["MathLib"]
}
```

| 필드 | 필수 | 설명 |
|------|:----:|------|
| `name` | ✅ | 패키지 이름. **폴더 이름과 동일해야 함** |
| `main` | ❌ | 진입점 스크립트 (확장자 제외). 지정 시 해당 파일만 자동 실행 |
| `dependencies` | ❌ | 이 패키지가 의존하는 다른 패키지 목록 |

### main 필드 동작

| main 설정 | 동작 |
|-----------|------|
| 없음 (기본) | `return`이 없는 모든 스크립트가 자동 실행됨 |
| `"main": "Init"` | `Init.lua`만 자동 실행. 나머지는 `require()`로만 호출 가능 |

**권장 패턴:** `main`을 지정하고, `Init.lua`에서 필요한 모듈을 `require()`해 초기화

```lua
-- UIKit/Scripts/Init.lua
local HUD = require("HUD")
local Config = require("Config")

HUD.initialize(Config)
```

---

## require() 참조 방식 정리

| 형식 | 설명 | 예시 |
|------|------|------|
| `require("ModuleName")` | 패키지 내부 → 루트 순으로 탐색 | `require("Config")` |
| `require("root:ModuleName")` | 루트에서만 탐색 | `require("root:Utility")` |
| `require("@PkgName/ModuleName")` | 지정한 패키지에서 탐색 | `require("@MathLib/MathHelper")` |

### 모듈 탐색 순서

**패키지 스크립트에서 `require("Config")`:**
1. 패키지 내 `Shared/Config.lua`
2. 패키지 내 `Scripts/Config.lua` (클라이언트) 또는 `ServerScripts/Config.lua` (서버)
3. 루트 `Shared/Config.lua`
4. 루트 `Scripts/Config.lua` 또는 `ServerScripts/Config.lua`

**`root:` 접두사 사용 시:**
- 패키지 내부 탐색을 건너뛰고 루트에서만 찾음

---

## 리소스 경로 참조 방식 정리

| 형식 | 설명 | 예시 |
|------|------|------|
| `"Pictures/file.png"` | 패키지 내부에서 탐색 | `image:SetImage("Pictures/hud.png")` |
| `"root:Pictures/file.png"` | 루트에서 탐색 | `image:SetImage("root:Pictures/bg.png")` |

### 리소스 탐색 규칙

**패키지 스크립트에서:**
```lua
-- 패키지 내부 리소스 (기본)
image.SetImage("Pictures/icon.png")
-- → Packages/UIKit/Pictures/icon.png

-- 루트 리소스 (명시적)
image.SetImage("root:Pictures/bg.png")
-- → Pictures/bg.png
```

**루트 스크립트에서:**
```lua
-- 루트 리소스 (기본)
image.SetImage("Pictures/bg.png")
-- → Pictures/bg.png

-- root: 접두사 사용해도 동일
image.SetImage("root:Pictures/bg.png")
-- → Pictures/bg.png
```

**주의:** 패키지 스크립트에서 리소스가 존재하지 않으면 자동으로 루트를 찾지 **않습니다**. 루트 리소스가 필요하면 반드시 `root:` 접두사를 사용하세요.

---

## 실행 순서

패키지는 **의존성 기반 위상정렬(Topological Sort)** 순서로 실행됩니다.

```
의존성 없는 패키지 → 의존성 있는 패키지 → 루트 스크립트
```

### 예시

```json
// MathLib/package.json
{ "name": "MathLib", "dependencies": [] }

// UIKit/package.json
{ "name": "UIKit", "dependencies": ["MathLib"] }

// GameCore/package.json
{ "name": "GameCore", "dependencies": ["UIKit", "MathLib"] }
```

**실행 순서:**
```
1. MathLib (의존성 없음)
2. UIKit (MathLib에 의존)
3. GameCore (UIKit, MathLib에 의존)
4. 루트 스크립트 (알파벳 순)
```

### 순환 의존성 오류

순환 의존성이 감지되면 크리에이터에서 오류가 발생합니다:

```
Circular dependency detected: UIKit → MathLib → GameCore → UIKit
```

---

## 전체 예시

### 1. MathLib 패키지 (유틸리티 함수)

**`Packages/MathLib/package.json`**
```json
{ "name": "MathLib", "dependencies": [] }
```

**`Packages/MathLib/Shared/MathHelper.lua`**
```lua
local M = {}

function M.add(a, b)
    return a + b
end

function M.clamp(v, min, max)
    if v < min then return min end
    if v > max then return max end
    return v
end

function M.lerp(a, b, t)
    return a + (b - a) * t
end

return M
```

### 2. UIKit 패키지 (UI 시스템)

**`Packages/UIKit/package.json`**
```json
{
    "name": "UIKit",
    "main": "Init",
    "dependencies": ["MathLib"]
}
```

**`Packages/UIKit/Scripts/Init.lua`**
```lua
-- 패키지 진입점
local HUD = require("HUD")
local Config = require("Config")

-- MathLib 패키지 참조
local Math = require("@MathLib/MathHelper")

-- 초기화
HUD.setup(Config)
print("UIKit initialized, Math.add(2,3) =", Math.add(2, 3))
```

**`Packages/UIKit/Scripts/HUD.lua`**
```lua
local M = {}

function M.setup(config)
    -- 패키지 내부 이미지 사용
    local img = GUI.Image.new()
    img.SetImage("Pictures/hud_frame.png")  -- UIKit/Pictures/hud_frame.png

    -- 루트 이미지 사용 (명시적)
    local bg = GUI.Image.new()
    bg.SetImage("root:Pictures/main_bg.png")  -- 루트 Pictures/main_bg.png
end

return M
```

**`Packages/UIKit/Shared/Config.lua`**
```lua
return {
    hudOpacity = 0.9,
    showMinimap = true
}
```

### 3. 루트 스크립트에서 패키지 활용

**`ServerScripts/Main.lua`**
```lua
-- 패키지가 전역에 등록한 것들은 바로 사용 가능
-- 또는 require로 명시적 호출

-- MathLib 모듈 직접 사용 (루트에서도 @패키지명 형식 사용 가능)
local Math = require("@MathLib/MathHelper")
print("Clamped value:", Math.clamp(150, 0, 100))  -- 100
```

---

## 스크립트 폴더 역할

| 폴더 | 실행 위치 |
|------|----------|
| `Scripts/` | 클라이언트에서만 실행 |
| `ServerScripts/` | 서버에서만 실행 |
| `Shared/` | 클라이언트·서버 양쪽에서 실행 |

**Shared 폴더는 클라이언트에 먼저 로드된 후, 서버에 로드됩니다.**

---

## 주의사항 체크리스트

- [ ] 패키지 폴더명 = `package.json`의 `name` 값 (대소문자 구분)
- [ ] `@패키지명/모듈` 사용 시 `dependencies`에 선언 필수
- [ ] 패키지 내 리소스는 자동 탐색, 루트 리소스는 `root:` 접두사 필수
- [ ] 순환 의존성 주의 (A→B→A 불가)
- [ ] `main` 지정 시 해당 파일만 자동 실행됨

---

## 요약 비교표

### 스크립트 참조

| 상황 | 기존 방식 | 패키지 방식 |
|------|----------|------------|
| 같은 폴더 내 모듈 | 불가능 (전역만 사용) | `require("ModuleName")` |
| 다른 패키지 모듈 | 없음 | `require("@PkgName/Module")` |
| 루트 모듈 명시적 참조 | 없음 | `require("root:ModuleName")` |

### 리소스 참조

| 상황 | 기존 방식 | 패키지 방식 |
|------|----------|------------|
| 로컬 리소스 | `"Pictures/img.png"` | `"Pictures/img.png"` (패키지 내) |
| 루트 리소스 | `"Pictures/img.png"` | `"root:Pictures/img.png"` |

### 실행 순서

| 상황 | 기존 방식 | 패키지 방식 |
|------|----------|------------|
| 순서 제어 | 파일명 알파벳순 (예측 불가) | 의존성 기반 위상정렬 (보장됨) |
