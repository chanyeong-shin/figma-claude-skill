---
name: figma
description: |
  Figma 실시간 협업 디자인 스킬. Talk To Figma MCP(자체호스팅, WebSocket 3055)로
  Figma 캔버스를 직접 조작한다. 소켓 상태 점검 → 채널 연결 → 3단 검증 → 작업 시작까지 자동 진행.

  자동 호출 트리거:
  - "피그마 시작" / "피그마 연결" / "figma 시작"
  - "피그마로 ~ 만들어줘" / "디자인 만들어줘"
  - "피그마 연결 확인" / "소켓 확인"
  - Figma 캔버스 조작이 필요한 모든 작업
user-invocable: true
---

# Figma 실시간 협업 스킬

> 환경: Talk To Figma MCP (자체호스팅) · WebSocket `localhost:3055` · Figma **Desktop** 앱 필수
> 서버 경로: `$FIGMA_MCP_DIR/`

---

## 🔴 1순위 규칙 — 플러그인 창을 닫지 말 것

**Figma 플러그인 창 = 연결 그 자체다.** 창을 닫으면 그 안에서 돌던 WebSocket도 같이 죽는다.
"Connect를 눌러 설정을 저장하는" 방식이 아니라, **창이 떠 있는 동안만 연결이 살아있다.**

작업 중 명령이 `Request to Figma timed out`으로 실패하면, **가장 먼저 이걸 의심한다.**
사용자에게 이렇게 안내한다:

```
Figma 플러그인 창이 켜져 있는지 확인해 주세요.
닫혀 있으면: Ctrl + / → "Claude" 입력 → Claude MCP Plugin 실행 → Connect
```

> 채널은 `claude`로 고정돼 있어 재실행해도 같은 채널로 붙는다. **채널 ID를 물어볼 필요 없다.**

---

## Step 0. 연결 상태 자동 점검 (묻지 말고 즉시 실행)

아래 3가지를 **사용자에게 묻기 전에** 먼저 확인한다.

| # | 확인 | 방법 | 통과 조건 |
|---|---|---|---|
| 1 | WebSocket 서버 | `curl -s http://localhost:3055/status` | `"status":"running"` |
| 2 | 플러그인 연결 | 위 응답의 `stats.activeConnections` | **≥ 1** |
| 3 | MCP 도구 | `mcp__figma__*` 도구 사용 가능 여부 | 로드됨 |

### 분기 처리

**1 실패 (소켓 안 떠 있음)** → 묻지 말고 즉시 백그라운드 기동:
```bash
cd "$FIGMA_MCP_DIR" && bun run dist/socket.js
```
`run_in_background: true`로 실행 → 2초 후 status 재확인 → 통과하면 계속.

**2 실패 (activeConnections = 0)** → 사용자 액션 필요. 아래를 안내하고 대기:
```
Figma Desktop에서 플러그인을 실행해 주세요.

1. Figma Desktop 앱 실행
2. 디자인 파일 열기 (없으면 새로 만들기 — 초안/Drafts 권장)
3. Ctrl + / → "Claude" 입력 → Claude MCP Plugin 실행
   (또는 좌측 "도구" 아이콘, 또는 캔버스 우클릭 → 플러그인 → 개발)
4. Connect 클릭 → 창을 켜둔 채로 "됐어"라고 알려주기

⚠️ 플러그인 창을 닫지 마세요. 창을 닫으면 연결이 끊깁니다.
```

**3 실패** → `claude mcp list`로 확인 안내 후 Claude Code 재시작 요청.

**전부 통과** → 상태 1줄 보고 후 Step 1로.

---

## Step 1. 채널 연결

채널은 **`claude`로 고정**돼 있다. 사용자에게 ID를 묻지 말고 그대로 호출한다.

```
mcp__figma__join_channel(channel: "claude")
```

> 플러그인의 `generateChannelName()`을 고정값 `"claude"`로 수정해 두었다
> (`$FIGMA_MCP_DIR/src/claude_mcp_plugin/ui.html`).
> 덕분에 플러그인을 몇 번 재실행해도 항상 같은 채널이라 ID 교환이 불필요하다.

---

## Step 2. 3단 검증 (자동, 묻지 않음)

연결 직후 **읽기 → 쓰기 → 삭제** 순으로 실제 동작을 확인한다.
사용자 확인을 받지 말고 즉시 실행한 뒤 결과만 보고한다.

| 단계 | 도구 | 확인 내용 |
|---|---|---|
| 읽기 | `get_document_info` | 문서 접근 가능 |
| 쓰기 | `create_rectangle` (10×10, 좌상단 구석) | 캔버스 편집 가능 |
| 삭제 | `delete_node` (방금 만든 노드) | 정리 완료 |

**쓰기 실패 시 원인 진단**
- `view-only` 파일 → 편집 권한 있는 파일로 이동 요청
- `parentId` 누락 → 페이지 지정 후 재시도

검증 통과 시 보고:
```
✅ Figma 연결 완료
   소켓: running · 채널: <ID> · 문서: <파일명>
   읽기 ✓ 쓰기 ✓ 삭제 ✓

무엇을 만들까요?
```

---

## Step 3. 디자인 작업

### 🔴 반드시 지킬 규칙 3가지

**1. 폰트는 먼저 로드한다**
텍스트를 만들거나 폰트를 바꾸기 전에 **반드시** `load_font_async`를 먼저 호출한다.
빠뜨리면 텍스트 조작이 실패한다.

**2. 색상은 0~1 범위다**
`set_fill_color` / `set_stroke_color`의 r·g·b는 **0~255가 아니라 0~1**이다.
- `#00D26A` → `r: 0, g: 0.824, b: 0.416`
- 변환: `값 / 255`

**3. 부모를 먼저 만든다**
프레임 → 자식 순서로 만든다. `create_frame`으로 컨테이너를 먼저 만들고
`parentId`를 지정해 자식을 넣는다. 부모 없이 만들면 캔버스 루트에 흩어진다.

### 작업 순서 권장
```
1. create_frame            컨테이너 (크기·위치 지정)
2. set_auto_layout         레이아웃 방향·간격 설정
3. load_font_async         쓸 폰트 미리 로드
4. create_text / create_rectangle ...   자식 요소
5. set_fill_color / set_corner_radius   스타일
6. get_node_info           결과 확인
```

### 사용 가능한 도구 57개

| 그룹 | 도구 |
|---|---|
| **생성** | `create_frame` `create_rectangle` `create_ellipse` `create_polygon` `create_star` `create_text` `create_page` `create_component_instance` `create_component_from_node` `create_component_set` `clone_node` |
| **수정** | `set_fill_color` `set_stroke_color` `set_corner_radius` `set_effects` `set_effect_style_id` `set_auto_layout` `move_node` `resize_node` `delete_node` `group_nodes` `ungroup_nodes` `flatten_node` `insert_child` `rename_node` |
| **텍스트** | `set_text_content` `set_multiple_text_contents` `set_font_name` `set_font_size` `set_font_weight` `set_line_height` `set_letter_spacing` `set_text_case` `set_text_decoration` `set_paragraph_spacing` `set_text_style_id` `scan_text_nodes` `get_styled_text_segments` `load_font_async` |
| **조회** | `get_document_info` `get_node_info` `get_nodes_info` `get_selection` `get_styles` `get_local_components` `get_remote_components` `get_pages` |
| **기타** | `join_channel` `export_node_as_image` `set_current_page` `rename_page` `delete_page` |

### ❌ 안 되는 것 (미리 알리고 대안 제시)
| 불가 | 대안 |
|---|---|
| 외부 이미지/아이콘 삽입 | 사각형 placeholder 생성 → 사용자가 직접 교체 |
| 그라디언트 | 단색만 가능 |
| 프로토타입 인터랙션 | Figma에서 수동 |
| 애니메이션 | 불가 |
| 펜툴 · 마스크 · 불린 연산 | 불가 |

> **AI가 80% 구조 작업, 사용자가 20% 마무리** — 처음부터 이 기대치를 공유한다.

---

## Step 4. 반복 협업

작업 후에는 **결과를 확인시키고 다음 지시를 받는다.**

```
✅ [작업명] 완료
   생성: <노드 요약>

Figma에서 확인해 보세요. 수정할 부분 있으면 말씀해 주세요.
```

- 부분 수정 요청 시 **전체를 다시 만들지 말고** 해당 노드만 수정한다
- 노드 ID를 기억해 두고 재사용한다
- 사용자가 Figma에서 직접 선택한 뒤 "이거 바꿔줘"라고 하면 `get_selection`으로 대상을 특정한다

---

## 트러블슈팅

| 증상 | 원인 | 해결 |
|---|---|---|
| **`Request to Figma timed out`** | **플러그인 창이 닫혀 있음** ← **1순위로 의심** | 플러그인 재실행 → Connect → **창 켜두기** |
| 소켓 응답 없음 | 서버 미기동 | `bun run dist/socket.js` 백그라운드 실행 |
| `activeConnections: 0` | 플러그인 미실행/종료 | Figma Desktop에서 플러그인 실행 |
| write timeout | view-only 파일 | 편집 권한 있는 파일로 이동 |
| 텍스트 조작 실패 | 폰트 미로드 | `load_font_async` 먼저 호출 (Bold/Regular 각각) |
| 색상이 이상함 | 0~255로 입력 | 0~1 범위로 변환 (`값/255`) |
| 플러그인이 메뉴에 없음 | 웹 버전 사용 | **Figma Desktop 앱** 사용 |

> **로그 확인법** — 소켓 서버 로그에서 `WebSocket closed ... Code 1001`이 보이면
> 플러그인 창이 닫힌 것이다. 다른 원인을 찾기 전에 창부터 확인한다.

---

## 참고 경로

| 항목 | 경로 |
|---|---|
| MCP 서버 | `$FIGMA_MCP_DIR/dist/talk_to_figma_mcp/server.js` |
| 소켓 서버 | `$FIGMA_MCP_DIR/dist/socket.js` |
| Figma 플러그인 매니페스트 | `$FIGMA_MCP_DIR/src/claude_mcp_plugin/manifest.json` |
| 원본 저장소 | `(원본 저장소)` |
| 상세 분석 | `(로컬 분석 문서)` |
