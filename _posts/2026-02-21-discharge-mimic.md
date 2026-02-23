---
title: "MIMIC-IV 원본 데이터 vs LLM 생성 결과물 대조 분석"
author: Lee Yebin
date: 2026-02-22 14:58:35 +0900
categories: [PROJECTS, DILAB]
tags: [연구실, LLM]
pin: false
math: true
mermaid: true
# image:
#   path: 
#   lqip: 
#   alt:
---

## discharge-summaries-from-mimic 프로젝트 분석

원본 데이터와 모델의 결과물을 비교 분석하기에 앞서, 해당 [Github 프로젝트](https://github.com/isobelweinberg/discharge-summaries-from-mimic/tree/main)의 전반적인 구조와 핵심 내용을 정리했다.

### 1. 프로젝트 개요

이 프로젝트는 MIMIC-IV 데모 데이터셋의 전자건강기록(EHR) 데이터를 활용하여, LLM이 환자의 퇴원 요악지(Discharge Summary)를 자동으로 생성할 수 있는지 평가하는 것을 목표로 한다.

- **핵심 과제**: 입력 데이터에는 환자 목록, 입원 기록, 검사 결과, 처방 내역, 시술과 같은 정형 데이터만 존재하며, 서술형 임상 노트는 포함되어 있지 않다. 즉, 주요 사건 기록만을 바탕으로 LLM이 입원 기간 동안의 narrative를 얼마나 잘 재구성하는지 확인하는 것이다.
- **사용 모델 및 세팅**: Google Gemini 1.5 Pro (Vertex AI 환경 사용 -> 프라이버시 보호 목적)
  - Temperature 값을 1로 설정
  - ICD 진단 코드를 텍스트로 변환하지 않고 숫자 그대로 제공
- **프롬프트 엔지니어링 결과**: 저자는 짧은 프롬프트와 긴 프롬프트를 비교하는 실험을 진행했다. 모델은 짧은 프롬프트를 주었을 때 더 나은 퇴원 요약지를 생성했으며, 모델의 출력을 개선하기 위해 추가적인 지시사항을 많이 넣을수록 오히려 임상적인 서사 흐름이 악화되는 경향을 보였다.
  - Short Prompt: "제공된 데이터로 퇴원기록지를 써줘. 영국식 날짜 포맷을 써."
  - Long Prompt: "너는 의사야. 진단명, 주요 문제를 각각 새 줄에 적고, 검사 결과를 간결히 적어. 타임스탬프와 진단 코드는 빼고, 다음 소제목들(환자 ID, 나이 등)을 마크다운으로 굵게 써."
- **출력 형식**: Patient ID, Age, Admission/Discharge Date, Diagnosis, Hospital couse, Discharge medications 등 지정된 마크다운 헤딩에 맞추어 결과를 출력하도록 유도했다.
  - Patient ID: 환자 ID
  - Age: 환자의 나이
  - Admission/Discharge Date: 입원/퇴원 일자
  - Diagnosis: 진단명
  - Hospital course: 입원 기간 동안 환자에게 어떤 검사와 치료를 진행했으며, 그에 따라 상태가 어떻게 변화했는지(입원 중 경과)
  - Discharge medications: 퇴원 처방 약물
  

### 2. 주요 파일 및 역할

- `extract_files.py`: PhysioNet에서 다운로드한 압축 파일 형식의 MIMIC 데이터셋을 프로젝트 내 /data/hosp 및 /data/icu 폴더로 추출하고 준비하는 스크립트

- `data_to_database.py`: 추출된 CSV 등의 원시 데이터를 로컬 데이터베이스에 적재하여, 환자 및 입원 단위로 필요한 기록을 쉽게 쿼리(조회)할 수 있도록 만드는 역할

- `collate_data_over_subjects.py`: 여러 환자(Subject)들의 흩어진 진료, 검사, 처방 기록 데이터를 환자 ID 및 입원 건별로 병합하고 취합(Collation)하는 스크립트

- `find-headings.ipynb`: 퇴원 요약지에 필수적으로 들어가야 할 항목(Heading)들을 정의하고 구조를 탐색하기 위한 분석용 파일

- `generate-model-inputs.ipynb`: 취합된 정형 데이터를 바탕으로 LLM에 전달할 텍스트 형태의 '입력 프롬프트(사건 기록 리스트)'를 생성하는 과정

- `pass_to_model.py 및 pass_to_model.ipynb`: 생성된 입력 데이터를 바탕으로 실제 모델 API를 호출하여 퇴원 요약지를 생성하는 로직이다. 모델의 에러를 처리하거나, 여러 가지 프롬프트(짧은 것 vs 긴 것)를 테스트하는 코드가 포함되어 있다.

- `convert-existing-files-to-md.py`: 모델이 생성한 기존 텍스트 결과물들을 마크다운(.md) 파일 형식으로 일괄 변환하여 가독성을 높여주는 유틸리티 스크립트


### 3. 최종 결론

1. **요약력**: 구조화되지 않은 숫자 데이터만으로도 LLM은 그럴싸한 퇴원기록지 형태를 만들어냄
2. **프롬프트의 역설**: 지시사항이 긴 프롬프트보다 짧은 프롬프트를 썼을 때 임상적인 서사가(Narrative)가 훨씬 더 고품질로 나옴. (긴 프롬프트는 덜 중요한 검사 결과를 나열하는 등 요약의 본질을 흐림)
3. **한계**: 입원 중 투여한 약을 퇴원 약으로 잘못 분류하거나, 서사가 부자연스러운 Hallucination가 존재함. (저자 본인도 Clinical notes가 포하된 데이터로 하면 더 결과가 좋을 것 같다고 한계를 인정함)

## 원본 데이터와의 비교

Patient 10001217은 뇌종양(Brain Abscess)으로 인한 개두술(Craniotomy)을 받은 다발성 경화증(MS) 환자이다. 이 환자에 대한 원본 퇴원 기록지와 깃허브 프로젝트에서 생성된 요약본을 대조해보았다.

### 1. 원본데이터 vs 요약본

#### 원본 기록

> Chief Complaint: Left hand and face numbness, left hand weakness and clumsiness, fever, and headache.

(주호소 - 왼손 및 안면 마비, 왼손 위약감 및 둔함, 발열, 두통)

#### 요약본 (Short Prompt)

> ...presented to the Emergency Department... with altered mental status.

(의식변화(Altered mental status)로 응급실에 내원함)

#### 요약본 (Long prompt)

> ...She presented with headache.

(두통으로 입원함)

- 뇌 질환에서 왼손마비(국소 신경학적 결손)와 의식변화는 완전히 다른 응급 상황이다. 원본 데이터에는 명백히 왼손 마비가 적혀있지만, 의식 변화라는 가짜 증상을 Hallucination하거나 아주 지엽적인 두통만 언급되어 있다.


#### 원본 기록

> On ___, Mrs. ___ was taken to the OR for a right parietal craniotomy...
> 
(며칠 뒤, 환자는 수술실로 옮겨져 우측 두정엽 개두술을 받았다.)


#### 요약본 (Short Prompt)

> Initial workup revealed encephalopathy and a history of recent craniotomy.

(초기 검사에서 뇌병증과 최근 개두술을 받은 과거력이 확인됨)

- 환자의 시술 코드표에서 개두수 ㄹ코드를 발견했지만, 문맥이 없으니 해당 수술이 입원해서 받은 수술인지, 과거에 받고 온 수술인지를 구분하지 못함.


#### 원본 기록

> The final results on the abcess culture was streptococcus Milleri.

(농양 배양 검사 결과 최종 결과는 스트렙토코쿠스 밀레리였음)

#### 요약본 (Long prompt)

> There was growth of Fusobacterium nucleatum in an abscess sample...

(농양 샘플에서 푸소박테리움 뉴클레아툼이 배양됨)

- 푸소박테리움은 해당 환자의 두 번째 입원 기록에 등장하는 균이다. 첫 번째 입원과 두 번째 입원의 검사 결과를 제대로 분리하지 못하고 다른 입원의 균 이름을 현재 요약본에 끌어다 썼다.


### 2. 프롬프트 비교

해당 프로젝트의 저자는 짧은 프롬프트가 긴 프롬프트보다 임상 서사가 좋았다고 말했지만, 실제로 비교해 보ㅗㄴ 결과 둘 다 좋지 못한 결과였다.

#### Short Prompt

- 문장은 매끄럽게 잘 읽힘
- 하지만 자연스러운 문장을 만들기 위해 없는 사실을 적극적으로 지어냄. 


#### Long Prompt

- 프롬프트로 양식과 페르소나를 지정해주자 환각이 줄어듦
- 하지만 형식에 갇혀버려서 임상적 흐름을 버리고 쓸데없는 숫자에 집착함.
  - ("Her sodium level dropped to 123 mmol/L" -> 원본 기록의 핵심은 뇌농양과 마비 증상이지, 입원 중 잠깐 떨어졌다 회복된 나트륨 수치 123이 아닙니다.)