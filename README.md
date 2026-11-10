# Daylog (실시간 일일 마이크로 브리핑 보드) — 개발자 웹 포트폴리오

> **불확실성이 높은 원격 근무자를 위해 데이터 출처와 시각을 투명하게 공개하고, 외부 API 장애 속에서도 마지막 정상값을 지켜내는 프론트엔드 복원력(Resilience) 중심의 일일 브리핑 보드 쇼케이스입니다.**

- **라이브 서비스:** [https://realtime-info-board-t04.vercel.app](https://realtime-info-board-t04.vercel.app)
- **장애 대응 검증 데모:** [https://realtime-info-board-t04.vercel.app/verification](https://realtime-info-board-t04.vercel.app/verification)
- **원본 코드베이스:** [https://github.com/stulss/realtime-info-board-t04](https://github.com/stulss/realtime-info-board-t04)

---

## 🏆 핵심 엔지니어링 지표
- **Lighthouse (모바일 프로덕션):** 성능 **98** / 접근성 **100** / 모범 사례 **100** / SEO **100**
- **총 차단 시간(TBT):** 2,130ms → **140ms** (93.4% 개선)
- **웹 접근성:** axe-core 기준 WCAG 2.1 AA 위반 **0건**
- **테스트 자동화:** 단위 116건 + Playwright E2E 24건 (총 140건 100% 통과)

---

## 📁 파일 구성
- `index.html`: 인터랙티브 웹 포트폴리오 쇼케이스 (라이브 장애 시뮬레이터, 스카이 히어로 씬 프리뷰어, WAI-ARIA 키보드 내비게이션, A4 PDF 인쇄 최적화)
- `PORTFOLIO.md`: 상세 기술서 (원격 근무자 페르소나, Human-in-the-Loop AI 워크플로우, C-P-A-D-I 대안 비교)
- `TECHNICAL_DEEP_DIVE.md`: 심층 기술 분석 (TBT 인과관계 분리, RFC 5861 Stale-if-error, SSR 상대시각 무한루프 방어, Map 최적화)
- `RESUME_SUMMARY.md`: 이력서 제출용 1페이지 핵심 요약본
- `INTERVIEW_QA.md`: 기술 면접 대비 Q&A 가이드 (면접관 예상 질문 10선)
- `assets/`: 2800x1800 고해상도 아키텍처 다이어그램 및 프로젝트 실측 스크린샷 12종
