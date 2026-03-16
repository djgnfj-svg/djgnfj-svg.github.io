---
title: LangChain 간단한 정리
date: 2026-03-16 22:17:14 +0900
categories: [AI]
tags: [LangChain, LLM, Python, RAG]
---

# LangChain이 뭔데

LLM 애플리케이션을 만들 때 반복되는 패턴들이 있다. 프롬프트 조립, API 호출, 응답 파싱, 외부 데이터 연결 같은 것들. LangChain은 이런 작업들을 체인(Chain)이라는 단위로 묶어서 조립할 수 있게 해주는 Python/JS 프레임워크다.

처음 봤을 때는 "그냥 OpenAI API 직접 호출하면 되는 거 아닌가?"라는 생각이 들었다. 간단한 챗봇 하나 만드는 데는 맞는 말이다. 그런데 프롬프트에 외부 문서를 넣고, 응답을 파싱해서 다음 액션을 결정하고, 그 결과를 다시 LLM에 넘기는 파이프라인을 짜야 할 때 — 직접 다 짜면 금방 스파게티가 된다.


# 핵심 개념 정리

## Chain

LangChain의 기본 단위. 입력 → 처리 → 출력을 하나의 흐름으로 엮는다.

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

prompt = ChatPromptTemplate.from_messages([
    ("system", "너는 기술 블로그 편집자야."),
    ("user", "{text}를 3줄로 요약해줘.")
])

model = ChatOpenAI(model="gpt-4o")

chain = prompt | model  # LCEL (LangChain Expression Language)
result = chain.invoke({"text": "긴 기술 문서 내용..."})
```

`|` 파이프 연산자로 체인을 연결하는 게 LCEL이다. Unix 파이프처럼 앞 단계의 출력이 다음 단계의 입력이 된다.

## Retriever + RAG

LLM이 모르는 정보를 외부에서 가져와서 프롬프트에 넣어주는 패턴이 RAG(Retrieval-Augmented Generation)다. LangChain에서는 이걸 Retriever로 추상화한다.

```python
from langchain_community.document_loaders import PyPDFLoader
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS
from langchain.text_splitter import RecursiveCharacterTextSplitter

# 1. 문서 로드 + 분할
loader = PyPDFLoader("manual.pdf")
docs = loader.load()
splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
chunks = splitter.split_documents(docs)

# 2. 벡터 저장소에 임베딩
vectorstore = FAISS.from_documents(chunks, OpenAIEmbeddings())
retriever = vectorstore.as_retriever()

# 3. RAG 체인 구성
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

rag_prompt = ChatPromptTemplate.from_template(
    "다음 문서를 참고해서 질문에 답해줘.\n\n{context}\n\n질문: {question}"
)

rag_chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | rag_prompt
    | model
    | StrOutputParser()
)

answer = rag_chain.invoke("설치 방법이 뭐야?")
```

PDF를 읽어서 벡터화하고, 질문이 들어오면 관련 청크를 찾아서 프롬프트에 넣는 전체 흐름을 10줄 남짓으로 엮을 수 있다.

## Tool / Agent

LLM이 "이 질문은 검색이 필요하겠다"고 판단하면 직접 도구를 호출하는 구조다.

```python
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI

@tool
def get_server_status(server_name: str) -> str:
    """서버 상태를 조회한다."""
    # 실제로는 모니터링 API 호출
    return f"{server_name}: CPU 45%, Memory 72%"

model = ChatOpenAI(model="gpt-4o").bind_tools([get_server_status])
response = model.invoke("web-prod-01 서버 상태 좀 알려줘")
```

`@tool` 데코레이터로 함수를 등록하면, LLM이 함수 시그니처와 docstring을 보고 언제 호출할지 스스로 결정한다. (MCP같은 느낌이라고 해야했다.)


# 실제로 쓸 만한 상황
근데 요즘은 claude 가 다해준다.... 정말 소프트웨어에 종말이 온걸까...

## 1. 사내 문서 검색 챗봇

Confluence, Notion, PDF 매뉴얼 같은 내부 문서를 벡터화해두고 RAG로 연결하면, "이거 어디 문서에 있었지?" 하는 질문에 바로 답하는 챗봇을 만들 수 있다. Document Loader가 다양한 포맷을 지원해서 연동이 편하다.


## 2. 데이터 분석 자동화

CSV나 DB 데이터를 읽어서 자연어로 질문하면 SQL을 생성하거나 pandas 코드를 실행해주는 파이프라인. Agent가 데이터를 조회하고 → 분석하고 → 요약까지 한 번에 처리한다.

## 3. 멀티스텝 워크플로우

"고객 문의 메일을 분류 → 긴급도 판단 → 담당 팀 라우팅 → 답변 초안 생성"처럼 여러 단계를 거치는 작업. 각 단계를 체인으로 만들고 연결하면 된다.


# 쓰면서 느낀 주의점

## 편한데... 뭔가 뭔가... 어정쩡

LangChain은 편하지만, 내부에서 어떤 API 호출이 일어나는지 감이 안 올 때가 있다.  그리고 어디를 어떻게 수정해야 더 좋아 질까라는 막연한 고민이 생겼다.
디버깅할 때 `langchain.debug = True`를 켜면 중간 과정이 다 찍힌다.


음 이전에 써봤을 때 솔직히 대답에 드라마틱한 변화는 없다. 체계적으로 쓰면 좋긴하겠지만 랭 그래프는 쓸만했다. 그건 다음포스트...

```python
import langchain
langchain.debug = True
```

## 단순한 건 직접 짜는 게 낫다

프롬프트 하나 보내고 응답 받는 수준이면 LangChain 없이 OpenAI SDK 직접 호출이 더 깔끔하다. LangChain의 가치는 체인이 2개 이상 연결될 때, 또는 외부 데이터 소스가 끼어들 때 나타난다.


# 정리

| 상황 | LangChain 필요? |
|------|----------------|
| 단순 챗봇 | X — API 직접 호출 |
| 프롬프트 + 외부 문서(RAG) | O |
| LLM이 도구를 호출해야 할 때 | O |
| 멀티스텝 파이프라인 | O |
| 프롬프트 템플릿만 필요 | △ — 직접 f-string도 충분 |

LangChain은 "LLM을 앱의 두뇌로 쓸 때, 두뇌와 팔다리(도구, 데이터, 외부 시스템)를 연결하는 배선" 같은 프레임워크다. 배선이 복잡해질수록 가치가 올라간다. 간단한 건 직접 짜고, 복잡해지는 순간 LangChain을 꺼내 들면 된다.
