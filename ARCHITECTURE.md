# Finance App — 구조 & 확장 가이드

개인 주식 포트폴리오 대시보드 + 계좌별 리밸런싱 + 매일 아침 카톡 알림 시스템.
이 문서는 추후 **계좌 추가/전략 변경 시 참조**용입니다.

- **로컬 디렉토리**: `~/Documents/Finance_app` (iCloud Drive 동기화 폴더)
- **공개 호스팅**: https://ew-jang.github.io/finance-dashboard (GitHub Pages)
- **GitHub repo**: `ew-jang/finance-dashboard`

---

## 1. 핵심 설계 원칙

1. **데이터는 Mac에서만 관리**, iPhone은 GitHub Pages에서 읽기만 한다.
2. **계좌별로 파일/전략 분리** — 계좌마다 투자 전략이 다르므로 독립적으로 확장 가능하게 네임스페이스를 나눈다.
3. **공개 repo에는 대시보드(HTML)와 데이터(JSON)만** 올린다. 알림 스크립트·동기화 스크립트는 로컬 전용(`.gitignore`).

---

## 2. 파일 인벤토리

| 파일 | 공개여부 | 역할 |
|------|---------|------|
| `index.html` | 공개 | 메인 대시보드 (F&G, 종목 신호, 차트) + 계좌별 리밸런싱 페이지 링크 허브 |
| `rebal_toss.html` | 공개 | **토스증권 계좌** 리밸런싱 페이지 (S4 4단계 레짐 전략, USD) |
| `pf_snaps_toss.json` | 공개 | 토스증권 보유 스냅샷 배열 (대시보드가 fetch) |
| `rebal_pension.html` | 공개 | **개인연금 계좌** 리밸런싱 페이지 (전략적 자산배분, KRW) |
| `pf_snaps_pension.json` | 공개 | 개인연금 보유 스냅샷 배열 |
| `signal_check.py` | 로컬 | 종목 신호 + 리밸런싱 필요 여부 분석 → 카톡 메시지 문자열 출력 |
| `send_alert.sh` | 로컬 | `signal_check.py` 실행 후 Claude CLI로 카톡 발송 (7AM) |
| `sync_snaps.sh` | 로컬 | 30초마다 Downloads/repo의 `pf_snaps_*.json` 변경분을 git push |
| `alert.log` | 로컬 | 알림 실행 로그 (append) |
| `.gitignore` | 로컬 | 로컬 전용 파일 목록 |

> 계좌가 늘면 `rebal_<계좌>.html` + `pf_snaps_<계좌>.json`이 추가된다. `sync_snaps.sh`는 `pf_snaps_*.json` 와일드카드라 **수정 불필요**.

---

## 3. 데이터 모델 — `pf_snaps_<계좌>.json`

날짜별 스냅샷 배열. **마지막 원소 = 최신 보유 현황**.

```json
[
  {
    "date": "2026-06-21",   // YYYY-MM-DD (같은 날짜는 덮어씀)
    "tqqq": 45,             // TQQQ 보유 주수
    "qqqm": 0,              // QQQM 보유 주수
    "usd": 7441,            // 보유 달러 현금 ($)
    "tqqqPx": 82.87,        // 저장 시점 TQQQ 시세
    "qqqmPx": 304.52,       // 저장 시점 QQQM 시세
    "usdkrw": 1529.89,      // 저장 시점 환율
    "totalUsd": 11170.15,   // 총 평가액 ($)
    "totalKrw": 17089101    // 총 평가액 (₩)
  }
]
```

> 다른 종목 구성을 쓰는 계좌라면 키 이름(tqqq/qqqm/usd)을 그 계좌 전략에 맞게 바꾸고, 해당 `rebal_<계좌>.html`과 `signal_check.py`도 같이 맞춘다.

---

## 4. 데이터 동기화 파이프라인

```
[Mac] rebal_*.html 에서 보유 현황 저장
   │
   ├─(Safari) pf_snaps_<계좌>.json 을 ~/Downloads 로 다운로드
   │            │
   │            └─ sync_snaps.sh (launchd, 30초) → Downloads에서 repo로 복사
   │
   └─(Chrome) File System Access API로 iCloud 폴더(repo)에 직접 쓰기
                │
                └─ sync_snaps.sh 가 repo 변경분 감지
   ▼
git add pf_snaps_*.json → commit → push
   ▼
[GitHub Pages] pf_snaps_<계좌>.json 서빙
   ▼
[iPhone/Mac] 페이지 로드 시 syncSnapsFromGitHub() 가 fetch → localStorage 병합 → 렌더
```

### launchd 잡: `com.finance.snapswatcher`

- 위치: `~/Library/LaunchAgents/com.finance.snapswatcher.plist`
- 30초 간격(`StartInterval`)으로 `sync_snaps.sh` 실행
- 실행 방식: **`osascript`로 감싸 bash 호출** — 아래 TCC 주의 참고
- 로그: `/tmp/snapswatcher.log`, `/tmp/snapswatcher.err`

```bash
# 재적용
launchctl unload ~/Library/LaunchAgents/com.finance.snapswatcher.plist
launchctl load   ~/Library/LaunchAgents/com.finance.snapswatcher.plist
# 상태/마지막 종료코드
launchctl print gui/$(id -u)/com.finance.snapswatcher | grep -i "last exit"
```

### ⚠️ TCC (Full Disk Access) 주의 — 가장 중요한 함정

macOS TCC는 launchd가 띄운 프로세스의 `~/Downloads`·`~/Documents` 접근을 막는다.
**Full Disk Access는 `/usr/bin/osascript`에 부여**되어 있다 (System Settings → Privacy & Security → Full Disk Access).

- 그래서 plist는 `/bin/bash`를 직접 부르지 않고 `osascript -e 'do shell script "/bin/bash …sync_snaps.sh"'` 형태로 호출한다. 자식 bash가 osascript의 권한을 상속받는다.
- `/bin/bash`로 직접 바꾸면 `Operation not permitted (exit 126)` 발생. 굳이 바꾸려면 `/bin/bash`에 별도로 Full Disk Access를 줘야 한다.

---

## 5. 리밸런싱 전략 — S4 (4단계 레짐)

QQQM 200일 이평선 + VIX 기준으로 시장 레짐 판정 → 권장 비중 결정.

| 레짐 | 조건 | TQQQ | QQQM | USD |
|------|------|-----:|-----:|----:|
| 🔴 하락장 | QQQM < 200MA | 0% | 20% | 80% |
| 🟡 공포 | VIX ≥ 30 | 15% | 35% | 50% |
| 🟠 중립 | VIX 20–30 | 40% | 35% | 25% |
| 🟢 황금 | VIX < 20 | 65% | 25% | 10% |

> 이 표는 **두 곳에 중복 정의**되어 있다 (한쪽 수정 시 다른 쪽도):
> - `rebal_toss.html` 의 `REBAL` 배열 (JS)
> - `signal_check.py` 의 `REBAL_ALLOCS` / `REGIME_LABELS` (Python)
>
> 계좌별 전략이 다르면 각 계좌의 html이 자기 `REBAL` 배열을 갖는다.

### 5-2. 개인연금 전략 — 전략적 자산배분(코어-새틀라이트)

연금계좌는 **레버리지/파생 제한 + 장기 보유**라 레짐 타이밍이 부적합하다. 대신 고정 목표 비중 + 리밸런싱 밴드 방식.

- **코어**: KODEX 미국S&P500(`379800.KS`, KRW) — 분산·안정
- **새틀라이트**: TIGER 미국테크TOP10 INDXX(`381170.KS`, KRW) — 성장·집중(상위 10종목)
- **목표 비중(기본)**: S&P500 70% / Tech10 30% (페이지 슬라이더로 50~90% 조절, `pension_target_sp`에 저장)
- **리밸런싱 밴드**: 실제 비중이 목표에서 ±5%p 이탈 시 조정 (`pension_band`)
- **조정 원칙**: 매도보다 **신규 납입금으로 부족분 매수** 우선
- 데이터는 전부 KRW (환율 변환 없음). 스냅샷 키: `sp500`, `tech10`, `sp500Px`, `tech10Px`, `totalKrw`

> 토스(USD·레짐)와 연금(KRW·고정배분)은 **데이터 모델·전략이 완전히 다른** 좋은 확장 예시다.
> 새 계좌는 둘 중 성격이 가까운 쪽을 템플릿으로 복사하면 된다.

---

## 6. 대시보드

### `index.html`
- CNN Fear & Greed 게이지, 종목 매수/매도 신호, 종목별 1년 차트
- 상단 **"계좌별 리밸런싱"** 카드 섹션 = 각 `rebal_*.html`로 가는 허브
- 시세: Yahoo Finance v8 API + CORS 프록시(`corsproxy.io`) 폴백

### `rebal_*.html` (계좌별)
- 좌측: 현재 레짐 배지 / 권장 구성 / 현재 보유 평가
- 우측: 보유 현황 입력(날짜·TQQQ·QQQM·USD) → 저장
- 레짐별 권장 수량 비교표 + 자산 변화 차트(Plotly) + 입력 기록 테이블
- ☁️ iCloud 폴더 연결(Chrome 전용, Mac 최초 1회)

#### localStorage 네임스페이스 (계좌별로 반드시 분리)
| 키 | 용도 |
|----|------|
| `pf_snaps_<계좌>` | 스냅샷 배열 캐시 |
| `rebal_krw_<계좌>` | 권장 구성 기준 금액 |

---

## 7. 알림 시스템 (매일 아침 7시)

- **클라우드 스케줄 에이전트**가 07:00 KST(22:00 UTC)에 트리거
- 흐름: `send_alert.sh` → `signal_check.py` 실행 → 신호 있으면 Claude CLI의 KakaoTalk MCP로 발송
- `signal_check.py` 로직:
  - 레버리지 ETF(TQQQ/QLD): 추세추종(200MA·50MA·VIX)
  - 일반 종목: 역추세(RSI·MDD낙폭·F&G)
  - 2개 이상 조건 충족 시 매수/매도 신호
  - 리밸런싱: `pf_snaps_toss.json` 최신 스냅샷 vs 레짐 권장 비중 비교 → 차이나면 알림

> 현재 알림은 **토스증권 계좌만** 본다 (`HOLDINGS_FILE = pf_snaps_toss.json`). 다른 계좌 알림이 필요하면 8번 참고.

---

## 8. 🔧 새 계좌 추가하는 법

예: KB증권 배당 계좌(`kb`) 추가.

1. **리밸런싱 페이지 생성**
   `rebal_toss.html` → `rebal_kb.html` 복사 후 아래를 모두 `kb`로 치환:
   - `<title>`, `<h1>` 제목
   - `SNAPS_URL` → `…/pf_snaps_kb.json`
   - `loadSnaps/saveSnaps` 의 localStorage 키 → `pf_snaps_kb`
   - `rebal_krw_toss` → `rebal_krw_kb` (2곳)
   - `downloadSnaps` 의 다운로드 파일명 → `pf_snaps_kb.json`
   - `writeToICloud` 의 `getFileHandle('pf_snaps_kb.json', …)`
   - `REBAL` 배열 → 그 계좌 전략에 맞는 비중표로 교체

2. **데이터 시드 파일**
   `pf_snaps_kb.json` 생성 (`[]` 또는 초기 스냅샷 1개).

3. **메인 허브에 카드 추가**
   `index.html` "계좌별 리밸런싱" 섹션의 주석 `<!-- 계좌 추가 시 여기에 카드 추가 -->` 위치에:
   ```html
   <a href="rebal_kb.html" style="…동일 스타일…">
     <span>🏦 KB증권</span>
     <span>배당 전략 · …</span>
   </a>
   ```

4. **동기화** — `sync_snaps.sh`는 `pf_snaps_*.json` 와일드카드라 자동 처리. **수정 불필요.**

5. **알림(선택)** — 그 계좌도 카톡 알림을 받으려면 `signal_check.py`를 다계좌 루프로 확장하거나 별도 `HOLDINGS_FILE`을 추가.

6. **커밋** — `rebal_kb.html`, `pf_snaps_kb.json`, `index.html` 만 git add/push (스크립트는 gitignore).

---

## 9. 알아둘 함정 / 메모

- **시세 조회 실패**: Yahoo가 CORS 막으면 `corsproxy.io` 폴백. VIX 티커는 `^VIX` (URL 인코딩 `%5EVIX`로 넣으면 이중 인코딩되어 실패하니 주의).
- **주말/공휴일 저장**: 해당 날짜 시세가 없으면 최신 시세(`_rd`)로 폴백 저장.
- **두 곳 동기화 필요**: 레짐 비중표(5번)는 html·py 양쪽에 있다.
- **git push 자격증명**: launchd 컨텍스트에서 osxkeychain 캐시 자격증명을 사용. 토큰 만료 시 push 무음 실패 가능 → `/tmp/snapswatcher.err` 확인.
- **데이터 흐름 단방향**: 편집은 Mac에서만. iPhone에서 저장하면 GitHub로 안 올라가고 로컬 localStorage에만 남는다.
