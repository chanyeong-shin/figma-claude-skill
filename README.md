# figma-claude-skill

Claude Code에서 **Figma 캔버스를 실시간으로 조작**하는 스킬입니다.
[Talk To Figma MCP](https://github.com/sonnylazuardi/cursor-talk-to-figma-mcp)(자체호스팅)를 기반으로,
실제로 써보면서 부딪힌 문제들을 해결한 설정과 절차를 정리했습니다.

```
Claude Code  ──MCP──>  WebSocket(3055)  ──>  Figma Desktop 플러그인  ──>  캔버스
```

---

## 왜 자체호스팅인가

Figma **공식 MCP**(`mcp.figma.com`)도 있지만 요금제 제약이 큽니다.

| | 공식 MCP | 자체호스팅 (이 방식) |
|---|---|---|
| 무료(Starter)·View·Collab 좌석 | **월 6회** | **무제한** |
| 실사용에 필요한 좌석 | Professional 이상 Dev/Full | 무료 계정 OK |
| 셋업 | 명령 1줄 | 빌드 + 소켓 + 플러그인 |
| 도구 수 | 적음 | 약 57개 |

시안 하나 만드는 데도 수십 번 호출이 필요하므로, **무료 계정이라면 자체호스팅이 사실상 유일한 선택**입니다.
유료 Dev/Full 좌석이 있다면 공식 MCP가 훨씬 편합니다.

---

## 사전 요구사항

| 항목 | 비고 |
|---|---|
| **Figma Desktop 앱** | **필수.** 웹 버전은 커스텀 플러그인 등록 불가 |
| Node.js 18+ | |
| bun | 빌드·실행용 |
| Claude Code | Pro 이상 (무료 플랜은 사용 불가) |

> Figma 계정은 **무료(Starter)로 충분**합니다.
> 파일은 **초안(Drafts)**에 만드세요 — 팀 프로젝트는 무료 플랜에서 3개 제한이지만 초안은 무제한입니다.

---

## 설치

### 1. MCP 서버 빌드

```bash
git clone https://github.com/sonnylazuardi/cursor-talk-to-figma-mcp.git figma-mcp
cd figma-mcp
bun install
bun run build        # Windows: bun run build:win
```

### 2. 플러그인 패치 적용 (권장)

```bash
git apply /path/to/patches/ui.html.patch
```

패치 없이도 동작하지만, 적용하면 **채널 ID를 매번 주고받을 필요가 없어집니다.**
자세한 내용은 [패치 설명](#플러그인-패치-설명) 참조.

### 3. Claude Code에 MCP 등록

```bash
claude mcp add --transport stdio figma -- bun /절대경로/figma-mcp/dist/talk_to_figma_mcp/server.js
claude mcp list      # ✔ Connected 확인
```

> ⚠️ **절대 경로**를 써야 합니다. 등록 후 Claude Code를 **재시작**해야 도구가 로드됩니다.

### 4. 스킬 설치

`SKILL.md`를 프로젝트의 `.claude/skills/figma/SKILL.md`로 복사합니다.

```bash
mkdir -p .claude/skills/figma
cp SKILL.md .claude/skills/figma/SKILL.md
```

스킬 안의 `$FIGMA_MCP_DIR`을 실제 경로로 바꿔주세요.

### 5. Figma 플러그인 임포트 (최초 1회)

Figma Desktop에서:
```
Plugins → Development → Import plugin from manifest...
→ figma-mcp/src/claude_mcp_plugin/manifest.json 선택
```

`Plugins → Development`에 **Claude MCP Plugin**이 보이면 성공입니다.

---

## 사용법

```bash
# 1) 소켓 서버 (별도 터미널, 작업 내내 켜둠)
cd figma-mcp && bun run dist/socket.js

# 2) Claude Code에서
/figma
```

Figma에서 플러그인 실행(`Ctrl + /` → "Claude") → **Connect** → 그대로 두면 됩니다.

그 다음은 자연어로:
```
모바일 로그인 화면 만들어줘.
다크 배경에 오렌지 포인트.
이메일·비밀번호 입력창, 로그인 버튼, 하단에 회원가입 링크.
```

### 프롬프트 3요소

결과 품질은 이 3가지를 명시하는지에 크게 좌우됩니다.

| 요소 | 예시 |
|---|---|
| **화면 종류** | 모바일 로그인 / 랜딩 히어로 / 카드뉴스 |
| **컬러 + 테마** | 오렌지 포인트, 다크 배경 |
| **요소 목록** | 이메일 입력, 비밀번호 입력, 로그인 버튼 |

---

## 🔴 가장 중요한 규칙

> ### 플러그인 창을 닫지 마세요.

**플러그인 창 = 연결 그 자체입니다.** Connect를 눌러 설정이 저장되는 방식이 아니라,
**그 창이 떠 있는 동안만 WebSocket이 살아있습니다.** 창을 닫으면 연결도 같이 죽습니다.

명령이 `Request to Figma timed out`으로 실패하면 **가장 먼저 이걸 의심하세요.**
소켓 로그에 `WebSocket closed ... Code 1001`이 찍혔다면 창이 닫힌 것입니다.

---

## 할 수 있는 것 / 없는 것

| ✅ 가능 | ❌ 불가 |
|---|---|
| 도형 생성 (사각형·원·별·다각형) | **이미지/아이콘 삽입** (외부 파일) |
| 프레임 · Auto Layout | **그라디언트** (단색만) |
| 텍스트 생성·수정, 폰트·크기·굵기 | 프로토타입 인터랙션 |
| 색상·테두리·모서리·이펙트 | 애니메이션 |
| 컴포넌트 인스턴스 | 펜툴 · 마스크 · 불린 연산 |
| HTML 변환 · 이미지 내보내기 | |

> **AI가 구조 80%, 사람이 마무리 20%** 정도로 기대치를 잡는 게 현실적입니다.
> 이미지 자리는 placeholder 사각형으로 잡아두고 나중에 직접 교체하는 방식이 잘 맞습니다.

---

## 플러그인 패치 설명

`patches/ui.html.patch`는 두 가지를 바꿉니다.

### ① 고정 채널 (핵심)

원본은 Connect할 때마다 **랜덤 8자 채널 ID를 새로 생성**합니다.
그래서 재연결할 때마다 ID가 바뀌고(`bbdoktad` → `7g33pq6v` → …), 매번 사람이 복사해서 전달해야 했습니다.

```js
function generateChannelName() {
  return "claude";   // 고정
}
```

고정하면 **ID 교환이 완전히 불필요**해집니다. 스킬이 `join_channel("claude")`로 바로 붙습니다.
**이 패치의 존재 이유가 사실상 이것입니다.**

### ② 자동 재연결 (선택)

`onclose`에 지수 백오프 재연결(1s → 최대 10s)과 `visibilitychange`/`focus` 복귀 시 즉시 재연결을 추가합니다.

다만 **효과는 제한적입니다.** 플러그인 창이 통째로 언로드되면(Code 1001) 이 코드도 같이 사라져서 동작하지 않습니다.
소켓만 끊기는 경우(Code 1005)에만 유효합니다. **없어도 무방하며, 해가 되지도 않습니다.**

### 되돌리기

```bash
git apply -R patches/ui.html.patch
```

---

## 트러블슈팅

| 증상 | 원인 | 해결 |
|---|---|---|
| `Request to Figma timed out` | **플러그인 창이 닫힘** ← 1순위 | 플러그인 재실행 → Connect → 창 켜두기 |
| 소켓 응답 없음 | 서버 미기동 | `bun run dist/socket.js` |
| `activeConnections: 0` | 플러그인 미실행 | Figma에서 플러그인 실행 |
| MCP 도구가 안 보임 | 등록 후 재시작 안 함 | Claude Code 재시작 |
| 텍스트 생성 실패 | 폰트 미로드 | `load_font_async` 먼저 (Bold/Regular 각각) |
| 색상이 이상함 | 0~255로 입력 | **0~1 범위**로 변환 (`값 / 255`) |
| 플러그인이 메뉴에 없음 | 웹 버전 사용 | **Figma Desktop 앱** 사용 |
| write timeout | view-only 파일 | 편집 권한 있는 파일로 이동 |

**소켓 상태 확인**
```bash
curl -s http://localhost:3055/status
# {"status":"running","stats":{"activeConnections":2,...}}
#   activeConnections 2 = MCP 서버 + 플러그인 (정상)
#   activeConnections 1 = 플러그인 미연결
```

---

## 동작 확인 예시

이 스킬로 실제 만들어본 비주얼 노벨 UI 구조입니다. 명령 26개를 연속 실행해 생성했습니다.

```
VN_Screen  1280×720
├── CharacterArea_Placeholder   500×420   ← 일러스트 자리
├── NameTag + CharacterName
├── DialogueBox (반투명, radius 12) + DialogueText
└── MenuBar (Auto Layout)
    └── AUTO / SKIP / SAVE / LOG
```

---

## 크레딧

- MCP 서버·플러그인: [sonnylazuardi/cursor-talk-to-figma-mcp](https://github.com/sonnylazuardi/cursor-talk-to-figma-mcp) (MIT)
- 이 저장소는 위 프로젝트를 사용하기 위한 **스킬과 패치**만 담고 있습니다. 원본 코드는 포함하지 않습니다.
