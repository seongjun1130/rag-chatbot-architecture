# rag-chatbot-architecture

> **※ 프로젝트 보안 및 비공개 안내**
> 본 RAG 챗봇 프로젝트의 원본 소스 코드는 팀 정책 및 보안 규정상 비공개(Private)로 관리되고 있습니다. 
> 따라서 본 README 문서는 실제 코드를 대신하여, 제가 백엔드 엔지니어로서 핵심적으로 기여하고 고민했던 **아키텍처 설계, 동시성 제어(GC 스케줄러), 트러블슈팅 및 기술적 의사결정 과정**을 요약하여 제공합니다.

# 목차

1. [RAG(Retrieval-Augmented Generation) 개요](#1-ragretrieval-augmented-generation-개요)  
2. [AI 모듈 변경](#2-ai-모듈-변경)  
3. [Weaviate Client 및 멀티테넌시 구조](#3-weaviate-client-및-멀티테넌시-구조)  
4. [Chat Memory 전략](#4-chat-memory-전략)    
5. [데이터 삽입 전략](#5-데이터-삽입-전략)  
6. [데이터 검색 전략](#6-데이터-검색-전략)  
7. [데이터 삭제 및 자동화(GC) 전략](#7-데이터-삭제-및-자동화gc-전략)  
8. [핵심 API 명세 및 데이터 파이프라인](#8-핵심-api-명세-및-데이터-파이프라인)
9. [시스템 고도화 및 확장 전략](#9-시스템-고도화-및-확장-전략)


## 1. RAG(Retrieval-Augmented Generation) 개요

### 정의  
Retrieval-Augmented Generation(RAG)은 대규모 언어 모델(LLM)의 생성 능력과 외부 지식 베이스의 검색 기능을 결합해, 신뢰도 높고 사실에 기반한 응답을 만들어 내는 기술입니다.


### 동작 흐름  
1. **질의 분석**  
   - 사용자가 입력한 질문을 토큰화·전처리하여 핵심 키워드를 추출  
2. **정보 검색(Retrieval)**  
   - 벡터 데이터베이스(문서 임베딩 저장소)에서 질의 임베딩과 유사도가 높은 문서 청크를 조회  
3. **문맥 보강(Augmentation)**  
   - 검색된 청크를 모델 입력 템플릿에 삽입해, LLM이 외부 지식을 참고하도록 프롬프트 구성  
4. **응답 생성(Generation)**  
   - 보강된 프롬프트를 LLM에 전달하여 답변을 생성  
5. **후처리(Post-processing)**  
   - 생성된 텍스트를 정제·포맷팅하고, 필요 시 요약·필터링하여 최종 결과 반환  


### 핵심 구성 요소  
- **벡터 인코더(Embeddings)**  
  - 텍스트를 고차원 벡터로 변환하는 모델 (예: OpenAI `text-embedding-3-small`, `text-embedding-ada-002`)  
- **벡터 데이터베이스(Vector DB)**  
  - 벡터화된 문서 청크를 저장·검색 (예: Weaviate, Pinecone, Milvus)  
- **프롬프트 템플릿**  
  - 검색된 청크와 사용자 질문을 결합해 LLM에 전달하는 입력 형식  
- **LLM(대규모 언어 모델)**  
  - 보강된 프롬프트로부터 답변을 생성 (예: OpenAI GPT-4o, Claude)  
- **애플리케이션 로직**  
  - 업로드·파싱·청킹·검색·API 엔드포인트 등을 담당하는 서버 사이드 코드 (Node.js/TypeScript 등)


### 주요 장점  
- **정확도 향상**: 외부 자료 기반 응답 → 환각 감소  
- **최신성 보장**: 실시간 업데이트된 문서 활용 가능  
- **토큰 절감**: 필요한 청크만 포함해 비용 최적화  
- **확장성**: 다양한 도메인 문서 추가로 지식 확장
- **기밀 보안성 강화**: 중앙화된 벡터 DB 접근 제어로 민감 정보 유출 위험 최소화 


### 활용 사례  
- 고객 지원 챗봇, 학술 정보 검색, 엔터프라이즈 서포트, 콘텐츠 생성 보조 등  


### 출처
- [RAG 개념 정리](https://beanistory.tistory.com/64)  
- [RAG 이해 및 활용](https://jeongwoo.tistory.com/151)  
- [벡터 DB 기초](https://beanistory.tistory.com/65)  


## 2. AI 모듈 변경
### Ollama → OpenAI 전환 배경
#### 1. 로컬서버 부담 경감
   -  Ollama를 로컬에서 돌릴 경우, CPU/GPU 자원과 메모리 사용량이 크게 늘어나고, 모델 업데이트·스케일링을 직접 관리해야 하는 유지보수 측면이 증가함.
   -  OpenAI API 사용 시 인프라 관리, 운영비용 절감가능.
#### 2. 최신 모델 성능
  - OpenAI GPT-4o는 Ollama의 로컬 모델 대비 더 넓은 도메인에 걸쳐 일관되게 높은 언어 이해·생성 능력을 제공.
  - 복잡한 질의 및 추론작업에서의 응답품질 향상이 가능하다.
#### 3. 멀티모달, 대화 최적화
- 향후 지원할 이미지, 텍스트 등의 멀티 모달 및 멀티턴 대화 관리 최적화 측면에서 유리함.
#### 4. 버전 관리 및 호환성
- OpenAI API는 모델 버전(예: gpt-4o-v1 → gpt-4o-v2)별로 명시적인 릴리즈 노트를 제공해, 새로운 기능·패치 적용이 용이함.
- 이후 가격대별 모델 교환식 기능이 추가 됄 경우 구현난이도가 낮음.
### 적용 방법
```ts
async function createCollection(client: weaviate.WeaviateClient, name: string): Promise<void> {
  await client.collections.create({
    name: name.toLowerCase(),
    properties: [
      { name: 'content', dataType: 'text' },
      { name: 'source', dataType: 'text' },
      { name: 'page', dataType: 'int' },
      { name: 'file_id', dataType: 'int' },
      { name: 'order', dataType: 'int' },
    ],
    vectorizers: vectorizer.text2VecOpenAI({
      model: 'text-embedding-3-small',
    }),
    generative: generative.openAI({
      model: 'gpt-4o',
    }),
  });
  console.log(`[Weaviate] 컬렉션 생성됨: ${name}`);
}
```
- `WeaviateClient` 를 불러와서 `client.collections.create` 메소드를 통하여 컬렉션 단위의 vectorizer 와 generater 를 설정하면 해당 컬렉션 `모든객체` 에 대하여 설정이 `일괄적용` 됩니다.
- 만약 모델별 설정이 필요할 경우 컬렉션 설정을 모델별로 단위설정 가능하도록 조정하는 방향이 필요합니다.
- OpenApiKey의 경우 `docker-compose.yml` 및 `.env` 파일에서 설정이 가능합니다.
### 출처 및 예제
- [Weaviate Docker 설치 가이드](https://weaviate.io/developers/weaviate/installation/docker-compose)
- [컬렉션별 AI 모델 설정 문서](https://weaviate.io/developers/weaviate/manage-data/collections?utm_source=chatgpt.com)


## 3. Weaviate Client 및 멀티테넌시 구조

### Client 호출
```ts
import weaviate from 'weaviate-client';
import type { WeaviateClient } from 'weaviate-client';

let client: WeaviateClient | null = null;

export async function getWeaviateClient(): Promise<WeaviateClient> {
    if (client) return client;
    client = await weaviate.connectToLocal();
    return client;
}
```
```ts
import { getWeaviateClient } from '../library/Weaviate';
const client = await getWeaviateClient();
```
- `Weaviate Client` 를 싱글톤 형태로 불러와서 사용 할 수 있도록 구성 하였음.
- `getWeaviateClient()` 메소드를 통해 호출이 전역 재사용이 가능함.

### Weaviate Multi-Tenancy

#### 1. Multi-Collection (컬렉션별 분리)

- **특징**  
  - 테넌트별로 완전히 독립된 클래스(컬렉션)를 생성  
  - 각 컬렉션마다 고유한 스키마(properties·인덱스·vectorizer) 설정 가능  
  - 한 컬렉션 장애 시 다른 컬렉션에 영향 없음  

- **장점**  
  - 고객별 요구사항에 맞춘 스키마 튜닝 자유  
  - 컬렉션 단위로 독립 백업·복원·삭제 가능 → 운영 리스크 최소화  
  - 완전 격리를 통해 보안·데이터 무결성 강화  

- **단점**  
  - 컬렉션 수 증가에 비례해 모니터링·업그레이드·백업 스크립트 복잡도 상승  
  - 메모리·디스크 등 리소스 중복 소모  
  - 서로 다른 컬렉션을 가로지르는 통합 검색·집계 구현 시 성능 저하 및 개발 복잡성 증가  



#### 2. Single-Collection + Tenant 샤딩

- **특징**  
  - 하나의 클래스 내에 `tenant_id` 필드를 추가해 내부 샤드(파티션)로 분리  
  - 모든 테넌트가 동일한 스키마(properties·vectorizer)를 공유  
  - `autoTenantCreation` 설정으로 동적 샤드 생성·관리  

- **장점**  
  - 컬렉션 수 1개 유지 → 백업·모니터링·업그레이드 단순화  
  - 클러스터 노드 추가 시 자동 샤드 재분배 통한 수평 확장 지원  
  - 테넌트 간 리소스 공유로 메모리·디스크 활용 효율 극대화  

- **단점**  
  - 모든 테넌트가 동일 스키마만 사용 가능 → 고객별 커스터마이징 제약  
  - 한 샤드 장애 시 같은 컬렉션 내 모든 테넌트에 영향 발생 가능  
  - 잘못된 `tenant_id`로 인한 불필요 샤드 생성 주의   


- 현재는 Multi-Collection을 채택하였으나, 향후 고객사 증가 시 리소스 효율을 위해 Single-Collection 기반의 샤딩(Tenant Sharding) 도입을 고려.

### 출처 및 예제
- [Client 연결 및 사용 가이드](https://weaviate.io/developers/weaviate/client-libraries/typescript/typescript-v3?utm_source=chatgpt.com)
- [Multi-Tenancy 가이드](https://weaviate.io/developers/academy/py/multitenancy/overview)

## 4. Chat Memory 전략

### 개요
GPT 기반 OpenAI Chat Completions API는 **stateless** 구조로, 호출 간에 대화 상태를 자동으로 유지하지 않음.  
따라서 매번 `messages` 배열에 이전 대화(시스템·유저·어시스턴트 메시지 전체)를 포함시켜 보내야만 모델이 문맥을 이해할 수 있습니다.


### 해결책

#### 1. RDB/NoSQL에 질문·답변 로그 저장
- 사용자별로 MySQL, PostgreSQL, MongoDB, Redis 등에 질의·응답 히스토리를 보관  
- LLM 호출 시 최근 N턴(chat history)을 꺼내 `messages` 앞부분에 포함해 전송

#### 2. 주기적 대화 요약
- 긴 대화로 토큰 한계에 근접할 때마다(예: 매 5턴) LLM에 “지난 대화를 요약해 달라”고 요청  
- 요약된 텍스트만 장기 저장 후, 이후 호출 시 해당 요약만 포함 → 토큰 소모 절감

#### 3. LangChain Memory 라이브러리 활용
- **BufferMemory**: 대화 전체를 메모리 버퍼에 저장 → 다음 호출 시 동일 맥락 유지  
- **ConversationSummaryBufferMemory**: 토큰 한계 초과 시 LLM으로 요약 → 요약문만 장기 저장  
- **VectorStoreRetrieverMemory**: 중요 발언 임베딩 저장·유사도 검색으로 관련 컨텍스트 재사용

#### 4. Assistants API 활용
- OpenAI Assistants API는 “어시스턴트”를 독립 자원으로 생성·관리  
- 플랫폼 레벨에서 대화 이력·도구·파일을 자동 보관 → 매번 `messages` 전송 없이도 상태 유지


### 장단점

#### RDB/NoSQL 로그 저장
- **장점**: 구현이 단순하며 기존 인프라 활용 가능  
- **단점**: 별도 저장소 관리 부담, 조회 시 추가 지연 발생

#### 주기적 대화 요약
- **장점**: 토큰 사용량 절감, 장기 대화 지원  
- **단점**: 요약 요청 자체가 추가 API 호출 필요, 요약 품질에 따라 문맥 왜곡 위험

#### LangChain Memory
- **장점**: 다양한 전략(Buffer/Summary/Vector) 선택 가능  
- **단점**: 라이브러리 의존, 커스터마이징 한계(특정 사례에 맞춘 세부 로직 필요할 수 있음)

#### Assistants API
- **장점**: 대화 이력·도구·파일 관리를 OpenAI 플랫폼이 대신 처리 → 개발·운영 부담 최소화  
- **단점**: 아직 베타 기능, 요금·쿼터 정책이 모델 요율과 별도 적용될 수 있음, 자체 커스터마이징 유연도 제한

### 출처
- [LangChain Memory Integrations](https://js.langchain.com/docs/integrations/memory/)  
- [OpenAI Assistants API 개요](https://platform.openai.com/docs/assistants/overview)  
- [Velog: OpenAI Assistants API 대화 상태 반영 예제](https://velog.io/@sobit/OpenAI-Assistants-API-l-%EB%8C%80%ED%99%94-%EC%83%81%ED%8…-%EB%B0%98%EC%98%81-%EC%98%88%EC%A0%9C)  


## 5. 데이터 삽입 전략

### 데이터 삽입 방법
`collection.data.insertMany(insertObject)`는 TypeScript 클라이언트에서 **한 번에 여러 객체를 컬렉션에 배치 삽입**할 때 쓰는 메소드.
**단일** 객체 삽입으로는 `insert(...)` 메소드가 있음.

- `collection`  
  해당 클래스(컬렉션)를 가리키는 인스턴스  
- `insertMany(...)`  
  배열 형태의 데이터 객체(`insertObject: Array<{ [prop: string]: any }>`)를 인자로 받아 **동작**  
  내부적으로 REST API의 `/v1/batch/objects` 호출 → 여러 객체를 한 번에 Weaviate에 저장  
- `insert(...)`  
  단일 데이터 객체(`insertObject: { [prop: string]: any }`)를 인자로 받아 **동작**  
  내부적으로 REST API의 `/v1/objects` 엔드포인트에 POST 요청 → 하나의 객체를 Weaviate 컬렉션에 저장  

#### 출처

- [Weaviate REST API: Create Objects (`/v1/objects`)](https://weaviate.io/developers/weaviate/manage-data/create) 
- [Weaviate REST API: Batch Import (`/v1/batch/objects`)](https://weaviate.io/developers/weaviate/manage-data/import)
- [Weaviate GraphQL: Add Objects Mutation](https://weaviate.io/developers/weaviate/graphql#add-objects) 
- [Weaviate TypeScript Client v3: Batch Inserts (`insertMany`)](https://weaviate.io/developers/weaviate/client-libraries/typescript/typescript-v3#batch-inserts)


### 데이터 청킹 방식
#### 1. Fixed Size Chunking(문자 분할)
 - **특징** : 내용이나 구조에 관계없이 지정된 문자 수만큼의 단위로 텍스트를 나누는 가장 간단한 방법
  
 - **장점**  
   - 로직이 단순하고 빠르다.
   - 아주 작은단위로 검색 범위를 줄일수 있다.

- **단점**  
  - 문장이 중간에 잘려서 의미 단절이 생긴다
- **결과값 예시**  

```text
  "안녕하세요 여러분. 이 문서는 RAG 시스템을 설명하기 위해 작성된 예시 텍스",
  "트를 포함합니다. 청크 분할은 단순 문자 기준이므로 문장이 중간에 끊길 수 있습니",
  "다. 예를 들어 ‘…API 호출 시 messages 배열을 모두 전달해야 한다’ 와 같은 문장도"
```
#### 2. Recursive Character Split(재귀 문자 분할)
- **특징** : 먼저 지정된 최대 길이(예: 1,000자)로 자르고, 오버랩이 필요할시 마지막 부분 일부(예: 100자)를 중복 포함해 재귀적으로 분할  
- **장점**  
  - 청크 간 일부 중복으로 문맥 연결성 유지  
  - 잔여 텍스트 유실 최소화  
- **단점**  
  - 재귀 호출로 연산량 증가  
  - 오버랩·최대 길이 값 튜닝 필요  
- **결과값 예시**  
```text
  "안녕하세요 여러분. 이 문서는 RAG 시스템을 설명하기 위해 작성된 예시 텍스트입니다. ... ",
  "텍스트입니다. ... ",
  "…API 호출 시 messages 배열을 모두 전달해야 한다’ 와 같은 문장도 … "
  ```

#### 3. Document-Based Chunking(문단 단위 분할)
- **특징**  : 빈 줄(또는 특정 구분자)을 기준으로 문단 전체를 하나의 청크로 분할  
- **장점**  
  - 자연스러운 읽기 단위 유지  
  - 문맥 단절 없이 온전한 문단 전달  
- **단점**  
  - 문단 길이 편차로 너무 짧거나 긴 청크 발생  
  - 토큰 제한 초과 위험  
- **결과값 예시**  
  ```text
  "첫 번째 문단: 오늘은 프로젝트 회의를 진행했다. 주요 일정과 책임자를 확정했고, 기술 스택을 논의했다.",
  "두 번째 문단: PDF 업로드 기능을 구현했다. Multer 미들웨어로 확장자·용량 검증, 임시 폴더 정리 로직을 추가했다.",
  "세 번째 문단: 벡터화 테스트를 완료했다. 텍스트 추출 후 512자 단위로 분할해 Weaviate에 인덱싱했다."
#### 4. Semantic Split (의미 기반 분할)

- **특징**  : AI 모델 또는 의미 파서를 사용해 주제·개념 단위로 텍스트를 분할

- **장점**  
  - 각 청크가 명확한 주제를 담아 검색 정확도 상승  
  - 의미 단위로 모델이 문맥을 이해하기 용이

- **단점**  
  - 추가 LLM 호출 또는 복잡한 파싱 로직 필요  
  - 비용 및 처리 지연 증가

- **결과값 예시**  
  ```text
  "프로젝트 개요: 이 시스템은 PDF 문서를 업로드하고, 텍스트를 추출해 벡터 DB에 저장한다."
  "파일 업로드: 사용자가 multipart/form-data로 PDF를 전송하면, Multer로 검증 후 저장한다."
  "검색 흐름: 추출된 텍스트를 임베딩해 Weaviate에 인덱싱하고, nearText 쿼리로 유사 문서를 조회한다."
  ```
#### 5. Agentic Chunking (에이전트 기반 분할)

- **특징** : LLM에게 최적 청킹 지점과 크기를 물어보고, 그 결과대로 동적으로 분할

- **장점**  
  - 질의 의도에 맞춘 맞춤형 청킹 가능  
  - 고정 전략 대비 유연성·효율성 높음

- **단점**  
  - 분할 전용 LLM 호출로 비용 및 처리 지연이 가장 큼  
  - 에이전트 설계 및 튜닝이 복잡

- **결과값 예시**  
  ```text
  "도입부: 시스템 목적 및 배경 설명"
  "구현 세부: 파일 업로드·텍스트 추출·청킹 전략"
  "검색·예외 처리: 벡터화, API 에러 핸들링 로직"
  ```

#### 현재 프로젝트 구조
- `RecursiveCharacterTextSplitter` , `agenticChunkingByPage` 둘 다 구현이 완료되어있음.
- `processUploadedFile` 메소드 내부에서 스위칭 처리가 가능하게끔 조정.
```ts
    // ✅ 사용할 청킹 방식 선택 (manualChunking | agenticChunkingByPage)
    const chunks = await agenticChunkingByPage(pages);  // LLM 기반 Agentic 방식
    const chunks = await manualChunkingByPage(pages); // 문장 기반 탐색
```
#### 출처
- [데이터 청킹 방식 개요](https://normalstory.tistory.com/entry/RAG-Agentic-Chunking-ing)

### 파일확장자 제한 및 용량 제한
> - 현재 프로젝트는 외부 라이브러리인 `multer` 를 사용하여 `multipart/form-data` 타입으로 파일을 받아 처리하고있음.

```ts
// upload.ts

// 허용 확장자 목록 txt, png, jpg, csv 등 추가구현 예정
const allowedExtensions = ['.pdf'];

export const upload = multer({
    storage,
    limits: {
        fileSize: 50 * 1024 * 1024, // 용량 제한 (50MB)
    },
    fileFilter: (req: Express.Request,
        file: Express.Multer.File,
        cb: FileFilterCallback) => {
        const ext = path.extname(file.originalname).toLowerCase();
        if (!allowedExtensions.includes(ext)) {
            const err = new multer.MulterError('LIMIT_UNEXPECTED_FILE');
            return cb(err);
        }
        cb(null, true);
    }
});
```
- Middleware 기반 필터링 코드로 업로드된 파일을 라우터 전송전 확장자, 용량 제한 검사.
- `fileFilter` 와 `allowedExtensions` 확장자 배열을 통해 파일 확장자 및 용량을 검사 시행.
> - 추후 확장자 추가시 `allowedExtensions` 배열에 확장자명 추가.

### 삽입 이상 처리
#### 1. 개요
- 여러 테이블에 걸친 RDB 행(ROW) DDL 작업과 S3, Weaviate 내부의 고아 객체(삭제 대상이지만 남아 있는 데이터)를 추적·로그하고, 스케줄러를 통해 자동 삭제 처리
#### 2. 주요 시나리오

1. **사용자 요청**  
   - 파일 삽입·삭제, 컬렉션 삭제 등 각종 작업 발생  

2. **실행 중 예외 로깅**  
   - 단계별 예외 발생 시 대상 파일·컬렉션 정보를 별도 로깅 테이블에 기록  

3. **스케줄러 기반 삭제**  
   - 예약된 시각 또는 주기마다 로깅 테이블을 조회해 고아 객체 판별  
   - S3와 Weaviate, RDB의 잔여 리소스를 일괄 삭제   

#### 3. 대안별 접근 방식

1. **동기적 플래그 기반 삭제**  
   - 각 객체에 처리 상태 플래그 추가 → 실패 시 롤백  
   - **장점**: 구현 단순, 외부 리소스 불필요  
   - **단점**: 요청 처리 시 동기적 작업으로 응답 지연  

2. **비동기 큐 클린업**  
   - RabbitMQ/Kafka 등 메시징 큐에 삭제 작업 적재 → 백그라운드에서 처리  
   - **장점**: 사용자 경험에 영향 없이 빠른 응답  
   - **단점**: 추가 인프라·러닝커브 필요  

3. **배치 스케줄러 기반 삭제**  
   - 실패 기록을 배치 작업으로 주기적 삭제  
   - **장점**: 비동기 처리로 성능 저하 최소화  
   - **단점**: RDB·스케줄러 자원 부담 증가  

- 초기 구축 단계에서는 RDB와 스케줄러를 활용해 오버헤드를 최소화(3번)하였으며, 향후 대규모 트래픽 발생 시 Message Queue 혹은 kafka 를 활용한 비동기 큐 클린업(2번)으로 고도화 가능.

```ts

// processUploadedFile()...
catch (error) {
    if (s3Url) {
      try {
        await DeletedCandidate.create({
          s3Key: rawFileName,
          fileId: fileId,
          deleteType: 'file',
          deleted: false,
          errorLog: error instanceof Error ? error.stack : String(error),
          companyId: companyId,
        });
      } catch (gcError) {
        console.warn('❗GC 후보 기록 실패:', gcError);
      }
    }
```
> - 삽입 동작중 예외 발생시 `deleted_candidates` 테이블에 고아객체 로깅 및 정해진 시간마다 스케줄러에 의한 삭제 처리가 진행

## 6. 데이터 검색 전략

### 기존 로직의 문제점 및 변경 이유

#### 1. 파일 필터링 부재  
- **문제**: 모든 파일을 대상으로 검색 → 불필요한 청크 포함, 응답 노이즈 증가  
- **이유**: 사용자 요구에 맞는 세분화된 검색 지원 필요  

#### 2. 검색 결과 이상  
- **문제**: `.query.near.Text` → .`generate.nearText` 사용하는 메소드 → 파일필터링 로직으로 인해 파일을 `검색` 후 AI 가 파일을 `재검색` 하는 이상구조.
- **이유**: 파일을 필터링 후 원하는 파일데이터로 답변 구축

#### 3. 빈 결과 처리 미흡  
- **문제**: 청크가 0개일 때 빈 배열 반환 → 사용자 경험 저하  
- **이유**: 의미 있는 대체 동작(파일 요약 제공) 필요  

#### 4. LangChain QAChain 파일 필터링 오류  
- **문제**: LangChain의 `retriever` 옵셔널 필터가 제대로 작동하지 않아 원하는 파일에 한정된 검색 불가  
- **이유**: 파일 필터링을 통하여 원하는 파일을 선택 할 수 있는 검색 로직 필요


### `.query.nearText` vs `.generate.nearText`

- **`.query.nearText`**  
  - **목적**: 벡터 DB에서 주어진 쿼리 임베딩과 유사도가 높은 문서(청크)만 `검색`하고 반환  
  - **특징**  
    - 오직 검색된 청크를 리턴 → 이후 별도 LLM 호출로 가공·요약 필요  
    - 검색 결과(청크) 자체를 확인하거나, 외부 로직에서 2차 처리할 때 주로 사용  
```ts
collection.query.nearText([query], {
    limit: 8,
    returnProperties: ['content', 'source', 'page', 'file_id'],
    filters: filter,
    certainty: 0.68,      // 벡터 유사도 설정
    returnMetadata: ["certainty", "distance"],
  },
  );
```


- **`.generate.nearText`**  
  - **목적**: 검색된 청크를 내부에 연결된 LLM 모듈로 `즉시 생성(생성형 요약·답변)`까지 한 번에 처리  
  - **특징**  
    - 검색 → 프롬프트 보강 → LLM 호출 → 요약/답변 생성까지 통합 실행  
    - “검색된 청크+사용자 질문”을 곧바로 모델에 전달해 최종 결과(return)를 받음  

```ts
  // fileId 가 없을경우 전체 컬렉션 벡터검색, ID가 N개 이상일경우 OR 연산
  const filter =
    fileIds && fileIds.length > 0
      ? Filters.or(
        ...fileIds.map(id =>
          collection.filter.byProperty("file_id").equal(id)
        )
      )
      : undefined;

const ragRes = await collection.generate.nearText(
    query,
    {
      groupedTask:
        `
          • 문서 청크 중 "${query}"와 관련된 정보가 하나라도 있으면:
            구체적 수치와 페이지·파일명을 출처로 포함해, 최대 3문장 이내로 해당 정보를 요약하세요.

          • 만약 관련 정보가 전혀 없다면:
            “${query} 라는 정보를 찾을 수 없습니다.”라고만 답한 뒤,
            전체 문서를 페이지·파일명 출처 포함하여 최대 3문장으로 요약하세요.
        `
      ,
    },
    {
      limit: 8, groupBy: {
        property: 'file_id',           // file_id 값으로 그룹핑
        numberOfGroups: 5,             // 최대 5개의 서로 다른 file_id 그룹
        objectsPerGroup: 1,            // 각 그룹당 1개 청크만 가져옴
      }, filters: filter,              // 각 파일별 데이터 필터링

    },
  );
```
> 해당 `filter` 방식을 통하여 file_id 기준에 맞는 데이터청크만들 리트리벌 해오도록 구현하였다.


### Group Task vs Single Task

- **Group Task**  
  - **개념**: 여러 파일(또는 여러 청크)에서 검색된 정보를 **하나의 응답**으로 묶어 요약  
  - **용도**:  
    - N개의 파일에서 공통 주제나 핵심 메시지만 뽑아 단일 문단으로 전달  
    - 예: “선택한 3개 PDF에서 ‘보안 정책’과 연관된 부분을 모아 한 문단으로 요약”  

- **Single Task**  
  - **개념**: 각 파일(또는 각 청크)에 대해 **독립적인 응답**을 개별 생성  
  - **용도**:  
    - 파일별 상세 답변이나 요약이 필요할 때  
    - 예: “PDF A에는 이런 내용, PDF B에는 저런 내용”처럼 각각 별도의 결과를 반환  

#### 출처

- [Weaviate 생성적 검색: 결과 변환(Transform Result Sets)](https://weaviate.io/developers/weaviate/starter-guides/generative#transform-result-sets)  
- [TypeScript 클라이언트 `.generate.nearText` API 문서](https://weaviate.github.io/typescript-client/interfaces/Generate.html#nearText)  
- [OpenAI 생성 모듈 설정 가이드](https://weaviate.io/developers/weaviate/model-providers/openai/generative#configure-collection)  
- [Weaviate 생성적 검색 전체 가이드](https://weaviate.io/developers/weaviate/search/generative)  

## 7. 데이터 삭제 및 자동화(GC) 전략

### 데이터 삭제 방법

- **`deleteById(id)`**  
  단일 객체 ID를 인자로 받아 **동작**  
  내부적으로 REST API의 `/v1/objects/{id}` DELETE 요청 → 지정된 객체만 Weaviate 컬렉션에서 제거  

- **`deleteMany(filters)`**  
  필터 조건 객체를 인자로 받아 **동작**  
  내부적으로 REST API의 `/v1/batch/objects` DELETE 요청 → 필터에 매칭되는 여러 객체를 한 번에 삭제  

```ts
async function deleteFileOnlyGC(fileId: number, companyCode: string): Promise<void> {
  const collection = (await getWeaviateClient()).collections.get(companyCode);
  await collection.data.deleteMany(
    collection.filter.byProperty('file_id').equal(fileId),
  );
}

async function deleteCollectionOnlyGC(companyCode: string) {
  const client = await getWeaviateClient();
  await client.collections.delete(companyCode);
}
```

#### 출처

- [Weaviate REST API: Delete Objects](https://weaviate.io/developers/weaviate/manage-data/delete) 
- [Weaviate TypeScript Client v3: JS/TS Client Examples](https://weaviate.io/developers/weaviate/client-libraries/typescript/typescript-v3)   


### Soft Delete vs Hard Delete 개요

#### 정의
- **Soft Delete**  
  - 데이터를 실제로 삭제하지 않고, 삭제 플래그(예: `is_deleted` 또는 `scheduled_delete` 타임스탬프)를 설정하여 ‘논리적’으로만 숨김  
- **Hard Delete**  
  - 데이터를 물리적으로 완전 제거하여, 복구 불가능하게 삭제


#### 주요 차이점
| 구분        | Soft Delete                                               | Hard Delete                     |
| ----------- | --------------------------------------------------------- | ------------------------------- |
| 저장 방식   | 삭제 플래그 또는 예약 시각만 변경 → 레코드 유지           | 레코드 및 연관 데이터 완전 삭제 |
| 조회 처리   | 조회 시 `is_deleted = false` 또는 `scheduled_delete` 검사 | 단순 선택(삭제된 레코드 없음)   |
| 복구 가능성 | 삭제 플래그만 해제하면 복구 가능                          | 복구 불가능                     |
| 저장 공간   | 삭제된 레코드도 유지 → 스토리지 증가                      | 재빠른 공간 회수                |


#### 장단점

#### Soft Delete
- **장점**  
  - **복구 용이**: 실수로 삭제한 데이터 복원 가능  
  - **감사·로그**: 삭제 이력(누가 언제 무엇을 삭제했는지) 유지  
  - **소프트 롤백**: 삭제 예약 후 일정 시간 동안 복구 여유 제공  

- **단점**  
  - **스토리지 증가**: 삭제된 레코드도 계속 저장 → DB 크기 확대  
  - **쿼리 복잡도**: 모든 조회에 삭제 플래그 조건 추가 필요  
  - **성능 저하**: 인덱스·테이블 스캔 비용 증가 가능

#### Hard Delete
- **장점**  
  - **공간 절약**: 불필요한 데이터 제거 → 디스크·메모리 확보  
  - **쿼리 성능**: 삭제 플래그 검사 불필요 → 단순 조회  
  - **보안·준수**: 민감 정보 완전 삭제로 GDPR·보안 요구사항 충족

- **단점**  
  - **복구 불가**: 삭제 후 데이터 복구 불가능  
  - **감사 공백**: 삭제 이력 유지 어려워, 추적성 감소  
  - **트랜잭션 위험**: 삭제 과정 중 오류 발생 시 롤백 고려 필요

> 현재 프로젝트에선 삭제요청시 `soft delete` 후 스케쥴러가 정해진 시각 (파일 GC: 매시 정각 , 컬렉션 GC: 매일 00시 30분 00초) 마다 삭제 요청된 파일 및 컬렉션을 `hard delete` 하고있음.
> - 정해진 시각마다 `hard delete` 를 진행하기에 삭제되는 시간의 편차가 파일별로 존재함으로 `soft delete` 구현 필요시 삭제 예정시간 + 복구기능 API 구현시 더 완벽한 삭제 기법 구현예정.


### 삭제 요청 처리 흐름

1. **요청 수신**  
   - 사용자가 파일 또는 컬렉션 삭제 API 호출  
2. **유효성 검사 & 인증**  
   - 요청자 권한 확인  
   - 대상 파일/컬렉션 존재 여부 검증  
3. **삭제 예약 로깅**  
   - `scheduled_delete` 타임스탬프와 함께 로깅 테이블에 삽입  
   - 성공 여부에 관계없이 예약 레코드 생성  
4. **스케줄러에 의한 최종 삭제**  
   - 예약 시각 도래 시 소프트 딜리트에서 하드 딜리트로 전환  
   - `scheduled_delete` 필드 확인 후 대상 삭제  

### 1. 파일 단위 GC (`runFileGc`)  
- **스케줄**: 매시 정각 (`cron.schedule('0 0 * * * *')`)  
- **시나리오**:  
  1. `DeletedCandidate` 테이블에서 `deleteType='file'`, `deleted=false`, `retryCount<3` 항목 최대 50건 조회 (비관적 락)  
  2. 각 후보별로  
     - S3에서 오브젝트 삭제  
     - Weaviate에서 해당 `fileId` 벡터 삭제  
     - RDB에서 파일 메타 삭제  
     - 성공 시 `deleted=true`, `deletedAt` 기록  
     - 실패 시 `retryCount++`, 3회 재시도 후 자동 스킵 및 에러 로그 기록  

### 2. 컬렉션 단위 GC (`runCollectionGc`)  
- **스케줄**: 매일 00:30 (`cron.schedule('0 30 0 * * *')`)  
- **시나리오**:  
  1. `DeletedCandidate` 테이블에서 `deleteType='collection'`, `deleted=false`, `retryCount<3` 항목 최대 50건 조회 (비관적 락)  
  2. 각 후보별로 트랜잭션 시작  
     - S3에서 복수 키(쉼표 구분) 삭제  
     - Weaviate에서 컬렉션 자체 삭제  
     - RDB에서 회사·파일 메타 일괄 삭제  
     - 관련 모든 `DeletedCandidate` 레코드 `deleted=true`, `deletedAt` 기록  
  3. 실패 시 `retryCount++`, 3회 재시도 후 스킵 및 에러 로그 기록  



### 레이스 컨디션 & 락 전략

- **동시 삭제 시나리오**  
  A와 B가 동시에 같은 파일/컬렉션을 삭제 요청 → 이중 삭제 시도 또는 충돌 발생

### 락 전략 비교

| 구분               | 낙관적 락 (Optimistic Lock)              | 비관적 락 (Pessimistic Lock)                      |
| ------------------ | ---------------------------------------- | ------------------------------------------------- |
| **충돌 확인 시점** | 트랜잭션 커밋 직전                       | 데이터 수정 직전에 즉시                           |
| **동시성**         | 매우 높음 (락 대기 없음)                 | 낮음 (잠금 대기 발생)                             |
| **성공률**         | 커밋 시 충돌 시 재시도 필요              | 충돌 사전 방지, 재시도 불필요                     |
| **성능 영향**      | 읽기·수정 속도 빠름, 충돌 시 오버헤드    | 잠금 대기 시간으로 인해 지연 발생                 |
| **구현 복잡도**    | 충돌 처리 재시도 로직 필요 → 복잡도 상승 | 단순(잠금만 걸어두면 됨) → 상대적으로 구현이 간단 |
| **적용 예시**      | 배치 작업, 비동시성 작업                 | 동시성이 높은 파일·컬렉션 삭제                    |

---

### 적용 방침

- **파일·컬렉션 삭제**: 동시 요청이 빈번하고 데이터무결성이 중요한 만큼 **비관적 락** 채택  
> 성능최적화가 필요할시 `메시징 브로커(RabbitMQ, Kafka 등)` 도입 후 비동기삭제처리 혹은 `낙관적 락` 사용 예정 

```ts
return sequelize.transaction({ isolationLevel: Transaction.ISOLATION_LEVELS.READ_COMMITTED }, async t => {
        const rows = await DeletedCandidate.findAll({
            // 재시도횟수 3회 이하만 조회
            where: { deleted: false, deleteType, retryCount: { [Op.lt]: 4 }, },
            attributes: ['id', 's3Key', 'fileId', 'companyId', 'retryCount'],
            order: [['id', 'ASC']],
            limit: 50,
            lock: t.LOCK.UPDATE,
            skipLocked: true,
            transaction: t,
        });
});
```

### 출처

- Soft Delete vs Hard Delete  
  - [Soft Delete(논리 삭제) vs Hard Delete(물리 삭제) – resilient-923](https://resilient-923.tistory.com/419)
  - [Soft Delete(논리 삭제)와 Hard Delete(물리 삭제) – heyazoo1007](https://heyazoo1007.tistory.com/773)

- Optimistic Lock vs Pessimistic Lock  
  - [낙관적 락 VS 비관적 락 – gyeongsuuuu](https://gyeongsuuuu.tistory.com/68) 
  - [낙관적 락, 비관적 락 – 주독야독](https://hulrud.tistory.com/113)


## 8. 핵심 API 명세 및 데이터 파이프라인
- **`POST /weaviate/insert-pdf/:companyName`**  
  - **개요**: PDF 문서를 페이지 단위로 청킹·임베딩해 Weaviate에 저장  
  - **Query Parameters**:
    - `files` (multipart/form-data,**required**) : 삽입하려는 파일데이터
  - **시나리오**:   
- 1. **[클라이언트]**: PDF 파일 선택 후 `/weaviate/insert-pdf/:companyName`로 POST 요청  
  1. **[서버]**: `multer`가 `uploads/` 폴더에 임시 저장  
  2. **[서버]**: `pdfjsLib`로 페이지 단위 텍스트 추출  
  3. **[서버]**: LangChain `RecursiveCharacterTextSplitter` 로 청킹 수행  
  4. **[서버]**: 각 청크를 Weaviate에 임베딩하여 삽입  
  5. **[서버]**: 원본 PDF를 S3에 업로드  
  6. **[서버]**: 파일 메타정보 및 `companyName` 매핑 데이터, 요약정보를 RDB에 저장  
  7. **[서버]**: 로컬 임시 파일 삭제 (`fs.unlink`)  
  8. **[서버]**: 파일업로드 실패시 고아객체 추적 GC처리 로깅
  ---
- **`GET  /weaviate/companies/:companyId/search`**  
  - **개요**: 질의어 기반 벡터 검색 수행 후 관련 청크 리스트 반환  
  - **Query Parameters**: 
    - `query`(string, **required**) : 검색할 질의어
    - `fileIds`(string, optional) : 검색할 파일 ID 리스트 (ex : 1,2,3), 필드가 없을시 컬렉션내 모든 벡터데이터 검색 
  - **시나리오**:  
- 1. **[클라이언트]**: `companyId`, `fileIds`(단일 또는 쉼표 구분 다중), `query`(질의어)를 포함한 GET 요청  
  1. **[서버]**: `companyId`와 `fileIds`의 유효성 검사 (회사·파일 PK 존재 여부 확인)  
  2. **[서버]**: `fileIds` 미지정 → 모든 파일(default) 대상으로 검색  
  3. **[서버]**: Weaviate `nearText` + `file_id` 필터링으로 벡터 검색 수행  
  4. **[서버]**: 검색 결과(청크) 응답  
  5. **[서버]**: 청크가 없을 경우 → LLM 요약 반환  
     - **단일 파일**: 해당 파일의 요약 문자열  
     - **다중 파일**: 선택된 파일별 요약 문자열 리스트  
     - **전체 검색**: 컬렉션 내 모든 파일의 요약 리스트  
---


- **`DELETE /weaviate/companies/:companyId`**  
  - **개요**: 컬렉션(회사) 단위 소프트 딜리트 예약 및 최종 하드 딜리트  
  - **시나리오**:
  1. **[유저]**: 특정 `companyId`의 컬렉션 삭제 요청 (`DELETE /weaviate/companies/:companyId`)  
  2. **[서버]**: `CompanyRepo.findById(companyId)`로 회사 엔티티 조회  
  3. **[서버]**: 이미 삭제 예약(`scheduledDelete=true`)되어 있으면 400 에러 반환  
  4. **[서버]**: `UploadedFileService.getStoredNamesByCompanyId(companyId)`로 S3 키 목록 조회  
  5. **[서버]**: `DeletedCandidate.create({ deleteType: 'collection', s3Key: 목록, ... })`로 GC 후보 테이블에 기록  
  6. **[서버]**: 회사 엔티티 `scheduledDelete` 플래그를 `true`로 업데이트  
  7. **[서버]**: 성공 응답(204 No Content) 반환  

 
## 9. 시스템 고도화 및 확장 전략

### 1. 비동기 이벤트 기반 파이프라인 구축 (Event-Driven Architecture)
 - 현재: PDF 업로드 시 S3 저장, 텍스트 추출, 청킹, Vector DB 인덱싱이 동기적으로 처리되어 대용량 파일 처리 시 타임아웃 및 병목 발생 가능성 존재.
 - 고도화 방안: 클라이언트 업로드 요청 시 S3에 파일을 임시 저장 후 즉각 응답(202 Accepted) 처리. 이후 실제 청킹 및 인덱싱 작업은 Message Queue (RabbitMQ / Kafka)를 활용해 백그라운드 워커 노드에서 비동기로 분산 처리하도록 아키텍처 개선.

 ### 2. Soft Delete 기반 데이터 생명주기(Lifecycle) 관리
 - 현재: 삭제 요청 시 GC 후보 테이블에 등록 후, 스케줄러가 일괄 Hard Delete 처리.
 - 고도화 방안: 즉각적인 물리 삭제의 위험을 방지하기 위해 완전한 유예 기간(Grace Period)을 두는 Soft Delete 로직 도입. 유예 기간 내 사용자의 취소(Restore) API를 지원하고, 만료 시 스케줄러가 S3 및 Weaviate에서 최종 삭제하도록 안전한 데이터 아카이빙 로직 구축.

 ### 3. 분산 환경의 스케줄러 안정성 및 모니터링(Observability) 강화
 - 현재 : 매시 정각 및 자정에 동작하는 단일 노드 기반의 GC 스케줄러 구현.
 - 고도화 방안: 다중 서버 환경에서 스케줄러가 중복 실행되는 것을 막기 위해 Redis 기반의 분산 락(Distributed Lock) 도입. 또한, 실패한 GC 작업에 대해 지수 백오프(Exponential Backoff) 및 지터(Jitter) 알고리즘을 적용하여 재시도 리소스를 최적화하고, 최종 실패 건은 DLQ(Dead Letter Queue)로 격리 후 Slack/Email Webhook을 통해 실시간 에러 관제망 연동.
