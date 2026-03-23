# AI NurtureNote Backend

육아 일기를 저장하고, OpenAI 기반 분석 결과를 생성해 다시 조회할 수 있도록 구성한 FastAPI 백엔드 프로젝트입니다.

![Architecture Diagram](assets/architecture-diagram.png)

## Overview

사용자가 육아 일기를 작성하면 백엔드는 먼저 기록을 DB에 저장합니다. AI 분석은 저장 응답에 직접 묶지 않고 백그라운드에서 수행합니다. 분석 시에는 사용자 일기와 함께 OpenAI Vector Store에 미리 적재된 발달이론, 육아 관련 자료를 `file_search`로 검색해 참고하고, 필요 시 제한된 도메인에 한해 `web_search`를 보조적으로 사용합니다. 생성된 분석 결과는 다시 DB에 저장되며, 이후 프론트엔드에서 조회할 수 있습니다.

## Features

- FastAPI 기반 API 계층 구성
- 일기 저장, 목록 조회, 기간 분석 엔드포인트 구현
- SQLite/Postgres 기반 데이터 저장 계층 구현
- OpenAI API 호출 결과를 서비스 흐름에 연결
- 저장 요청과 AI 분석을 분리한 비동기 처리 구조 구성
- AI 응답 정규화 및 DB 저장 로직 연결
- 프론트엔드가 재조회 가능한 응답 구조 설계

## Core Flow

1. 프론트엔드가 `POST /entries`로 육아 일기를 전송합니다.
2. 백엔드는 일기 원문과 메타데이터를 DB에 저장합니다.
3. 저장 응답은 바로 반환하고, AI 분석은 백그라운드 태스크로 분리합니다.
4. 분석 로직은 OpenAI API를 호출하면서 `file_search`로 OpenAI Vector Store를 조회합니다.
5. 부족한 경우에만 허용된 공신력 도메인에 대해 `web_search`를 사용합니다.
6. 분석 결과를 표준 JSON 형식으로 정규화한 뒤 DB에 저장합니다.
7. 프론트엔드는 `GET /entries` 또는 `POST /analyze`로 저장된 결과를 조회합니다.

## Architecture Notes

- 앱 데이터 저장소: SQLite 또는 Postgres
- RAG 지식 저장소: OpenAI Vector Store
- 백엔드 프레임워크: FastAPI
- AI 호출 방식: OpenAI Responses API 우선, 실패 시 Assistants API 폴백
- 비동기 처리: FastAPI `BackgroundTasks`

즉 이 프로젝트에서 Postgres는 벡터 DB가 아니라 사용자 일기와 AI 분석 결과를 저장하는 애플리케이션 DB 역할을 합니다. RAG에 필요한 발달이론/논문/가이드 문서는 OpenAI Vector Store에 사전 적재되어 있고, 분석 시점에는 해당 스토어를 검색해 답변 생성에 활용합니다.

## Backend Design

- API 요청을 검증하고 DB에 저장하는 흐름 구현
- AI 분석 요청을 백엔드 서비스 계층에서 호출하도록 연결
- 분석 결과를 정규화해 DB에 저장하고 API 응답으로 노출
- 저장과 분석을 분리해 응답 지연을 줄이는 구조 설계
- 앱 데이터 저장소와 RAG 지식 저장소를 분리한 구조 유지

전체적으로 AI 기능을 백엔드 서비스 흐름에 연결하고, 저장과 조회가 가능한 형태로 운영하는 데 초점을 둔 구조입니다.

## Repository Map

### `app/main.py`

FastAPI 앱 생성, CORS 설정, 라우터 등록, 서버 시작 시 DB 초기화를 담당합니다.

### `app/routers/entries.py`

핵심 API 엔드포인트를 제공합니다.

- `POST /entries`: 일기 저장 후 단건 분석을 백그라운드로 큐잉
- `GET /entries`: 최근 일기와 분석 결과 조회
- `POST /analyze`: 최근 N일치 기록을 기반으로 종합 분석 수행

### `app/models.py`

Pydantic 기반 요청/응답 스키마와 내부 데이터 모델을 정의합니다.

- `EntryCreate`: 일기 생성 요청
- `EntryResponse`: 일기 조회 응답
- `AnalyzeRequest`: 기간 분석 요청
- `AnalyzeResponse`: 기간 분석 응답
- `EntryRecord`: 내부 DB 변환용 dataclass

### `app/db.py`

애플리케이션 DB 접근 계층입니다.

- SQLite 또는 Postgres 연결 생성
- `entries` 테이블 초기화
- 일기 저장/조회
- AI 분석 결과 `analysis_json` 저장
- DB row와 응답 모델 간 변환

### `app/analysis.py`

AI 분석 서비스 레이어입니다.

- OpenAI 클라이언트 초기화
- 프롬프트 생성
- Responses API 호출
- `file_search`를 통한 OpenAI Vector Store 검색
- 필요 시 `web_search` 병행
- 실패 시 Assistants API 폴백
- 표준 구조 반환

### `app/analysis_normalization.py`

LLM 응답을 서비스 내부 표준 스키마로 맞추는 후처리 계층입니다. 응답 키 이름이 달라져도 일관된 구조로 정리해 DB 저장 및 프론트 반환 형식을 안정적으로 유지합니다.

### `app/tasks.py`

저장 직후 AI 분석을 백그라운드에서 수행하는 비동기 작업을 담당합니다. 사용자가 저장 응답을 기다리는 시간을 줄이기 위한 계층입니다.

## Design Highlights

- 저장 API와 AI 분석을 분리해 응답 지연을 줄인 점
- 앱 DB와 RAG 지식 저장소를 분리한 점
- 외부 AI API 실패를 고려해 폴백 경로를 둔 점
- LLM 결과를 그대로 쓰지 않고 정규화한 점
- 분석 결과를 재조회 가능한 데이터로 저장한 점

## Tech Stack

- Python
- FastAPI
- Pydantic
- SQLite / Postgres
- OpenAI Responses API / Assistants API
- OpenAI Vector Store

## Run Locally

### 1. 백엔드 실행

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
```

API 문서:

```bash
http://127.0.0.1:8000/docs
```

### 2. 필수 환경 변수

```env
OPENAI_API_KEY=sk-...
VECTOR_STORE_ID=vs_xxx
```

선택 환경 변수:

```env
OPENAI_MODEL=gpt-5
OPENAI_ASSISTANT_ID=asst_xxx
DATABASE_URL=postgresql://user:pass@host:5432/dbname
CORS_ORIGINS=http://localhost:3000,http://127.0.0.1:3000
```

## API Summary

- `POST /entries`
  - 육아 일기 저장
  - 저장 직후 단건 분석을 백그라운드로 실행
- `GET /entries`
  - 최근 일기와 저장된 분석 결과 조회
- `POST /analyze`
  - 최근 N일 데이터를 기준으로 종합 분석 수행

## Notes

프롬프트 설계와 Vector Store에 적재할 문서 구성은 별도로 관리되며, 이 저장소는 FastAPI 백엔드와 데이터 처리 흐름 중심으로 구성되어 있습니다.
