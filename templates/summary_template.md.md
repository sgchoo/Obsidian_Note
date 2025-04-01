<%*const GEMINI_API_KEY="AIzaSyALDzRiwn0Wf4adbYnUNcUBskrqwyW3gr0"%>
 
<%*
// 요약 프롬프트
const summary_prompt = `당신은 **[문서 유형 (예: 학술 논문, 기술 보고서, 뉴스 기사)] 요약 전문가**로서, 제공된 텍스트를 **[요약 목적 (예: 핵심 정보 파악, 의사 결정 지원)]**을 위해 철저히 요약하고 핵심 내용과 중심 주제를 추출하는 역할을 수행합니다. 요약은 **[대상 독자 (예: 해당 분야 연구자, 일반 독자)]**가 원문을 읽지 않고도 텍스트의 내용을 명확하게 이해할 수 있도록 **[요약 유형 (예: 정보 요약, 발췌 요약)]** 형태로 작성되어야 합니다.
 
핵심 아이디어를 효과적으로 전달하는 데 집중하여 **[요약 길이 (예: 200단어 내외, 원본의 20%)]**로 명확하고 간결하게 작성해 주세요. 요약에는 **[핵심 정보 기준 (예: 주요 연구 결과, 핵심 주장과 근거)]**을 포함해야 하며, **[포함해야 할 특정 정보 (예: 연구 방법론, 주요 통계)]**는 반드시 포함하고 **[제외해야 할 특정 정보 (예: 개인적인 의견, 부가 설명)]**는 제외해야 합니다.
 
요약 결과물은 **[품질 기준 (예: 정확성, 완결성, 논리적 일관성, 높은 가독성)]**을 충족해야 합니다. 한국어로 답변하며 제목은 포함하지 마세요.

이 요약은 다음과 같은 구조로 작성해주세요:

1. 첫 문단: 중요한 핵심 내용 요약 (2-3문장)
2. 두 번째 문단: 중요 포인트 (핵심 주장과 근거)
3. 세 번째 문단: 결론 또는 시사점

각 문단 사이에는 적절한 줄바꿈을 포함해주세요.`;
%>
 
<%*
// 노트 내용 가져오기
const fileContent = tp.file.content;

// 현재 날짜 및 시간 가져오기
const now = new Date();
const formattedDate = now.toLocaleDateString('ko-KR', {
  year: 'numeric',
  month: 'long',
  day: 'numeric'
});
const formattedTime = now.toLocaleTimeString('ko-KR', {
  hour: '2-digit',
  minute: '2-digit'
});
%>
 
<%*
try {
    // API 요청
    const response = await tp.obsidian.requestUrl({
        method: "POST",
        url: "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=" + `${GEMINI_API_KEY}`,
        headers: {
            "Content-Type": "application/json"
        },
        body: JSON.stringify({
            contents: [
                {
                    role: "user",
                    parts: [
                        { text: summary_prompt },
                        { text: fileContent }
                    ]
                }
            ],
            generationConfig: {
                temperature: 0.2
            }
        })
    });
    
    // 응답 파싱
    let summary = "";
    if (response && response.json && response.json.candidates && 
        response.json.candidates[0] && response.json.candidates[0].content && 
        response.json.candidates[0].content.parts && response.json.candidates[0].content.parts[0]) {
        summary = response.json.candidates[0].content.parts[0].text;
    } else {
        summary = "API 응답을 파싱하는 데 문제가 발생했습니다.";
    }
    
    // 결과 서식화
    tR = `## 📝 요약
> [!summary]+ ${tp.file.title} 요약
> *${formattedDate} ${formattedTime}에 생성됨*
> 
> ${summary.split("\n").join("\n> ")}
> 
> ---
> *Gemini API로 자동 생성된 요약입니다.*`;

} catch (error) {
    // 오류 처리
    console.error("API 오류:", error);
    tR = `## ❌ 요약 생성 실패
> [!failure] 오류 발생
> 요약을 생성하는 중 문제가 발생했습니다.
> 
> 오류 메시지: ${error.message || "알 수 없는 오류"}`;
}
%>