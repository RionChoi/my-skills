# ByteByteGo 다이어그램 — 주제별 확장 레퍼런스

SKILL.md의 핵심 패턴을 바탕으로 주제별 세부 구성입니다.

---

## API Gateway 아키텍처 (기본 템플릿)

**노드 구성**
```
Client → API Gateway → Auth Service
                    → User Service  → Redis
                    → Order Service → PostgreSQL
                    → Notif Service
```

**엣지 라우팅 (겹침 방지 x값)**
- GW→Auth:  좌측 꺾기 `x=222`, `y=108`
- GW→User:  직선 `y=224`
- GW→Order: `x=336` 엘보
- GW→Notif: `x=352` 엘보 (Order와 16px 분리)
- Auth→User 콜백: `x=500`
- Order→Notif: `x=510`

---

## Kafka 이벤트 스트리밍

**노드 구성**
```
Producer ──→ Kafka Broker (Topic) ──→ Consumer Group
                  ↓                        ↓
             Zookeeper              DB / Cache
```

**컬러 포인트**
- Broker: `#1c0f02` / `#ea580c` (주황)
- Producer: `#0f1f0f` / `#16a34a` (초록)
- Consumer: `#0f1628` / `#2563eb` (파랑)
- Topic 컨테이너: 점선 `stroke-dasharray:4 3` 큰 박스

**아이콘 아이디어**
- Broker: 다단 레이어 사각형 (로그 세그먼트 표현)
- Producer: 오른쪽 화살표 묶음
- Consumer: 왼쪽 당기는 화살표

---

## CDN 아키텍처

**노드 구성**
```
User → Edge Node (지역별 3개) → Origin Server
              ↓
         Cache Hit/Miss
```

**컬러 포인트**
- Edge: `#0c1f1f` / `#0d9488` (teal)
- Origin: `#1a1005` / `#b45309` (amber)
- 지역 레이블: 작은 배지 (US-WEST, AP-EAST 등)

**특이 패턴**
- 부채꼴 엣지: Origin에서 여러 Edge로 퍼지는 형태
- 지연 시간 표시: 엣지 중간에 `<text>` 레이블 (12ms, 45ms 등)

---

## Kubernetes 클러스터

**노드 구성**
```
Ingress → Service → Pod (replica 3개)
                       ↓
                  PersistentVolume
```

**컬러 포인트**
- Ingress: `#0a1628` / `#3b82f6`
- Pod: `#071a12` / `#059669` (작은 박스 3개 나란히)
- PV: `#0d1117` / `#374151`

**특이 패턴**
- Pod 3개를 작은 크기로 나란히 배치
- 점선 네임스페이스 컨테이너 박스 (`stroke-dasharray:5 3`)

---

## Rate Limiting (토큰 버킷)

**노드 구성**
```
Request → Rate Limiter → Token Bucket → Allow/Deny
                              ↓
                         Redis Counter
```

**컬러 포인트**
- Limiter: `#1a0505` / `#dc2626` (red)
- Allow: `#071a12` / `#059669` (green)
- Deny: `#1a0505` / `#dc2626` (red)

**특이 패턴**
- 토큰 버킷 아이콘: 원통에 토큰 점들 표현
- Allow/Deny 분기: 다이아몬드 결정 노드

---

## DB 샤딩

**노드 구성**
```
App → Shard Router → Shard 1 (User 1-1M)
                  → Shard 2 (User 1M-2M)
                  → Shard 3 (User 2M-3M)
```

**컬러 포인트**
- Router: `#0a1628` / `#3b82f6`
- 각 Shard: 동일 컬러 계열, 번호로 구분
- 범위 레이블: 각 Shard 아래 작은 텍스트

**특이 패턴**
- Consistent Hashing 링 표현 시: 원형 배치 대신 선형으로
- 핫스팟 표시: 특정 Shard에 amber 하이라이트

---

## 공통 재사용 패턴

### 컨테이너 박스 (점선 그룹핑)
```svg
<rect x="X" y="Y" width="W" height="H" rx="12"
      fill="none" stroke="#374151"
      stroke-width="1" stroke-dasharray="5 3" opacity=".4"/>
<text x="X+10" y="Y+16" style="fill:#4b5563;font-size:9px;
      font-family:var(--font-mono)">namespace: production</text>
```

### 지연 시간 배지
```svg
<rect x="MX-18" y="MY-8" width="36" height="14" rx="4" fill="#1f2937"/>
<text x="MX" y="MY+1" text-anchor="middle" dominant-baseline="central"
      style="fill:#9ca3af;font-size:8px;font-family:var(--font-mono)">12ms</text>
```

### 상태 인디케이터 (초록 점)
```svg
<circle cx="X+W-10" cy="Y+10" r="3" fill="#22c55e"/>
<circle cx="X+W-10" cy="Y+10" r="3" fill="#22c55e" opacity=".3">
  <animate attributeName="r" values="3;6;3" dur="2s" repeatCount="indefinite"/>
  <animate attributeName="opacity" values=".3;0;.3" dur="2s" repeatCount="indefinite"/>
</circle>
```
