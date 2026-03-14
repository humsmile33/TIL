### Context
* 목적: 디스코드(Discord)에서 메시지를 수신하면 n8n 워크플로우가 트리거 되어, 메시지를 TIL(오늘 배운 것) 형식의 Markdown 파일로 변환한 뒤 GitHub에 push(생성/갱신)하고 README.md에 인덱스를 추가한 후 완료 메시지를 디스코드로 발신하는 자동화 구현.
* 배경: n8n을 이용한 자동화는 각 노드의 입력(input)과 출력(output)을 정확히 이해하면 설계가 단순해진다. 본 워크플로우는 다음 역할을 수행한다.
  * Discord -> Webhook(또는 Discord Trigger)
  * 메시지 변환(Function/Agent) -> TIL Markdown
  * GitHub 노드(또는 HTTP Request) -> 파일 생성/업데이트
  * README.md 업데이트 -> 인덱스 추가
  * 완료 메시지 전송(Discord)

### Core
* 기대 노드 구성(논리 흐름 설명):
  * Webhook(Discord에서 수신) 또는 Discord Trigger 노드
  * Function 노드(메시지 파싱 및 TIL Markdown 생성)
  * GitHub 노드(파일 생성/업데이트)
  * Function 노드(README 내용 가져와 인덱스 추가)
  * GitHub 노드(README 업데이트)
  * Discord 노드(Webhook으로 완료 메시지 발신)

* Function 노드 예제 (TIL 생성용 JavaScript):
```javascript
// 입력: {{$json}} 에서 message, author, title 등 추출
const msg = $json["content"] || $json["message"] || '';
const author = $json["author"] || $json["username"] || 'unknown';
const rawTitle = $json["title"] || msg.split('\n')[0].slice(0,50) || 'til';

function slugify(s){
  return s.toString().trim()
    .toLowerCase()
    .replace(/[^\w\s-가-힣]/g, '') // 영문/숫자/한글/공백/하이픈 허용
    .replace(/\s+/g, '-')
    .replace(/-+/g,'-')
    .replace(/^[-]+|[-]+$/g,'');
}

const date = new Date();
const yyyy = date.getFullYear();
const mm = String(date.getMonth()+1).padStart(2,'0');
const dd = String(date.getDate()).padStart(2,'0');
const filename = `${yyyy}-${mm}-${dd}-${slugify(rawTitle)}.md`;

const tilContent = `### Context\n* Source: Discord message from ${author}\n\n### Core\n* 메시지 원문:\n\n\`` + msg.replace(/```/g,'`\`\`') + `\`\`\`\n\n### Insight\n* 요약: 자동 변환된 TIL 파일\n`;

return [{ json: { filename, content: tilContent } }];
```
* GitHub 노드 설정(내장 GitHub 노드 사용 예시):
  * Resource: File
  * Operation: Create Or Update
  * Owner: <your-github-username-or-org>
  * Repository: <repo-name>
  * File Path: expressions -> e.g. `workdays/til/{{$json["filename"]}}`
  * Content: expressions -> `{{$json["content"]}}` (n8n GitHub 노드가 내부적으로 Base64로 인코딩 처리함)
  * Commit Message: `Add TIL: {{$json["filename"]}}`

* GitHub HTTP API 대체(HTTP Request 노드) - 파일 생성/업데이트 예제
```json
PUT https://api.github.com/repos/{owner}/{repo}/contents/{path}
Headers:
  Authorization: token <GITHUB_TOKEN>
  Accept: application/vnd.github+json
Body (JSON):
{
  "message": "Add TIL: 2026-03-14-example.md",
  "content": "<BASE64_ENCODED_CONTENT>",
  // "sha": "<existing-file-sha>" // 업데이트 시 필요
}
```
* README 인덱스 추가 방법(안전한 순서):
  * 1) GitHub 노드 또는 HTTP Request로 README.md의 현재 content를 가져온다.
  * 2) Function 노드에서 Base64 디코딩 후 인덱스 라인(예: `- [YYYY-MM-DD 제목](path/to/file)`)을 맨 위 또는 원하는 위치에 추가.
  * 3) GitHub 노드로 다시 커밋(업데이트) 수행. 업데이트 시 기존 README의 SHA를 포함해야 충돌 방지.

* README 업데이트용 Function 노드 예시 (n8n 환경에서):
```javascript
const readme = Buffer.from($json["readme_content_base64"], 'base64').toString();
const newIndexLine = `- [${$json["filename"].replace('.md','')}](./${$json["repo_path"]}/${$json["filename"]})`;
const updated = newIndexLine + '\n' + readme;
return [{ json: { path: 'README.md', content: updated } }];
```

* 최종 완료 메시지 발신(Discord Webhook 또는 Discord 노드 사용):
  * 메시지 내용 예: `TIL 파일 생성 완료: <repo_url>/blob/main/<path>/<filename> — README에 인덱스 추가됨.`

* 운영상 고려사항 및 팁:
  * GitHub 업데이트 시 동시성 문제를 막기 위해 파일 생성(First commit)과 README 업데이트를 트랜잭션처럼 처리하거나, 실패 시 롤백 로직을 설계한다.
  * GitHub 노드가 제한적이면 HTTP Request로 REST API를 직접 호출(Authorisation: token)하여 create/update 파일 엔드포인트를 사용한다.
  * 디스코드에서 보낸 메시지가 멀티파트(첨부파일 포함)인 경우, n8n의 Binary Data 흐름을 활용해 첨부를 별도로 처리한다.
  * 시크릿(토큰)은 n8n Credentials로 관리하여 노드에서 참조한다.

### Insight
* 기술적 검증:
  * Discord에서 메시지를 받는 것은 n8n의 Webhook 노드 또는 Discord 전용 노드를 사용하면 가능하다(디스코드 쪽에는 Webhook을 생성하거나 Bot을 통해 이벤트를 보내야 함). n8n에서는 Webhook 노드의 입력(JSON을 파싱)만 정확히 이해하면 이후 로직은 직관적이다.
  * GitHub에 파일을 생성/업데이트할 때는 REST API의 PUT /repos/{owner}/{repo}/contents/{path} 또는 n8n의 GitHub 노드를 활용한다. 업데이트 시 기존 파일의 sha가 필요하다.
  * README를 업데이트하려면 현재 content를 읽어 base64로 디코딩한 뒤 변경하고 다시 업로드하는 과정이 필요하다.
* 개인적 소감(요약): 처음 n8n으로 워크플로우를 만든 경험은 노드의 입력/출력 데이터 구조만 명확히 알면 생각보다 간단했다. 두 개(트리거 노드와 액션 노드)의 입출력만 이해하면 되며, 이를 알려준 후츠릿님 덕분에 빠르게 구현했다.

**출처:** [n8n Webhook node](https://docs.n8n.io/nodes/n8n-nodes-base/webhook/)
**출처:** [n8n GitHub node (File create/update) — n8n docs](https://docs.n8n.io/nodes/n8n-nodes-base/github/)
**출처:** [GitHub REST API - Create or update file contents](https://docs.github.com/en/rest/repos/contents#create-or-update-file-contents)
**출처:** [Discord Webhooks documentation](https://discord.com/developers/docs/resources/webhook)
