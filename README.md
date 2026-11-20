# 강의노트 봇 (Lecture QA Bot)

슬랙에서 질문하면 수강 중인 「최신인공지능」 강의 슬라이드를 검색해, 근거(주차·교시·쪽 번호)와 함께 답해 주는 RAG 기반 슬랙 봇입니다.

## 1. 목적

시험 대비나 복습 중 "그 내용이 몇 주차 어느 슬라이드에 있었지?"를 찾느라 PDF 수십 개를 넘겨 보는 불편을 해결합니다. 강의 자료에서 근거를 검색해 답하고 출처를 함께 표시하므로, 답변을 원본 슬라이드에서 바로 확인할 수 있습니다.

## 2. 기능

- 슬랙 채널·DM에서 `@봇 질문`으로 강의 내용 질의응답
- 답변 끝에 출처 표기 (예: `[10주차 2교시, p.9 · 청크 크기와 오버랩]`)
- 질문에 주차가 포함되면 해당 주차로 검색 범위 제한 (메타데이터 필터)
- 자료에 근거가 없으면 추측하지 않고 "자료에서 찾지 못했습니다"로 응답
- 같은 스레드에 답변 게시
- LangSmith로 검색·생성 단계 추적

## 3. 사용 모델과 선택 근거

- 생성 모델: gemma3:4b (Ollama)
- 임베딩 모델: qwen3-embedding:0.6b (Ollama)
- 근거:
  - 로컬 실행: 모델이 모두 로컬에서 동작하므로 상용 API 호출 없이 상시 사용 가능 (오프라인·비용)
  - 한국어 지원: 강의 자료가 한국어이므로 한국어를 학습한 임베딩 모델 선택. 최종 확정 전 코사인 유사도 검증(예: "휴학"과 "학업 중단")으로 한국어 의미 포착을 확인 예정
  - VRAM: 8GB 이내 동작 (아래 계산)
- VRAM 계산: gemma3:4b 3.3GB + KV 캐시·오버헤드 1.5GB + qwen3-embedding:0.6b ≈ 5.4GB → 8GB 안에 들어감 ✅ (모델 크기는 Ollama 라이브러리 표기 기준 추정치이며, 실행 후 `nvidia-smi`로 실측하여 갱신)

## 4. 사용 스택

- LangChain: PyMuPDF 계열 PDF 로더, `RecursiveCharacterTextSplitter`, 벡터 저장소 리트리버, 프롬프트 템플릿, LCEL 체인
- LangSmith: 검색(retrieve) 단계와 답변 생성(generate) 단계를 별도 span으로 추적하고, 검색된 청크와 최종 답변을 대조
- 벡터 저장소: Chroma (로컬 영속 저장)
- 슬랙 연동: slack_bolt, Socket Mode
- LangGraph: 사용하지 않음

## 5. 실행 방법

```bash
# 1) 환경 준비
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# 2) 모델 내려받기
ollama pull gemma3:4b
ollama pull qwen3-embedding:0.6b

# 3) 환경 변수 (.env, 커밋 금지)
#   SLACK_BOT_TOKEN=xoxb-...
#   SLACK_APP_TOKEN=xapp-...
#   LANGSMITH_API_KEY=...
#   LANGSMITH_TRACING=true
#   LANGSMITH_PROJECT=lecture-qa-bot

# 4) 강의 PDF 배치
#   data/lectures/week10_2_분할_메타데이터.pdf 형식

# 5) 인덱스 생성 (적재 → 분할 → 임베딩 → 저장)
python build_index.py

# 6) 봇 실행
python slack_bot.py
```

슬랙에서 `@봇 재귀적 분할의 동작 순서가 뭐야?`처럼 질문하면 같은 스레드에 답변이 달립니다.

## 6. 프로젝트 구조 (예정)

```
.
├── data/lectures/        # 강의 PDF
├── build_index.py        # 적재 → 분할 → 메타데이터 → 임베딩 → 저장
├── rag_chain.py          # 검색 + 생성 체인
├── slack_bot.py          # slack_bolt 입출력
├── CONTEXT.md            # 요구사항 명세
└── README.md
```

## 7. 한계

- 슬라이드의 이미지·도식 내용은 인식하지 못하고 텍스트만 사용합니다.
- 4B급 소형 모델이므로 복합 추론 질문에서 답변 품질이 낮을 수 있습니다.
- 자료 범위 밖 질문에는 답하지 않습니다.
