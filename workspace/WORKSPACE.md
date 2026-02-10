# Workspace Structure

OpenClaw workspace 폴더 구조 및 파일 관리 가이드

## 📁 폴더 구조

```
workspace/
├── AGENTS.md                    # Agent 동작 규칙 (핵심)
├── SOUL.md                      # Agent 성격/톤
├── USER.md                      # 사용자 정보
├── IDENTITY.md                  # Agent 정체성
├── TOOLS.md                     # 로컬 도구 설정
├── HEARTBEAT.md                 # 주기적 체크 작업
├── calendar-access-policy.md    # 캘린더 접근 권한 정책
├── calendar-usage-log.json      # 캘린더 사용 로그
│
├── config/                      # 설정 파일
│   └── mcporter.json
│
├── data/                        # 데이터 파일 📊
│   ├── README.md
│   ├── rss-feeds.json           # RSS/Twitter/YouTube 목록
│   ├── people-to-track.md       # 추적 대상 인물
│   └── gog-credentials.json     # Google 인증 정보
│
├── docs/                        # 문서 📚
│   ├── README.md
│   └── setup/                   # 설정 가이드
│       ├── youtube-api-*.md
│       ├── daily-tech-articles-setup.md
│       ├── github-trending-integration.md
│       └── ...
│
├── ideas/                       # 아이디어 💡
│   ├── interview-prep-checklist.md
│   ├── tech-reading-list.md
│   └── ...
│
├── knowledge-graph/             # 지식 그래프 🕸️
│   ├── README.md
│   ├── nodes.json
│   ├── edges.json
│   └── search.py
│
├── learning/                    # 학습 자료 📖
│   ├── README.md
│   └── databricks/
│       ├── databricks-learning-progress.md
│       ├── databricks-day1-lakehouse-architecture.md
│       └── databricks-day2-delta-lake-acid.md
│
├── memory/                      # 일일 메모리 🧠
│   ├── 2026-02-01.md
│   ├── 2026-01-31.md
│   └── ...
│
├── reports/                     # 보고서 📄
│   ├── README.md
│   ├── github-trending-report-2026-02-01.md
│   ├── job-posting-report-2026-02-01.md
│   ├── openclaw-ideas-explosion.md
│   └── ...
│
└── scripts/                     # 스크립트 ⚙️
    ├── get-youtube-channel-ids.sh
    └── track-twitter.js
```

## 📋 파일 관리 규칙

### Core 파일 (루트)
**절대 삭제 금지 - Agent 동작의 핵심**
- `AGENTS.md` - 모든 세션이 시작할 때 읽음
- `SOUL.md` - Agent 성격 정의
- `USER.md` - 사용자 컨텍스트
- `calendar-access-policy.md` - iMessage 보안 정책

### 폴더별 용도

**1. `data/`** - 자동화 작업의 데이터 소스
- Cron job이 참조하는 파일
- JSON 설정 파일
- **수정 시 주의**: cron job 영향 확인 필요

**2. `docs/setup/`** - 설정 가이드
- 한 번 설정 후 참고용
- 트러블슈팅 가이드
- 주기적 업데이트 불필요

**3. `learning/`** - 학습 자료
- 체계적 학습 관리
- 진행 상황 추적
- 카테고리별 하위 폴더

**4. `reports/`** - 보고서 아카이브
- 날짜별 보고서 (YYYY-MM-DD 형식)
- 일회성 리서치 결과
- 30일 이상 된 파일은 압축 권장

**5. `memory/`** - 일일 메모리
- 매일 자동 생성 (`YYYY-MM-DD.md`)
- 최근 7일치만 유지 권장
- 중요한 내용은 MEMORY.md로 이동

**6. `ideas/`** - 아이디어 스토리지
- 실행 전 아이디어
- 체크리스트
- 프로젝트 계획

**7. `knowledge-graph/`** - 지식 그래프 데이터
- 구조화된 지식 저장
- 검색 스크립트

**8. `scripts/`** - 재사용 가능한 스크립트
- Bash/Node.js 스크립트
- Cron job에서 호출
- 실행 권한 필요 (`chmod +x`)

## 🧹 정리 팁

### 주간 정리 (매주 금요일)
```bash
# 오래된 reports 압축
cd reports && tar -czf archive-$(date +%Y-%m).tar.gz *-report-*.md && rm *-report-*.md

# 오래된 memory 정리 (7일 이상)
cd memory && find . -name "*.md" -mtime +7 -delete
```

### 파일 이름 규칙
- **보고서**: `{주제}-report-YYYY-MM-DD.md`
- **설정 가이드**: `{서비스}-setup.md` 또는 `{서비스}-guide.md`
- **학습 자료**: `{주제}-day{N}-{제목}.md`
- **메모리**: `YYYY-MM-DD.md` (자동 생성)

### Git 관리
현재 workspace는 Git으로 관리되고 있습니다.
- `.gitignore`에 민감 정보 추가 권장:
  ```
  data/gog-credentials.json
  calendar-usage-log.json
  *.secret.*
  ```

## 🔧 유지보수

### 정기 작업
- **매일**: `memory/` 폴더 자동 생성 (cron)
- **주간**: 오래된 `reports/` 정리
- **월간**: `learning/` 진행 상황 리뷰

### 백업 권장
중요 파일:
- `AGENTS.md`, `SOUL.md`, `USER.md`
- `data/*.json`
- `calendar-access-policy.md`
- `learning/` 전체

---

**Last Updated:** 2026-02-01  
**Maintained by:** Atlas (OpenClaw Agent)
