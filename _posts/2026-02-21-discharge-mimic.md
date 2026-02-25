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

### Patient 10000032

#### 진단명 및 진료과 (Diagnosis & Service)

- 원본 데이터
  - 진료과: `Service: MEDICINE` (내과)
  - 퇴원 진단명: `Discharge Diagnosis: Ascites from Portal HTN` (문맥고혈압으로 인한 복수)
- Short Prompt
  - 진료과: `Admitting Service: Transplant` (이식외과)
  - 진단명: `Discharge Diagnosis: OTHER DISORDERS OF THE LIVER` (간의 기타 질환)
    - 구체적인 복수(Ascites)라는 병명이 사라지고, 뭉뚱그려진 상위 카테고리 병명으로 대체됨
- Long Prompt
  - 진단명: `Diagnosis Other disorders of the liver` (간의 기타 질환)

#### 주호소 및 입원 중 경과 (HPI & Hospital Course)

이 환자의 핵심 치료 기록은 복수에 찬 물을 빼내는 시술(복수천자)과 이뇨제 조절이다.

- 원본 데이터
  > Chief Complaint: Worsening ABD distension and pain (주호소: 악화되는 복부 팽창과 통증)
  > 
  > Major Surgical or Invasive Procedure: Paracentesis (주요 시술: 복수천자)
  >
  >Discharge Instructions:
... did a paracentesis to remove 1.5L of fluid from your belly. We also placed you on you 40 mg of Lasix and 50 mg of Aldactone (배에서 1.5L의 체액을 제거하기 위해 복수천자를 시행함. 또한 이뇨제인 라식스 40mg과 알닥톤 50mg을 처방함.)

- Short Prompt
  > During her brief stay, the patient received intravenous fluids and medications... (짧은 입원 기간 동안 환자는 정맥 수액과 약물을 투여받았다.)
- Long Prompt
  > She was found to be hypokalaemic and received potassium chloride replacement. A urine dipstick was positive for blood but urine culture showed mixed growth consistent with contamination. (환자는 저칼륨혈증이 발견되어 염화칼륨 보충을 받았다. 소변 스틱 검사에서 혈액 양성이 나왔으나 소변 배양 검사에서는 오염과 일치하는 혼합 성장이 나타났다.)

    - 원본의 경과 기록(Brief Hospital Course)에 따르면 `UA negative, Urine culture blood culture negative`(소변검사 음성, 소변 및 혈액 배양검사 음성) 이다. 


#### 퇴원 처방 약물 (Discharge Medications) 

- 원본 데이터
  - 총 8개의 약물
  - `Albuterol, Truvada, Furosemide, Ipratropium, Nicotine Patch, Raltegravir, Spironolactone, Acetaminophen.`
- Short & Long Prompt
  - `Heparin 5000 units SC TID` (해파린 5000단위 피하주사 하루 3번)
    - 두 응답 모두 원본 퇴원 약물 목록에 없는 헤파린(Heparin, 항응고제)을 넣었다. 
    - 하지만, 롱 프롬프트는 원본 데이터에 있는 8개의 약물 이름을 추출하여 목록에 포함시켰다. 특히 숏 프롬프트는 약물의 투여 용량과 횟수등 세부 처방 지시사항까지 일치했다.


#### 퇴원 지시사항 (Discharge Instuctions)

- 원본 데이터
  > eat a low salt diet. You will follow up with Dr. ___ in liver clinic and from there have your colonoscopy and EGD scheduled. (저염식을 드세요. 간 클리닉의 ___ 의사와 후속 진료를 하고, 대장내시경과 위내시경 일정을 잡을 것입니다.)

- Short Prompt
  > She was advised to follow up with her primary care physician for continued management of her liver disorder. (간 질환의 지속적인 관리를 위해 주치의와 후속 진료를 하도록 권고받았다.)

- Long Prompt
  > The patient was discharged home. (환자는 자택으로 퇴원했다.)

### Patient 10001217

Patient 10001217은 뇌종양(Brain Abscess)으로 인한 개두술(Craniotomy)을 받은 다발성 경화증(MS) 환자이다.

#### 초기 내원 상태 및 병력 (HPI & Mental Status)

- 원본 데이터
  - 내원 사유: `headaches and left hand clumsiness` (두통과 왼손의 둔함), `Awake and alert` (정신은 깨어있고 명료함)
  - 수술 시점: 입원 후 MRI를 찍고 뇌농양이 발견되어 개두술(Craniotomy)을 진행함.
- Short Prompt
  - `presented to the Emergency Department via ambulance... with altered mental status.` (구급차를 타고 의식 변화 상태로 응급실에 내원함.)
  - `history of recent craniotomy` (최근 개두술 병력)
    - 이번에 입원해서 수술을 받은 건데, 입원하기 전부터 이미 개두술을 받은 병력이 있다고 요약함
- Long Prompt
  - `She presented with headache.` (두통을 호소함)
  - `Craniotomy except for trauma` (외상으로 인한 경우를 제외한 개두술)

#### 진단명 및 수치 (Diagnosis & Labs)

- 원본 데이터
  - 진단명: `Brain abscess` (뇌종양), 기저질환으로 인한 다발성 경화증(MS) 보유
  - 배양 검사: `streptococcus Milleri` (연쇄상구균 밀레리 나옴)
- Short Prompt
  - `streptococcus anginosus infection` 
  - 진단명: `encephalopathy` (뇌병증)
- Long Prompt
  - `Other issues during admission Encephalopathy Essential hypertension Hyponatraemia` (기타 문제: 본태성 고혈압, 저나트륨혈당증)
  - `Her sodium level dropped to 123 mmol/L` (나트륨 수치가 123으로 떨어짐) -> 원문 데이터에는 나트륨 수치에 관한 정보가 없음


#### 퇴원 처방 약물 (Discharge Medications) 

- 원본 데이터
  - `Heparin Flush (10 units/ml) 2 mL IV DAILY and PRN, line flush` -> 수액 라인(PICC)이 막히지 않게 소량으로 씻어내는 단순 관류용
- Long & Short Prompt
  - `Heparin 5000 units TID SC / Heparin 5000 units three times a day` (관류용 액체를 하루 3번 피하 주사하는 고용량 항응고제 치료)
    - 다만 실제 원본에 있었던 약들은 이름, 투여 용량, 복용 횟수까지 완벽하게 뽑아냄


## 분석 결론


### Short vs. Long Prompt

일반적으로 AI에게 지시사항을 길고 상세하게 줄수록 좋은 결과가 나올 것이라 기대하지만, 본 프로젝트의 정형 데이터 요약 태스크에서는 완전히 정반대의 현상이 나타났다.

- Short Prompt: 구조와 항목만 간단히 제시했을 때, 모델은 제공된 입력 데이터 안에서 약물 이름, 날짜 등을 비교적 무난하게 매핑(Mapping)했다. 비록 복수천자 같은 핵심 사건을 서술하지 못하고 단순 나열에 그쳤지만, 데이터에서 크게 벗어나는 내용을 쓰려는 경향은 덜했다.

- Long Prompt: 지시사항과 제약 조건이 늘어나자 모델 내부에서 어텐션(Attention) 분산이 심각하게 발생했다. 환자의 메인 질환(뇌농양, 문맥고혈압)을 보지 못하고, 경미한 랩 수치(칼륨 3.4) 하나에 과몰입하여 엉뚱한 병명을 메인으로 내세웠다. 심지어 원본에 없는 가짜 수치(나트륨 123)와 세균(푸조박테리움)을 창작해 내는 Hallucination을 보였다.