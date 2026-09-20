# SysWatch 전수조사 & 활용 분석 (한국어)

> 이 문서는 `syswatch` 저장소를 코드 레벨까지 전수조사하고, 활용 방안과
> 수익화 아이디어까지 정리한 기록이야. 작성: 카리나 ✨

## 📎 관련 링크

| 구분 | 주소 |
|---|---|
| **이 저장소 (포크)** | https://github.com/bmshin94/syswatch |
| 원본(upstream) | https://github.com/matthart1983/syswatch |
| 자매 프로젝트 — NetWatch | https://github.com/matthart1983/netwatch |
| 자매 프로젝트 — DiskWatch | https://github.com/matthart1983/diskwatch |
| netwatch-sdk (의존성) | https://github.com/matthart1983/netwatch-sdk |
| crates.io | https://crates.io/crates/syswatch |
| Repology (패키징 현황) | https://repology.org/project/syswatch/versions |

---

## 1. 한 줄 요약

**`syswatch`는 Rust로 만든 "기억하는" 단일 호스트 시스템 진단 TUI다.**

`htop`이 *지금 무엇이 돌아가는지*를 보여준다면, `syswatch`는
*방금 왜 시스템이 이상해졌는지*를 평문으로 설명한다.

---

## 2. 저장소 기본 정보

| 항목 | 내용 |
|---|---|
| 버전 | v0.14.2 |
| 언어 | Rust (edition 2021, **Rust 1.75+** 필요) |
| 코드 규모 | `src/` 48개 파일 / 약 **24,953줄** |
| 테스트 | `#[test]` **417개** |
| 라이선스 | **MIT** (상업적 이용·개조·재배포 가능) |
| 지원 플랫폼 | **macOS / Linux** (Windows 전용 코드 0줄) |
| 릴리스 타겟 | linux-gnu/musl × x86_64/aarch64, **armv5te-musleabi**, apple-darwin × x86_64/aarch64 |

### 주요 의존성
`ratatui` 0.29 · `crossterm` 0.28 · `sysinfo` 0.32(멀티스레드 off) · `clap` 4 ·
`tokio` 1 · `serde`/`serde_json` · `chrono` · `unicode-width` · `netwatch-sdk` 0.1 ·
`toml` · `dirs` · `postcard` 1 · `zstd` 0.14 · `ctrlc` 3
(opt-in: `nvml-wrapper` = `gpu-nvidia` 피처 / macOS 전용: `macpow` / unix: `libc`)

---

## 3. 폴더 구조

```
src/
├── main.rs (225)          CLI 진입점 + 서브커맨드 분기 + SIGPIPE 복원
├── app.rs (2,299)         이벤트 루프, 탭 상태, 스크럽 플럼빙
├── config.rs (203)        ~/.config/syswatch/config.toml (fail-soft 로드)
├── recording.rs (1,230)   .swr 세션 녹화 포맷 v3 (postcard + zstd)
├── report.rs (480)        비대화형 리포트 (snapshot/insights/diff/why)
├── snapshot.rs            스냅샷 직렬화
├── collect/               서브시스템별 수집기 (14개 파일)
│   ├── collector.rs (983)       sysinfo 기반 CPU/Mem/Procs/Net + 디스패치
│   ├── gpu.rs (892)             ioreg AGXAccelerator / sysfs DRM / nvml
│   ├── macos_sampler.rs (218)   IOReport + SMC 공용 워커 (GPU/전력/팬)
│   ├── proc_bandwidth.rs (939)  프로세스별 대역폭 (백그라운드 스레드)
│   ├── proc_memory.rs (294)     Activity Monitor와 일치하는 메모리 수치
│   ├── power.rs (643)           ioreg / pmset / sysfs power_supply
│   ├── services.rs (227)        launchctl / systemctl
│   ├── sanitize.rs (229)        터미널 이스케이프 인젝션 방어
│   └── ring.rs                  링버퍼 + nth_back (스크럽용)
├── insights/mod.rs (1,671)  ★ 휴리스틱 이상탐지 엔진
├── tabs/                    12개 탭 렌더러 (탭당 파일 1개)
└── ui/
    ├── lite.rs (2,253)      80x24 한 화면 모드
    ├── dense/ (2,594+820)   130x44 전 서브시스템 한 화면
    ├── graph.rs (524)       브라유 그래프 / 페이드 렌더링
    ├── theme.rs (501)       테마 시스템 (terminal 테마 포함)
    ├── palette.rs           색상 단일 진실 공급원
    └── chrome.rs / help.rs / settings.rs / widgets.rs
```

---

## 4. 핵심 기능

### 4.1 12개 탭 — 유닉스 명령어를 하나로

| # | 탭 | 대체 대상 |
|---|---|---|
| 1 | Overview | 전체 대시보드 |
| 2 | CPU | `htop` CPU 패널, `top -d`, `mpstat` |
| 3 | Memory | `free`, `vm_stat` |
| 4 | Disks | `iostat`, `iotop` |
| 5 | Filesystems | `df -h`, `df -i`, `mount` |
| 6 | Procs | `htop`, `ps auxf`, `pstree` |
| 7 | GPU | `ioreg AGXAccelerator` / `/sys/class/drm` |
| 8 | Power | `pmset`, `AppleSmartBattery` / `power_supply` |
| 9 | Services | `launchctl list` / `systemctl list-units` |
| 0 | Net | `nettop`, `iftop` |
| - | **Timeline** | **대체 대상 없음** (세션 로그 + 스크러버) |
| + | **Insights** | **대체 대상 없음** (평문 이상 카드) |

### 4.2 Insights 엔진 — 13개 휴리스틱 + 상관분석

`src/insights/mod.rs`는 전부 **순수 함수**이며 시그니처가
`fn insight_*(h: &History, snap: &Snapshot) -> Option<Insight>` 로 통일돼 있다.

```
insight_swap_thrash              스왑 폭주
insight_runaway_proc             CPU 독식 프로세스
insight_disk_full                디스크 포화 (읽기전용 composefs root 제외 처리)
insight_memory_pressure          메모리 압박
insight_high_load                로드 애버리지 과부하 (코어 수 대비 2x/4x)
insight_zombie_party             좀비 프로세스 5개/25개 임계
insight_gpu_pegged               GPU 지속 풀가동 (90%/98%)
insight_vram_high                VRAM 포화 (85%/95%)
insight_psi_memory               커널 PSI 메모리
insight_psi_io                   커널 PSI I/O
insight_energy_hog               전력 소모 상위 프로세스
insight_mem_leak                 메모리 누수 (약 2분 이상 지속 성장 필요)
insight_cpu_baseline_deviation   머신별 CPU 평소값 대비 이탈
```

**가장 중요한 차별점 — `correlate_shared_culprits()`**

같은 프로세스가 2개 이상의 카드에서 범인(`culprit`)으로 지목되면,
그것을 우연이 아닌 **근본 원인**으로 보고 **CRIT 카드를 합성**한다.
`BTreeMap`을 써서 출력 순서가 결정적(deterministic)이다.

그 다음 `severity` 내림차순 정렬 후 **최대 6장으로 truncate** — 정보 과부하 방지.

### 4.3 시간축 — Timeline / Recording / Replay

- `←` `→` : 모든 패널을 **동시에** 과거로 스크럽 (라이브 120샘플 = 1Hz에서 2분)
- `R` : 세션 전체를 `.swr` 파일로 녹화
- `--replay <file>` : 길이 제한 없이 스크럽
- `--record --keep 24h` : TUI 없이 헤드리스 녹화, 시간별 청크 로테이션 + 자동 프루닝

**`.swr` 포맷 v3 설계 (약 110배 압축)**
```
6바이트 헤더 (b"SWR\0" + u16 version)
  → zstd 압축 블록 시퀀스 (u32 길이 + 데이터)
      → 블록 내부: 길이 프리픽스 postcard EncodedTick 레코드
```
- **프로세스 identity(이름/cmd/유저/ppid/start_time)는 pid당 1회만 기록**
  (pid 재사용 대비로 "본 적 있는 pid"가 아니라 identity 자체를 비교)
- **변하는 값(ProcVolatile)만 매 틱 기록**
- **services 목록이 직전 틱과 동일하면 `None`으로 생략**
- 키프레임 불필요 (재생은 항상 바이트 0부터 순차 디코드)
- 손상된 꼬리 관용: `RecordingReader`는 첫 불량 레코드에서 **에러 대신 정지**
  → 크래시로 잘린 녹화도 그 앞까지는 재생됨

### 4.4 세 가지 뷰 모드

| 모드 | 명령 | 크기 | 컨셉 |
|---|---|---|---|
| Lite | `--lite` / `L` | 80x24 | "왜 뜨겁고 느린가" 하나만 답함. 키 6개, 색 4개 |
| Full | (기본) | 자유 | 12탭 정식 |
| Dense | `--dense` / `V` | 130x44 | 크롬 행 0개, 박스 6개로 전 서브시스템 |

**Dense 레이아웃**
```
rows  0-11  cpu            풀 높이 브라유 그래프 + 축 + vitals
rows 12-23  mem  | net     구성 + 히스토리 | 미러 down/up
rows 24-31  cores | disk   코어 그리드 | read/write 스파크라인
rows 32-43  procs          detail-in-place + 프로세스 테이블
```
- **네트워크만 "미러" 그래프** — down은 위, up은 아래 → 백업/복원이 형태로 구분됨
- **색이 정체성이 아니라 크기를 인코딩** — 축을 읽기 전에 심각도가 보임
- 초록→주황→빨강은 **높으면 진짜 나쁜 값**(온도/메모리압박/디스크포화)에만 사용
- `1`~`6`으로 박스 하나를 전체 화면 확대, `esc`로 복귀
- 100x37 미만이면 3박스 컴팩트 배치로 폴백

---

## 5. 설계 철학 — 코드로 검증한 사실

| 약속 | 검증 방법 | 결과 |
|---|---|---|
| 네트워크 통신 없음 | `grep reqwest\|http\|TcpStream\|telemetry\|api_key` | ✅ 시스템 통신 히트 0건 |
| sudo 사용 안 함 | README + 수집기 코드 | ✅ 권한 필요 값은 `--` 표시 + 한 줄 안내 |
| 읽기 전용 | kill/renice/unmount/restart 부재 | ✅ 없음 |
| 데몬/포트 없음 | `--record`는 평범한 포그라운드 프로세스 | ✅ 설치/리스닝 없음 |

**정직성 관련 디테일**
- 센서를 못 읽으면 0이 아니라 `--` 렌더링
- 그래프 축은 **실제로 보유한 히스토리**만 표기 (120칸 @1Hz = 2분 걸려 채워짐)
- 히스테리시스: 3샘플 발동 / 5샘플 해제 → 플래핑 방지
- 메모리 압박은 추정이 아니라 **커널 판정**(Linux PSI, macOS `kern.memorystatus_vm_pressure_level`)
- **ZFS ARC를 available로 계산** — 압박 시 반납되는 캐시라 used에 두면 거짓 경보
- 데모 GIF도 "빨리감기 없음, `--tick 250`으로 촬영"을 README에 명시
- `main.rs::reset_sigpipe()` — `| head`, `| jq` 파이프에서 패닉 대신 조용히 종료

**`SECURITY.md` 스코프 1번**: *"읽기 전용 약속을 어기는 코드 경로는 그 자체로 보안 버그"*

---

## 6. 설치 및 사용법

### 설치
```bash
brew install syswatch                 # macOS / Linux
nix-shell -p syswatch                 # NixOS / Nix
paru -S syswatch                      # Arch (AUR)
cargo install syswatch                # Rust 있으면 어디서나

# 소스 빌드 (이 포크)
git clone https://github.com/bmshin94/syswatch.git
cd syswatch && cargo build --release && ./target/release/syswatch
```

### 대화형 실행
```bash
syswatch                       # 12탭, 기본 1Hz
syswatch --dense               # 전 서브시스템 한 화면
syswatch --lite                # 한 화면 요약
syswatch --tick 500            # 2Hz
syswatch --tab procs           # 특정 탭으로 바로 진입
syswatch --replay session.swr  # 녹화본 스크럽
syswatch --record --keep 24h   # 무인 녹화 (TUI 없음)
```

### 단축키
```
1..9      Overview/CPU/Mem/Disks/FS/Procs/GPU/Power/Services
0 - +     Net / Timeline / Insights
Tab       탭 순환                 up/down  행 선택
s         정렬 순환               / 또는 f  테이블 필터
left/right 세션 스크럽            Home/End  가장 오래된 샘플 / 라이브
p 일시정지  g 그래프 스타일         t 테마 순환   , 설정
S 스냅샷    R 세션 녹화            V 뷰 순환     L Lite로 점프
? 도움말    q / Ctrl-C 종료
```

### 비대화형 (스크립트/자동화)
```bash
syswatch snapshot --json               # 한 샘플, 전체 구조
syswatch insights --since 30s --json   # 지정 시간 관찰 후 발동한 카드
syswatch why                           # 같은 내용을 평문 진단문으로
syswatch diff before.swr after.swr     # 두 녹화본의 마지막 스냅샷 비교
syswatch diff session.swr              # 한 녹화본의 처음 vs 끝
```
> `--json`이 출력하는 `Snapshot` / `Insight` 타입은 **TUI가 렌더링하는 바로 그 타입**이다.
> 리포트 전용 스키마가 따로 없어 드리프트가 구조적으로 불가능하다.

> ⚠️ `insights` / `why`는 백그라운드 수집기가 없기 때문에 **`--since` 만큼 블로킹**된다(기본 30초).

### 설정 파일
`~/.config/syswatch/config.toml` — theme / graph_style / graph_fade / view /
default_tab / tick_ms(100~5000 클램프). 파싱 실패 시 기본값으로 fail-soft.

---

## 7. 정체성 — 플러그인? 스킬? MCP?

**전부 아니다. 독립 실행 CLI 바이너리다.**

| 의심 | 검증 | 결과 |
|---|---|---|
| MCP 서버 | Cargo.toml에 MCP SDK, stdio JSON-RPC 핸들러 | ❌ 없음 |
| Claude 플러그인/스킬 | `.claude/`, `plugin.json`, `skill.md` | ❌ 없음 |
| 라이브러리 크레이트 | `[lib]` 섹션 | ❌ 없음 (`[[bin]]`만 존재) |
| 데몬/서비스 | 네트워크 리스닝, 설치 스크립트 | ❌ 없음 |

**단, MCP로 감싸기는 매우 쉽다** — `snapshot --json` / `insights --json`이
이미 존재하므로 래퍼는 수백 줄 수준이다.

### API 토큰 / 계정 / 인터넷
**전부 불필요.** 완전 오프라인 동작. 텔레메트리 없음. sudo 없음.
→ 에어갭(망분리) 환경에서도 그대로 사용 가능.

---

## 8. GitHub에서 주목받는 이유 (코드에서 읽어낸 요인)

> ⚠️ 실제 스타 수는 이 세션의 저장소 접근 범위(`bmshin94/syswatch`) 밖이라 확인하지 않았다.
> 아래는 숫자가 아니라 **관찰 가능한 인기 요인**이다.

1. **명확한 포지셔닝** — "htop shows *what's running*, SysWatch shows *what's happening*"
2. **README 최상단에 GIF 3개** (총 약 9MB) + `.tape` VHS 스크립트까지 커밋 → 재현 가능한 데모
3. **Rust + TUI** — HN / r/rust / r/commandline 에서 강한 조합
4. **패밀리 전략** — netwatch / diskwatch와 같은 크롬·팔레트·`V` 키 → 머슬 메모리 공유
5. **배포 마찰 0** — brew / nix / AUR / cargo / 프리빌트 바이너리, **armv5te 구형 NAS 빌드까지**
6. **Anti-goals 섹션** — "안 하는 것"을 명시하는 태도가 엔지니어 신뢰를 얻음
7. **코드 품질** — 테스트 417개, 3-OS CI(fmt + clippy + build + test), Nix flake CI 검증
8. **보안 스토리** — no sudo / no network / read-only 3종 세트 → 기업 도입 장벽이 낮음

---

## 9. 로컬 AI 에이전트 구축에 주는 가치

### 에이전트가 얻는 것 3가지
1. **깨끗한 구조화 데이터** — `snapshot --json` 하나로 전 서브시스템
2. **이미 추론된 결론** — raw `ps aux`를 LLM에 던지면 수만 토큰이지만
   `why` 출력은 수백 토큰이고 **범인까지 지목**돼 있다 (토큰·품질 양쪽 승리)
3. **시간축** — `--record` + `diff`로 "배포 전후 무엇이 달라졌나"를 답할 수 있음

### 권장 아키텍처
```
Claude / 로컬 LLM
      | MCP (stdio)
syswatch-mcp  (TypeScript 래퍼)
      |  get_snapshot / diagnose(since) / diff(a,b) / list_recordings
      | exec + JSON parse
syswatch 바이너리 (읽기 전용, 오프라인)
```
- syswatch가 **읽기 전용**이라 에이전트에게 노출해도 파괴적 행위가 구조적으로 불가능
- 네트워크를 쓰지 않아 시스템 정보가 외부로 유출되지 않음
- MIT 라이선스라 상업 제품 내장 가능

### 주의점
- `insights` / `why`는 `--since` 동안 블로킹 → **비동기 처리 필수**
- 메모리 누수 탐지는 2분 이상 관찰 필요 → 상시 `--record` 권장
- macOS / Linux 전용

---

## 10. React / PHP 로 만들 수 있는가

### 계층별 현실 진단

| 계층 | 현재(Rust) | React | PHP |
|---|---|---|---|
| 커널 데이터 수집 | sysinfo / IOKit / sysfs | ❌ 브라우저 샌드박스로 **불가능** | ⚠️ `shell_exec` 뿐, 매 틱 fork |
| 이상 탐지 로직 | 순수 함수 1,671줄 | ✅ 포팅 쉬움 | ✅ 포팅 쉬움 |
| 차트 / UI | ratatui | ✅✅ **최적 영역** | ⚠️ 가능 |
| 바이너리 포맷 | postcard + zstd | ⚠️ WASM 필요 | ⚠️ 어려움 |
| 1Hz 상시 폴링 | 네이티브, 저비용 | ❌ Electron 오버헤드 | ❌ 요청-응답 모델과 불일치 |

**결론: 재작성은 비추. 하이브리드가 정답.**

### 플랜 A — 하이브리드 (권장)
```
React 대시보드  <-- WebSocket/SSE --  Node 브리지  -- exec -->  syswatch 바이너리
 (Recharts/uPlot,                    (1초 주기로
  타임라인 스크러버,                   snapshot --json)
  Insight 카드 UI)
```
수집은 검증된 Rust가, UI는 React가 담당. 개발 기간이 월 단위 → 주 단위로 줄어든다.

### 플랜 B — PHP는 "수집"이 아니라 "집계"로
```
서버 N대: cron으로 syswatch snapshot --json | curl POST
   -> PHP(Laravel) 수집 API + MySQL/TimescaleDB
   -> React 플릿 대시보드
```

### 플랜 C — Insights 로직만 TypeScript 포팅
`insights/mod.rs`가 순수 함수라 `(History, Snapshot) -> Insight | null` 형태로
그대로 옮길 수 있다 → 브라우저에서 오프라인 재분석 / 리플레이어 구현 가능.

---

## 11. 수익화 아이디어

| # | 아이디어 | 난이도 | 기간 | 초기비용 | 잠재력 | 스택 적합성 |
|---|---|---|---|---|---|---|
| 1 | **syswatch-mcp** (AI 에이전트용 MCP 서버) | 낮음 | 1~2주 | $0 | ★★★★ | TypeScript ✅ |
| 2 | **웹 리플레이어** (.swr 브라우저 뷰어) | 중간 | 4~6주 | 낮음 | ★★★ | React ✅ |
| 3 | Fleet SaaS (멀티호스트 집계) | 높음 | 3~6개월 | 높음 | ★★★★★ | React + PHP ✅ |
| 4 | Windows 포트 | 중간 | 6~10주 | $0 | ★★★ | Rust 필요 |
| 5 | NAS / 홈랩 어플라이언스 | 중간 | 4~8주 | 낮음 | ★★ | React ✅ |
| 6 | **교육 콘텐츠** (강의/전자책/유튜브) | 낮음 | 2~4주 | $0 | ★★★ | 글·영상 |
| 7 | 커스텀 휴리스틱 개발 (B2B) | 중간 | 건별 | $0 | ★★★★ | Rust 필요 |

### 1. syswatch-mcp
- 노출 툴: `get_snapshot()` / `diagnose(since)` / `why()` / `diff(a,b)` / `list_recordings()`
- 차별점: 경쟁 툴이 raw 출력으로 토큰을 태울 때, 이쪽은 **이미 요약된 결론**을 준다
- 모델: Free(OSS) → Pro $5~9/월(자동 녹화·장기 히스토리·커스텀 휴리스틱·알림) → Team $29/월~

### 2. 웹 리플레이어
- `.swr` 드래그앤드롭 → 타임라인 스크러빙 → `?t=03:14:22` 링크 공유
- 파싱은 Rust→WASM 재사용 또는 TS 포팅, 차트는 uPlot/visx
- 셀링포인트: **100% 클라이언트 사이드 파싱 = 프라이버시**
- 모델: Free(로컬 파싱) / Pro $12월(클라우드 저장·영구 링크·주석·팀) / Enterprise 셀프호스트

### 3. Fleet SaaS
- 프로젝트의 명시적 anti-goal("Not multi-host")을 정면으로 채우는 포지션
- 타겟: 서버 5~50대 소규모 팀 (Datadog은 비싸고 Prometheus는 세팅 부담이 큰 구간)
- 가격: 호스트당 $2~4/월
- **리스크**: 원작자가 "멀티호스트는 NetWatch 웹 대시보드 담당"으로 선을 그어둠 → 경쟁 가능성

### 4. Windows 포트
- 현재 Windows 전용 코드 0줄, 릴리스 타겟에도 없음 = 빈 시장
- `sysinfo`가 Windows를 지원하므로 CPU/Mem/Proc/Disk/Net은 상당 부분 커버
- GPU(DXGI/NVML), 전력, 서비스(SCM)는 중간 난이도, 온도/팬(WMI, 벤더별)은 난이도 높음
- 보너스: upstream PR로 보내면 메인 컨트리뷰터 등극 → 개인 브랜딩·컨설팅 유입

### 5. NAS / 홈랩 어플라이언스
- 근거: **armv5te 정적 빌드**(Iomega ix2-dl 등 Kirkwood NAS)를 일부러 지원 → 홈랩이 실사용자
- Synology / QNAP / Unraid / TrueNAS 패키지 + React 웹 UI + Docker 이미지
- `Cargo.toml`에 `smart` 피처가 이미 예약돼 있어 SMART 연동 확장 여지 있음

### 6. 교육 콘텐츠
- 교재로서의 강점: 실전 Rust 25k줄, 테스트 417개, `#[cfg]` 플랫폼 추상화,
  바이너리 포맷 설계(110배 압축), 성능 최적화 사례(sysinfo rayon 비활성화)
- 상품: 인프런/Udemy 강의, Gumroad 전자책, 유튜브 시리즈, 한글 번역 문서

### 7. 커스텀 휴리스틱 개발 (B2B)
- `insights`가 순수 함수 + `compute()` 한 줄 등록 구조 → 신규 탐지기 추가가 쉬움
- 예: 게임사(렌더 스레드 스톨), 핀테크(프로세스 레이턴시 이탈), ML팀(GPU 파편화/OOM 예측)
- 인프라 비용 0, 마진 최고

### 권장 로드맵
1. **1단계 (~4주)**: `syswatch-mcp` — 우리 스택으로 즉시 가능, 실패해도 손실은 시간뿐
2. **2단계 (1~3개월)**: React 웹 리플레이어 — 1단계 사용자에게 업셀, 데모 바이럴 가능
3. **3단계 (3~6개월)**: 유저가 모였으면 Fleet SaaS / B2B 문의가 왔으면 커스텀 휴리스틱
4. **병행**: 1~2단계 개발 과정을 콘텐츠화 → 브랜딩 + 마케팅 동시 달성

### 리스크 체크
| 리스크 | 대응 |
|---|---|
| 원작자의 NetWatch 웹 대시보드와 경쟁 | 겹치는 Fleet SaaS보다 MCP·리플레이어 우선 |
| TUI 모니터링은 니치 시장 | 개발자 대상이라 ARPU가 높음 |
| htop/btop/glances 등 무료 대안 | "기억 + 진단 + 범인 지목" 차별점 강조 |
| MCP 붐이 식을 가능성 | 1단계는 2주 투자라 리스크가 작음 |
| 라이선스 | MIT — 저작권 고지만 유지하면 문제 없음 |

---

## 12. 한 줄 결론

> **syswatch = 기억력이 있고, 범인을 지목하며, 모르면 모른다고 말하는,
> 읽기 전용 · 오프라인 · 무권한 터미널 시스템 진단 도구.**
>
> 우리에게는 **"AI 에이전트에게 먹일 고품질 시스템 진단 데이터 소스"**로서
> 가장 큰 가치가 있고, 가장 빠른 수익화 경로는 **MCP 래퍼 + 콘텐츠 병행**이다.
