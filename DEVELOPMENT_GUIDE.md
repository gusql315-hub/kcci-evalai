# 개발 가이드 (Development Guide)

이 문서는 `index.html` 하나로 구성된 「대한상공회의소 교육평가 AI」 목업을 개발할 때
**일관된 UI/UX**를 유지하기 위한 규칙을 정리합니다. 새로운 화면이나 기능을 추가하기 전에
반드시 이 문서를 먼저 확인하세요.

## 핵심 원칙: 기능은 공용 컴포넌트로 관리한다

> 같은 동작(모달, 드로어, 버튼, 배지, 페이지네이션 등)을 화면마다 새로 만들지 않습니다.
> 공용 함수를 하나만 고치면 그 함수를 쓰는 모든 화면에 동일하게 적용되어야 합니다.
> 그래야 "여기는 되는데 저기는 안 된다" 같은 불일치가 생기지 않습니다.

규칙:
1. 새 UI 패턴이 필요하면 먼저 아래 "기존 공용 컴포넌트 목록"에 이미 쓸 수 있는 것이 있는지 확인합니다.
2. 없다면 커스텀 컴포넌트(함수)를 새로 만들 수 있습니다. 단, 그 자리에서 한 번만 쓰고 끝내지 말고
   **재사용 가능한 함수로 분리**하고, 이 문서의 목록에도 반드시 추가합니다.
3. 컴포넌트의 동작(닫기 방식, 접근 권한 체크, 스타일 등)을 고치고 싶을 때는 그 컴포넌트를 쓰는
   모든 화면을 하나하나 고치는 게 아니라, **공용 함수 자체를 고쳐서** 전체에 반영합니다.

## 기존 공용 컴포넌트 목록

### 1. 모달 · 드로어 (팝업 UI)

플랫폼에서 팝업(가운데 모달)이나 옆에서 슬라이드되는 패널(드로어)이 필요하면
반드시 아래 함수를 통해서만 열고 닫습니다. 직접 `<dialog>`를 만들거나 `showModal()`을 새로 호출하지 않습니다.

| 함수 | 역할 |
|---|---|
| `openModal(title, description, body, foot)` | 가운데 모달(`#modal`)을 연다. |
| `openDrawer(title, description, body, foot, size='sm')` | 오른쪽 드로어(`#drawer`)를 연다. `size:'lg'`로 넓은 드로어 가능. |
| `closeDrawer()` | 드로어를 닫기만 한다(상태 초기화 없음). |
| `dismissDrawer()` | 드로어와 관련된 `state`(`analysisDetail`, `drawer`)를 초기화하고 닫는다. **드로어를 닫을 때는 항상 이 함수를 사용한다** — `render()`가 매번 `state`를 기준으로 드로어를 다시 그리기 때문에, 상태를 지우지 않고 DOM만 닫으면 다음 렌더링에서 드로어가 다시 열려 보일 수 있다. |
| `wireDialogDismiss(dialog, onDismiss)` | 다이얼로그의 배경(백드롭) 클릭 시 닫히도록 연결한다. `onDismiss`를 생략하면 단순 `.close()`, 지정하면 그 함수를 호출한다(드로어는 `dismissDrawer`를 넘겨 상태까지 함께 정리). |

**배경 클릭으로 닫기(백드롭 딤 클릭 dismiss)**는 부트스트랩 시점에 한 번만 연결합니다(`boot()` 직전):
```js
wireDialogDismiss($('entityModal'));
wireDialogDismiss($('modal'));
wireDialogDismiss($('drawer'), dismissDrawer);
```
새로운 다이얼로그를 추가할 경우, 반드시 여기에 한 줄을 추가해서 동일하게 배경 클릭 닫기가 동작하도록 합니다.

동작 원리: `<dialog>`를 `showModal()`로 열면 브라우저가 자동으로 `::backdrop`을 만들어 페이지 나머지 영역을 어둡게 하고 상호작용을 막습니다(포커스 트랩 포함, 닫히면 포커스는 원래 위치로 자동 복원됨 — 표준 동작). 배경(패딩 바깥 어두운 영역)을 클릭하면 클릭 이벤트의 `target`이 `dialog` 엘리먼트 자신이 되는 특성을 이용해 `e.target===dialog`일 때만 닫습니다. 다이얼로그 내부 콘텐츠를 클릭했을 때는 target이 내부 요소이므로 닫히지 않습니다.

### 2. 기본 UI 조각

| 함수 | 역할 |
|---|---|
| `button(label, action, cls='', extra='')` | `data-action` 속성이 달린 버튼 HTML을 만든다. 전역 클릭 위임 리스너가 `data-action` 값을 보고 동작을 실행한다. |
| `badge(text, tone='')` | 상태 표시용 배지(칩) HTML. |
| `icon(name)` | 사이드바 등에서 쓰는 인라인 SVG 아이콘. |
| `heading(name, description, action='')` | 페이지 상단 제목 + 설명 + (선택) 버튼 영역. |

### 3. 목록 페이지네이션

| 함수 | 역할 |
|---|---|
| `pageSlice(list, key)` | 배열을 `state.pagination[key]` 기준으로 페이지 단위로 자른다. |
| `paginationControls(key, pg)` | 페이지 이동 버튼 + 페이지당 개수 선택 UI를 렌더링한다. |

목록 화면을 새로 만들 때는 이 두 함수를 그대로 재사용합니다(직접 slice/버튼을 새로 만들지 않음).

### 4. 권한/스코프 판단

| 함수 | 역할 |
|---|---|
| `accessibleBusinesses(user, d)` | 로그인한 사용자가 접근 가능한 사업 목록. |
| `canManageBusiness(businessId)` | 현재 사용자가 해당 사업의 데이터를 등록·수정·삭제할 수 있는지. |
| `sharesBusinessWith(user, targetUser)` | 두 사용자가 담당 사업을 공유하는지(사업 관리자의 사용자 관리 화면 범위 제한에 사용). |
| `bizLabels(businessId, d)` / `bizEvalMode(businessId, d)` | 사업별 커스텀 명칭 · 평가 운영방식. |
| `navAllowed(entry)` | 사이드바 메뉴 항목을 현재 role로 노출할지 판단. |

새 화면에서 데이터 접근 범위나 버튼 노출 여부를 판단할 때는 이 함수들을 재사용하고, 화면마다 별도의
role 체크 로직을 새로 작성하지 않습니다.

### 5. 평가 항목 템플릿 (중간평가/최종평가 공용)

「평가 항목 관리」 화면(중간평가 항목 / 최종평가 항목 탭)은 하나의 공용 템플릿 시스템으로 구현되어 있습니다.
새로운 평가 종류(예: 만족도 재조사 등)가 추가되더라도 이 시스템에 종류를 하나 더 등록하는 방식으로 확장합니다 —
관리 화면·편집 다이얼로그를 새로 만들지 않습니다.

| 함수/상수 | 역할 |
|---|---|
| `TEMPLATE_KINDS` | 템플릿 종류 정의(`interim`/`final`) — localStorage 키, 표시 라벨, 평가구분 문자열, 페이지네이션 키를 담는다. 새 템플릿 종류를 추가하려면 여기에 항목을 추가한다. |
| `templateRows(kind)` / `saveTemplateRows(kind, rows)` | 종류별 항목 목록을 읽고 저장한다. |
| `normalizeTemplateRow(r, i, defaultChoiceIndex)` | 항목 하나를 `{category, question, type:'score'\|'choice'\|'narrative', scale, options}` 형태로 정규화한다. |
| `evalTemplateManagement()` / `editTemplate(kind)` / `templateEditRow(r, i)` / `saveEvalTemplate()` | 관리 화면·편집 다이얼로그·행 렌더링·저장 로직. `state.evalTemplateTab`으로 현재 탭(kind)을 관리한다. |

문항 유형은 `score`(점수형) · `choice`(선택형) · `narrative`(서술형) 세 가지입니다. 서술형은 척도/선택지 설정이 없고,
대신 아래 "서술형 응답 매칭·분석" 엔진이 응답 Excel/CSV에서 실제 텍스트를 추출해 분석합니다.

### 6. 서술형 응답 매칭·분석 엔진 (Narrative response matching)

응답 Excel/CSV의 열 제목을 평가 항목 관리에 등록된 질문과 매칭해, 점수형은 평균을 자동 계산하고
서술형은 실제 응답 텍스트를 추출해 키워드 빈도 기반으로 요약합니다. **원본에 없는 의견을 새로 만들지 않는다는
원칙**을 지키므로, 새로운 "AI 분석" 기능을 추가할 때도 이 엔진을 우선 재사용하세요.

| 함수 | 역할 |
|---|---|
| `matchHeadersToTemplate(kind, headers)` | Excel/CSV 헤더를 템플릿 질문과 매칭(대괄호 태그 제거 후 비교). |
| `averageColumn(rows, colIndex)` | 점수형 열의 평균을 계산(빈 값·숫자가 아닌 값 제외). |
| `extractNarrativeColumn(rows, colIndex)` / `buildNarrativeItem(row, colIndex, rows)` | 서술형 열에서 실제 응답을 추출하고, 건수·요약·반복 키워드를 계산한다. |
| `keywordFrequency(responses, topN)` / `tokenizeNarrative(text)` | 실제 응답 텍스트 기반 키워드 빈도 계산(불용어 제외). |
| `classifyNarrativeSentiment(text)` | 긍정/부정 단어 사전 기반의 단순 감성 분류. |
| `buildNarrativeAnalysis(matchedNarrativeCols, rows)` | 서술형 문항 분석(`narrativeItems`)과, 기존 canned 분석 화면(키워드/의견/개선방안/참여자별 피드백)과 **동일한 데이터 모양**으로 만든 `adapter`(실응답 기반 대체 데이터)를 함께 반환한다. `applyTemplateMatching()`(중간평가 자동 분석)과 `saveFinalEntry()`(최종평가 서술형 파일 업로드)가 이 함수를 공유한다. |
| `parseXLSXFile(file)` (Services) | JSZip(이미 내장됨)으로 `.xlsx`를 직접 읽어 `{headers, rows}`로 반환한다. `.xlsx`도 CSV와 동일하게 실제 매칭에 사용된다. |

새 화면에서 "실응답 기반인지 canned 샘플인지"를 구분해야 할 때는 `analysis.realNarrative` 플래그를 확인하세요
(예: `midDiagnosis()`는 실응답 기반일 때 표시하지 않습니다 — 고정된 canned 진단 문구가 실제 데이터와 맞지 않을 수 있기 때문).

## 새 컴포넌트를 추가할 때 체크리스트

- [ ] 기존 공용 컴포넌트로 해결되지 않는지 먼저 확인했다.
- [ ] 재사용 가능한 함수로 분리했다(화면 렌더 함수 안에 로직을 박아넣지 않았다).
- [ ] 이 문서(`DEVELOPMENT_GUIDE.md`)의 표에 함수와 역할을 추가했다.
- [ ] 같은 성격의 다른 화면에도 적용 가능한지 확인하고, 가능하면 그 화면들도 새 공용 함수를 쓰도록 맞췄다.

## 참고: 이 프로젝트의 성격

`index.html` 하나로 동작하는 프론트엔드 전용 목업이며, 데이터는 브라우저 `localStorage`에만 저장됩니다
(실제 서버·이메일 발송 없음). 프레임워크 없는 순수 JS이므로 "공용 컴포넌트"는 리액트 컴포넌트가 아니라
**재사용 가능한 JS 함수 + CSS 클래스**를 의미합니다.
