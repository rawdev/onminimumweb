# Windows 로컬 MemoryWeft를 ChatGPT에 연결하는 방법

이 문서는 Windows PC에서 실행되는 로컬 MemoryWeft(MWeft) MCP 서버를
ChatGPT의 사용자 지정 앱으로 연결하는 절차를 설명합니다.

## 1. 연결 구조

MemoryWeft Manager가 제공하는 MCP 서버는 기본적으로 로컬 `stdio`
방식입니다. ChatGPT는 이 프로세스에 직접 접속할 수 없으므로 다음과
같이 HTTPS MCP 주소로 변환해야 합니다.

```text
MWeft stdio MCP
  -> Supergateway (Streamable HTTP)
  -> Cloudflare Tunnel (공개 HTTPS)
  -> ChatGPT 사용자 지정 MCP 앱
```

## 2. 준비물

- Windows PowerShell
- MemoryWeft Manager 및 생성된 프로젝트
- Node.js
- Cloudflared
- 사용자 지정 MCP 앱을 만들 수 있는 ChatGPT 계정

설치 여부를 확인합니다.

```powershell
node --version
npx.cmd --version
cloudflared --version
```

PowerShell 실행 정책 때문에 `npx`가 차단될 수 있습니다. 이 문서에서는
`npx` 대신 항상 `npx.cmd`를 사용합니다.

Cloudflared가 설치되지 않았다면 다음 명령으로 설치합니다.

```powershell
winget install --id Cloudflare.cloudflared
```

설치가 끝나면 PowerShell을 닫았다가 다시 실행한 후 확인합니다.

```powershell
cloudflared --version
```

## 3. MemoryWeft Manager에서 Show config 확인

MWeft 실행 경로와 도메인 설정은 임의로 작성하지 않고 MemoryWeft
Manager에서 가져옵니다.

메뉴 위치:

```text
MemoryWeft Manager
-> 설정
-> 도메인 설정
-> MCP 설치(AI 클라이언트)
-> ChatGPT Desktop
-> Show config
```

`Show config`에는 다음과 같은 JSON이 표시됩니다.

```json
{
  "mcpServers": {
    "mweft": {
      "command": "D:\\...\\Mweft 설치 경로\\k2g-mcp.exe",
      "args": [],
      "env": {
        "K2G_USER_MEMORY_SAVE_GROUP": "default",
        "K2G_USER_MEMORY_SAVE_DOMAIN": "default_domain",
        "K2G_USER_SEARCH_TARGETS": "default_domain",
        "K2G_MCP_LAZY_INIT": "true",
        "EMBEDDING_PROVIDER": "onnx",
        "EMBEDDING_MODEL": "BAAI/bge-m3",
        "EMBEDDING_ONNX_PATH":"C:\\Mweft 설치 경로\\models\\bge-m3-onnx",
        "EMBEDDING_DIM": "1024",
        "DATA_DIR": "D:\\AI\\프로젝트 경로"
      }
    }
  }
}
```

이 JSON 전체를 ChatGPT에 붙여 넣는 것은 아닙니다. 다음 값만 이후
PowerShell 명령에 사용합니다.

- `command`: `k2g-mcp.exe` 실행 경로
- `K2G_USER_MEMORY_SAVE_GROUP`: 저장 그룹
- `K2G_USER_MEMORY_SAVE_DOMAIN`: 저장 도메인
- `K2G_USER_SEARCH_TARGETS`: 검색 대상 도메인
- 임베딩 관련 설정
- `DATA_DIR`: 프로젝트 데이터 폴더

> JSON의 `\\`는 이스케이프 표현입니다. PowerShell 경로에는 일반
> 역슬래시 하나만 사용합니다.

```text
JSON:       D:\\AI\\MemoryWeftTest
PowerShell: D:\AI\프로젝트 경로
```

저장 도메인과 검색 대상은 Manager에 표시된 값을 그대로 사용해야
합니다. 예를 들어 기존 기억이 `Test` 도메인에 있다면 이를 임의로
`default`로 변경하지 않습니다.

## 4. 환경변수 설정

새 PowerShell 창에서 `Show config`의 값을 적용합니다.

```powershell
$env:K2G_DOTENV_FILE=""
$env:K2G_USER_MEMORY_SAVE_GROUP="default"
$env:K2G_USER_MEMORY_SAVE_DOMAIN="Test"
$env:K2G_USER_SEARCH_TARGETS="Test"
$env:K2G_MCP_LAZY_INIT="true"

$env:EMBEDDING_PROVIDER="onnx"
$env:EMBEDDING_MODEL="BAAI/bge-m3"
$env:EMBEDDING_ONNX_PATH="C:\Mweft 설치 경로\models\bge-m3-onnx"
$env:EMBEDDING_DIM="1024"
$env:DATA_DIR="D:\AI\프로젝트 경로"
```

위 값은 예시입니다. 특히 도메인과 경로는 자신의 `Show config` 값을
사용해야 합니다. `K2G_DOTENV_FILE=""`는 `.mwf` 파일을 사용하지 않겠다는
의미이며, `EMBEDDING_PROVIDER`와 `EMBEDDING_ONNX_PATH`는 `Show config`에
표시된 값(포터블 빌드는 `onnx`)을 그대로 사용합니다.

## 5. MWeft 실행 파일 확인

`Show config`의 `command` 경로가 실제로 존재하는지 확인합니다.

```powershell
Test-Path "C:\...\Mweft 설치 경로\runtime\venv\Scripts\k2g-mcp.exe"
```

결과가 `True`여야 합니다.

## 6. Streamable HTTP 서버 실행

먼저 프로젝트 폴더로 이동합니다.

```powershell
Set-Location "D:\AI\프로젝트 경로"
```

다음 명령을 한 줄로 실행합니다. `--stdio` 뒤의 경로는 자신의
`Show config`에 있는 `command` 값으로 바꿉니다.

```powershell
npx.cmd -y supergateway --stdio '"C:\...\Mweft 설치 경로\runtime\venv\Scripts\k2g-mcp.exe"' --outputTransport streamableHttp --port 8000 --streamableHttpPath /mcp --healthEndpoint /healthz
```

정상적으로 실행되면 다음과 비슷한 로그가 표시됩니다.

```text
[supergateway] Listening on port 8000
[supergateway] StreamableHttp endpoint: http://localhost:8000/mcp
```

이 PowerShell 창은 종료하지 않습니다.

## 7. 로컬 서버 상태 확인

새 PowerShell 창을 열고 실행합니다.

```powershell
Invoke-WebRequest http://localhost:8000/healthz
```

다음과 같이 `200`과 `ok`가 나오면 정상입니다.

```text
StatusCode : 200
Content    : ok
```

## 8. HTTPS 터널 생성

새 PowerShell 창에서 실행합니다.

```powershell
cloudflared tunnel --url http://localhost:8000
```

잠시 후 다음과 같은 임시 HTTPS 주소가 표시됩니다.

```text
https://random-name.trycloudflare.com
```

ChatGPT에 등록할 최종 MCP 주소는 끝에 `/mcp`를 붙인 주소입니다.

```text
https://random-name.trycloudflare.com/mcp
```

Cloudflared가 실행 중인 PowerShell 창도 종료하지 않습니다. 임시
터널을 다시 실행하면 주소가 바뀔 수 있습니다.

## 9. ChatGPT에 MWeft 앱 등록

ChatGPT에서 다음 메뉴로 이동합니다.

```text
설정
-> 앱
-> 고급 설정
-> 개발자 모드 활성화
-> 앱 만들기
```

다음과 같이 입력합니다.

| 항목 | 입력값 |
| --- | --- |
| 이름 | `MWeft` |
| 설명 | `개인 장기 기억을 저장하고 검색하는 MCP 시스템` |
| 연결 | `서버 URL` |
| 서버 URL | `https://random-name.trycloudflare.com/mcp` |
| 인증 | `인증 없음` |

주의 사항을 확인하고 동의 체크박스를 선택한 후 `만들기`를 누릅니다.

ChatGPT에는 MemoryWeft Manager의 JSON을 입력하지 않습니다. ChatGPT
앱 생성 화면에는 Cloudflare가 발급한 HTTPS 주소와 `/mcp` 경로만
입력합니다.

## 10. 연결 테스트

새 ChatGPT 대화에서 MWeft 앱을 활성화한 뒤 다음과 같이 요청합니다.

```text
MWeft의 mweft_search 도구를 사용해서 저장된 기억을 검색해줘.
```

쓰기 도구가 허용되는 계정에서는 다음 요청도 시험할 수 있습니다.

```text
MWeft의 mweft_remember 도구로 이 내용을 저장해줘.
```

## 11. 자주 발생하는 오류

### `npx.ps1` 실행 정책 오류

증상:

```text
PSSecurityException
UnauthorizedAccess
```

해결:

```powershell
npx.cmd --version
```

이후 모든 Supergateway 명령에서도 `npx` 대신 `npx.cmd`를 사용합니다.

### `cloudflared`를 찾을 수 없음

해결:

```powershell
winget install --id Cloudflare.cloudflared
```

설치 후 PowerShell을 다시 열고 확인합니다.

```powershell
cloudflared --version
```

### ChatGPT에서 앱을 만들 수 없음

다음을 확인합니다.

- Supergateway PowerShell 창이 실행 중인가?
- `http://localhost:8000/healthz`가 `200 OK`를 반환하는가?
- Cloudflared PowerShell 창이 실행 중인가?
- ChatGPT URL 끝에 `/mcp`가 포함되어 있는가?
- Cloudflare 임시 주소가 재실행으로 변경되지 않았는가?
- 인증 설정이 `OAuth`가 아니라 `인증 없음`인가?
- `K2G_DOTENV_FILE`가 `""`(빈 문자열)로 설정되어 있는가? (`.mwf config file not found` 방지)

## 12. 종료 방법

테스트를 마치면 다음 두 PowerShell 창에서 각각 `Ctrl+C`를 누릅니다.

1. Cloudflared 실행 창
2. Supergateway 실행 창

두 프로세스를 종료하면 외부에서 MWeft에 접속할 수 없습니다.

## 13. 보안 주의

`trycloudflare.com` 임시 터널과 `인증 없음`의 조합은 짧은 연결 시험에
적합합니다. URL을 아는 외부 사용자가 MWeft의 기억을 조회하거나 쓰기
도구를 실행할 위험이 있으므로 주소를 공유해서는 안 됩니다.

지속적으로 운영하려면 다음 구성을 권장합니다.

- 고정 Cloudflare Tunnel
- Cloudflare Access 등의 접근 제어
- OAuth 또는 별도의 인증 계층
- 읽기 도구와 쓰기 도구의 권한 분리
- 개인 기억이 아닌 별도 테스트 도메인에서 최초 검증

