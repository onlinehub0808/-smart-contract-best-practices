우리는 기여하는 분들을 환영합니다. 이슈를 열어주거나 풀리퀘스트를 보내주세요. 가능하다면, 섹션/토픽 별로 다르게 풀리퀘스트를 보내주세요.

조금 더 확장될 필요가 있는 아이디어들을 위해서 [열린 이슈들](https://github.com/ConsenSys/smart-contract-best-practices/issues)들을 많이 읽어주세요.

## 편집 및 변경 사항의 제출

1. 저장소를 포크해주세요
2. [docs 디렉토리](../../../../tree/master/docs) 내의 마크다운 파일들에 변경을 만들어 주세요
3. 풀리퀘스트를 제출해주세요

## 청중 수준

중급 이더리움 개발자를 대상으로 작성해주세요. 솔리디티 프로그래밍의 기본을 알고 있고, 많은 컨트랙트를 코딩한 경험이 있다고 가정해주세요.

## 스타일 가이드라인

### 일반적 가이드

- **명료하게 글을 써주세요**
  - 한 섹션 당 최대 3,4개의 단어를 사용해주세요 (중요할 경우엔 예외가 있을 수 있습니다)
  - 말하기보다는 직접 보여주세요(예제가 장황한 설명보다 더 많은 것을 말해줍니다)
  - 너무 많고 관련없는 것까지 요구하는 복잡한 예제보다는 단순하고 설명이 충분한 예제를 포함해주세요
- **가능하다면 원 문서에 대한 링크를 연결해주세요**
- **확실할 경우에 새로운 섹션을 추가해주세요**
- **가능한 코드라인은 80자 이하로 작성해주세요**
- insecure, bad, good 등으로 **코드를 적절하게 표시**
- Use the format of the [Airbnb Javascript Style guide](https://github.com/airbnb/javascript) as a starting point

### Recommendations Section

- Always favor a declarative tip starting with a verb for the section title
- Include good and bad examples, when possible
- Ensure each subsection has an anchor tag for future hyperlinking

### Attacks Section

- Provide an example - then point to a recommendation for the solution in the relevant section of the doc
- List first/most visible attack, where possible
- Ensure each subsection has an anchor tag for future hyperlinking
- Mark vulnerable pieces of code as `// INSECURE`
