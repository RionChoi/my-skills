---
name: bytebytego-diagram
version: 2.0
description: >
  ByteByteGo 스타일의 인터랙티브 시스템 아키텍처 다이어그램 에디터를 생성합니다.
  다크 배경, 컬러 노드, 유동적 점선 애니메이션, 3D SVG 아이콘에 더해
  줌/팬, 노드 드래그 이동, 텍스트 인라인 편집, 노드 추가/삭제,
  연결선(엣지) 추가/삭제/색상/스타일/방향 반전, 컬러 테마 조절 기능을 포함합니다.
  순수 HTML + SVG + CSS + Vanilla JS만 사용하며 외부 라이브러리 없음.

  트리거 키워드:
  "아키텍처 다이어그램", "시스템 흐름도", "ByteByteGo 스타일",
  "마이크로서비스 다이어그램", "API 흐름도", "인터랙티브 다이어그램",
  "편집 가능한 다이어그램", "다이어그램 에디터"
---

# ByteByteGo Diagram Skill v2.0

오늘 세션에서 완성된 최종 인터랙티브 다이어그램 에디터 스킬입니다.
참조 파일: `references/editor_final.html` (완성 소스), `references/topics.md` (주제별 확장)

---

## 1. 전체 구조 원칙

### HTML 레이아웃
```
body (flex column, height:100vh, overflow:hidden)
├── #toolbar     (flex-shrink:0) — 툴바
├── #canvas      (flex:1, min-height:0) ← 반드시 min-height:0 필수!
│   └── svg#cvs  (position:absolute, width/height:100%)
│       └── g#W  (transform="translate(vp.x,vp.y) scale(vp.s)")
│           ├── g#EL  — 엣지 레이어
│           └── g#NL  — 노드 레이어
└── #sbar        (flex-shrink:0) — 상태바
```

> ⚠️ **핵심 버그 주의**: `#canvas`에 `min-height:0` 없으면 flex 자식의 height가 0이 되어 캔버스가 보이지 않음.

### SVG 뷰포트 패턴
```js
let vp = { x:20, y:20, s:1 };

function applyVP(){
  W.setAttribute('transform', `translate(${vp.x},${vp.y}) scale(${vp.s})`);
  document.getElementById('zoom-lbl').textContent = Math.round(vp.s*100)+'%';
}
// 반드시 renderAll() 전에 applyVP() 먼저 호출
applyVP();
renderAll();
setTimeout(fitView, 120);
```

---

## 2. 노드 시스템

### 노드 데이터 구조
```js
{
  id: 'c1',
  x: 50, y: 190, w: 118, h: 52,
  label: 'Client',
  sub: 'Browser/App',
  sub2: '...',        // 선택적 3번째 줄
  type: 'client',
  clr: { b:'#3b82f6', g:'#0a1628', t:'#93c5fd' }  // 개별 컬러 오버라이드
}
```

### 노드 타입 & 기본 컬러
```js
const NTYPES = [
  {id:'client',  b:'#2d3d55', g:'#131b2e', t:'#cbd5e1'},
  {id:'gateway', b:'#3b82f6', g:'#0a1628', t:'#93c5fd'},
  {id:'auth',    b:'#6d28d9', g:'#120b28', t:'#c4b5fd'},
  {id:'service', b:'#059669', g:'#071a12', t:'#6ee7b7'},
  {id:'cache',   b:'#92400e', g:'#160f03', t:'#fbbf24'},
  {id:'storage', b:'#374151', g:'#0d1117', t:'#9ca3af'},
  {id:'queue',   b:'#b45309', g:'#1a0f04', t:'#fcd34d'},
  {id:'notif',   b:'#9d174d', g:'#190b13', t:'#f9a8d4'},
];
```

### 노드 블록 내부 y 배치 규칙
```js
const hasSub2 = !!n.sub2;
const y1 = n.y + n.h * (hasSub2 ? 0.36 : 0.40);  // 제목 (아이콘과 동일 y)
const y2 = n.y + n.h * (hasSub2 ? 0.65 : 0.72);  // 서브텍스트
const y3 = n.y + n.h * 0.86;                       // sub2 (있을 때만)
// 아이콘 위치: translate(n.x+9, n.y+n.h/2-9)
// 제목 텍스트 x: n.x+33 (아이콘 오른쪽)
```

### 노드 SVG 구조
```svg
<g id="ng-{id}" class="nd-g">
  <circle />                          <!-- 배경 링 -->
  <rect class="nb" rx="10"/>          <!-- 블록 본체 -->
  <g transform="translate(x+9,y+h/2-9)"> <!-- 3D 아이콘 -->
  <text dominant-baseline="central">  <!-- 제목 -->
  <text text-anchor="middle">         <!-- 서브 -->
  <rect class="nh" fill="transparent"/> <!-- 히트 영역 -->
</g>
```

---

## 3. 엣지(연결선) 시스템

### 엣지 데이터 구조
```js
{ id:'e1', f:'c1', t:'c2', color:'#94a3b8', slow:false, dash:'4 6' }
```

### context-stroke 마커 (색상 자동 연동)
```svg
<marker id="arr" viewBox="0 0 10 10" refX="9" refY="5"
        markerWidth="5" markerHeight="5" orient="auto-start-reverse">
  <path d="M1 2L8 5L1 8" fill="none" stroke="context-stroke"
        stroke-width="1.8" stroke-linecap="round"/>
</marker>
<!-- 사용: marker-end="url(#arr)" — 엣지 stroke 색상 자동 상속 -->
```

### 엣지 경로 (베지어 커브)
```js
function ePath(f, t){
  const fx=f.x+f.w, fy=f.y+f.h/2, tx=t.x, ty=t.y+t.h/2;
  if(Math.abs(fy-ty) < 5) return `M${fx} ${fy} L${tx} ${ty}`;
  const mx = fx + Math.max(18, (tx-fx)*.44);
  return `M${fx} ${fy} C${mx} ${fy} ${mx} ${ty} ${tx} ${ty}`;
}
```

### 엣지 CSS
```css
@keyframes dash-f { to { stroke-dashoffset: -30; } }
@keyframes dash-s { to { stroke-dashoffset: -42; } }

.ep { fill:none; stroke-dasharray:6 5; stroke-linecap:round; cursor:pointer; }
.ep.on     { stroke-width:2; opacity:.9; animation:dash-f .95s linear infinite; }
.ep.on.sl  { animation:dash-s 1.45s linear infinite; }
.ep.off    { stroke-width:1; opacity:.12; stroke-dasharray:3 8; animation:none; }
.ep.sel    { stroke-width:3.5!important; filter:drop-shadow(0 0 5px currentColor); }
```

---

## 4. 인터랙션 기능

### 줌 / 팬
```js
// 마우스 중심 기준 줌
function zoomBy(f, ox, oy){
  const ns = Math.max(.15, Math.min(4, vp.s*f));
  vp.x = ox-(ox-vp.x)*(ns/vp.s);
  vp.y = oy-(oy-vp.y)*(ns/vp.s);
  vp.s = ns; applyVP();
}
canvas.addEventListener('wheel', e=>{
  e.preventDefault();
  const r=canvas.getBoundingClientRect();
  zoomBy(e.deltaY<0?1.1:.91, e.clientX-r.left, e.clientY-r.top);
},{passive:false});
```

### SVG 좌표 변환 (드래그 필수)
```js
function svgP(e){
  const r=canvas.getBoundingClientRect();
  return { x:(e.clientX-r.left-vp.x)/vp.s, y:(e.clientY-r.top-vp.y)/vp.s };
}
```

### 노드 삭제 주의
```js
// ⚠️ SVG 이벤트 루프에서 confirm() 사용 금지 → 이벤트 블로킹 버그
// 우클릭 컨텍스트 메뉴 / 툴바 버튼 / Delete 키 방식만 사용

function delNode(id){
  N = N.filter(x=>x.id!==id);
  E = E.filter(x=>x.f!==id && x.t!==id);  // 연결 엣지 자동 삭제
  document.getElementById('ng-'+id)?.remove();
  reEdges();
}
```

### 텍스트 인라인 편집
```js
function inlineEdit(id, field){
  const r=canvas.getBoundingClientRect();
  const lx=(n.x+n.w/2)*vp.s+vp.x+r.left;
  const ly=(n.y+n.h*.38)*vp.s+vp.y+r.top;
  const inp=document.createElement('input');
  inp.className='ied'; // position:fixed
  inp.style.left=(lx-60)+'px'; inp.style.top=(ly-14)+'px';
  document.body.appendChild(inp);
  inp.focus(); inp.select();
  inp.addEventListener('blur',()=>{ n[field]=inp.value.trim()||n[field]; inp.remove(); rNode(id); });
}
```

### 엣지 연결 모드
```js
let cSrc=null, prev=null;

function startConn(id){ cSrc=id; canvas.classList.add('connecting'); }

function finishConn(toId){
  E.push({id:'e_'+Date.now(), f:cSrc, t:toId,
          color:gT(gN(cSrc).type).b, slow:false});
  cancelConn(); reEdges();
}
// reEdges()에서 prev(프리뷰 선)도 함께 렌더
```

### 엣지 조작
```js
function applyEStyle(v){
  const e=gE(selE);
  if(v==='normal'){delete e.dash; e.slow=false;}
  if(v==='dashed'){e.dash='4 5'; e.slow=true; }
  if(v==='dotted'){e.dash='1 6'; e.slow=false;}
  reEdges();
}
function reverseEdge(){ [e.f,e.t]=[e.t,e.f]; reEdges(); }
function deleteSelEdge(){ E=E.filter(x=>x.id!==selE); closeEP(); reEdges(); }
```

---

## 5. 컬러 시스템

### 테마 프리셋
```js
const THEMES = {
  dark:   null,  // 타입 기본값 사용
  ocean:  {b:'#0ea5e9', g:'#082f49', t:'#38bdf8'},
  forest: {b:'#16a34a', g:'#052e16', t:'#4ade80'},
  sunset: {b:'#c2410c', g:'#2d0c00', t:'#fb923c'},
  mono:   {b:'#4b5563', g:'#111827', t:'#d1d5db'},
};
```

### 노드별 개별 컬러 (히든 피커 패턴)
```js
// HTML: <input type="color" id="gp" style="position:fixed;opacity:0;width:0;height:0"/>
function pickClr(nid, prop, cur){
  const pk=document.getElementById('gp');
  pk.value=cur;
  pk.oninput=e=>{ if(!n.clr)n.clr={}; n.clr[prop]=e.target.value; rNode(nid); };
  pk.click();
}
```

---

## 6. 3D 아이콘 (20×20px)

```js
function mkIcon(type, x, y, bc){
  const T=`<g transform="translate(${x},${y})" style="pointer-events:none">`;
  switch(type){
    case 'client':  return T+`<!-- 노트북: 화면 레이어 3단 + 힌지 + 본체 -->`;
    case 'gateway': return T+`<!-- 허브: 상단 LED 2개 + 포트 4개 + 측면 path -->`;
    case 'auth':    return T+`<!-- 방패: 3단 path + 체크마크 stroke -->`;
    case 'service': return T+`<!-- 사람: 머리 circle 3단 + 몸 path 2단 -->`;
    case 'cache':   return T+`<!-- 원통: 상하 ellipse + rect 몸통 + 줄무늬 -->`;
    case 'storage': return T+`<!-- 적층 디스크: ellipse 4개 + rect 3개 -->`;
    case 'queue':   return T+`<!-- 쇼핑카트: 바디 path 3단 + 바퀴 circle -->`;
    case 'notif':   return T+`<!-- 벨: 곡선 path 3단 + 클래퍼 circle -->`;
  }
}
// 실제 SVG 코드는 references/editor_final.html의 mkIcon 함수 참조
```

---

## 7. 핵심 버그 & 주의사항

| 버그 | 원인 | 해결 |
|---|---|---|
| 캔버스 높이 0 | flex 자식 `min-height:0` 누락 | `#canvas { flex:1; min-height:0; }` |
| 노드 삭제 불작동 | SVG 내 `confirm()` 블로킹 | `confirm()` 제거, 즉시 삭제 |
| 드래그 좌표 오차 | scale 미적용 | `svgP()` 함수 항상 사용 |
| applyVP 미반영 | renderAll 전 호출 누락 | `applyVP()` → `renderAll()` 순서 |
| 화살표 색 불일치 | 마커 stroke 하드코딩 | `stroke="context-stroke"` 사용 |

---

## 8. 키보드 단축키

| 키 | 동작 |
|---|---|
| `Delete` / `Backspace` | 선택 노드/엣지 삭제 |
| `Escape` | 연결 모드 취소 / 선택 해제 |
| `+` | 줌 인 |
| `-` | 줌 아웃 |
| `0` | fitView 전체 맞춤 |

---

## 9. 주제별 확장 → `references/topics.md`

| 주제 | 컬러 | 핵심 노드 |
|---|---|---|
| API Gateway | `#3b82f6` | Client→GW→Services→DB |
| Kafka | `#f97316` | Producer→Broker→Consumer |
| CDN | `#0d9488` | Origin→Edge→Client |
| K8s | `#2563eb` | Ingress→Pod→Service→PV |
| DB 샤딩 | Gray+Green | Shard1~N + Router |
| Rate Limiting | `#dc2626` | Token Bucket / Sliding Window |
