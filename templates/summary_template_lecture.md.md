<%*const GEMINI_API_KEY="AIzaSyALDzRiwn0Wf4adbYnUNcUBskrqwyW3gr0"%>
 
<%*
// 강의 요약 프롬프트
const summary_prompt = `당신은 강의 내용 요약 전문가입니다. 제공된 강의 노트를 분석하여 핵심 개념, 중요 정의, 주요 아이디어를 중심으로 명확하고 간결하게 요약해주세요.
요약은 다음 구조로 작성해주세요:
1. 첫 문단: 강의의 핵심 주제와 목적 (2-3문장)
2. 두 번째 문단: 주요 개념과 이론 설명 (3-5문장)
3. 마지막 문단: 결론 또는 실용적 적용점 (1-2문장)
전체 요약은 200-300단어 내외로 작성하며, 강의에서 사용된 전문 용어와 핵심 개념을 정확히 포함해주세요. 학습자가 강의 내용을 빠르게 복습할 수 있도록 명확하고 체계적으로 작성해주세요.
한국어로 답변하며, 불필요한 장식이나 제목은 포함하지 마세요.`;
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
%>
 
<%*
try {
    // API 요청 - 작동하는 템플릿 구조로 수정
    const response = await tp.obsidian.requestUrl({
        method: "POST",
        url: "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=" + `${GEMINI_API_KEY}`,
        headers: {
            "Content-Type": "application/json"
        },
        body: JSON.stringify({
            contents: [
                {
                    role: "user",  // role 추가
                    parts: [
                        { text: summary_prompt },
                        { text: fileContent }
                    ]
                }
            ],
            generationConfig: {
                temperature: 0.2,
                maxOutputTokens: 800
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
    
    // 별도의 키워드 추출 API 호출도 동일하게 수정
    const keywords = await tp.obsidian.requestUrl({
        method: "POST",
        url: "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=" + GEMINI_API_KEY,
        headers: {
            "Content-Type": "application/json"
        },
        body: JSON.stringify({
            contents: [
                {
                    role: "user",  // role 추가
                    parts: [
                        { text: "다음 텍스트에서 5개의 핵심 키워드를 추출하여 쉼표로 구분된 목록으로 제공해주세요(키워드만 나열):" },
                        { text: fileContent }
                    ]
                }
            ],
            generationConfig: {
                temperature: 0.1,
                maxOutputTokens: 100
            }
        })
    });
    
    let keywordList = "";
    if (keywords && keywords.json && keywords.json.candidates && 
        keywords.json.candidates[0] && keywords.json.candidates[0].content && 
        keywords.json.candidates[0].content.parts && keywords.json.candidates[0].content.parts[0]) {
        keywordList = keywords.json.candidates[0].content.parts[0].text;
    }
    
    // 결과 서식화
    tR = `## 📚 강의 요약: ${tp.file.title}
> [!abstract]+ 강의 핵심 요약
> *${formattedDate}에 생성됨*
> 
> ${summary.split("\n").join("\n> ")}
> 
> ---
> 
> **핵심 키워드**: ${keywordList}
> 
> *AI로 자동 생성된 요약입니다.*
} catch (error) {
    // 오류 처리
    console.error("API 오류:", error);
    tR = `## ❌ 요약 생성 실패
> [!failure] 오류 발생
> 요약을 생성하는 중 문제가 발생했습니다.
> 
> 오류 메시지: ${error.message || "알 수 없는 오류"}
> 세부 정보: ${JSON.stringify(error)}`;
}
%>