# CONTEXT.md — 강의 자료 QA 슬랙 봇

## 대상 문서 / 데이터
- 무엇을: 수강 중인 「최신인공지능」 주차·교시별 강의 슬라이드 (한국어, 코드·표 포함)
- 몇 건: PDF 10~12건 (약 300쪽) ※ 실제 수집 후 수정
- 형식: PDF (텍스트 추출 가능한 슬라이드 PDF만 사용)
- 저장 위치: `data/lectures/week{NN}_{session}_{title}.pdf`

## 분할·메타데이터 설계
- 분할 방식: **구조 기반(C) + 재귀적(B) 2단 구성**
  - 1단(구조 기반): 슬라이드 1쪽을 1개 청크로 취급. 슬라이드는 쪽 단위로 의미가 완결되므로 출처 표기(`page`)가 정확해짐
  - 2단(재귀적): 500자를 넘는 쪽만 `RecursiveCharacterTextSplitter`로 재분할
  - 이유: 고정 크기(A)는 단어 중간에서 끊겨 코드·용어가 훼손됨. Markdown 헤더가 없는 PDF라 `MarkdownHeaderTextSplitter`는 직접 적용 불가
- chunk_size / overlap: 500 / 50 (경험칙 출발점, 7주차 방식의 평가로 조정)
- separators: `["\n\n", "\n", "다. ", ". ", " ", ""]` (한국어 종결어미 추가)
- 심을 메타데이터:

| 필드 | 타입 | 용도 |
|---|---|---|
| source | str | 출처 표기 (파일명) |
| page | int | 출처 표기 (쪽 번호) |
| week | int | 필터 검색 / Self-Query ("10주차에서") |
| session | int | 필터 검색 (교시) |
| section_title | str | 출처 표기 (슬라이드 제목, 첫 줄에서 추출) |
| year | int | 필터 검색 (학기 구분) |
| chunk_id | int | 운영·디버깅 |
| splitter | str | 분할 설정 기록 (예: `page+recursive-500-50`) |

- 타입 원칙: `week`, `session`, `page`, `year`는 숫자형으로 통일 (범위 필터 대비)

## 처리 흐름
슬랙 질문(@봇)
→ 질문 임베딩 → 벡터 검색(Top-k=4, 필요 시 `week` 필터)
→ 근거 청크를 프롬프트에 주입, gemma3:4b가 답변 생성 (근거 없으면 "자료에서 찾지 못함" 응답)
→ 같은 스레드에 답변 + 출처(`[10주차 2교시, p.9]`) 표기

## 사용 기술 요건 (수업 필수 요건 대응)
- LangChain: 문서 로더, 분할기, 벡터 저장소 리트리버, 프롬프트·체인
- LangSmith: 검색 단계와 생성 단계를 구분해 추적 (프로젝트명 `lecture-qa-bot`)
- 입출력: slack_bolt, Socket Mode, 스레드 답변
- 처리 방식: RAG
- 토큰: `.env`에만 보관 (`SLACK_BOT_TOKEN`, `SLACK_APP_TOKEN`), 커밋 금지

## 하드웨어 (VRAM 8GB)
- 생성 모델 gemma3:4b + KV 캐시·오버헤드 + qwen3-embedding:0.6b ≈ 5.4GB (수업 자료의 추정치 기준, 실측으로 재확인 예정)

## 범위 밖 (안 하는 것)
- 슬라이드 내 이미지·도식·그림의 내용 해석 (텍스트만 사용)
- 자료 자동 수집·크롤링 (수동으로 PDF 배치)
- 대화 이력 기반 다중 턴 문맥 유지
- Re-ranking, Parent Document 등 11주차 고급 검색 전략의 필수 적용 (여유 시 선택 적용)
- 다른 수업 자료 및 외부 지식 질의 (최신인공지능 자료 범위 외 질문은 답변 거절)
- 웹 UI 제공 (입출력은 슬랙으로만)
