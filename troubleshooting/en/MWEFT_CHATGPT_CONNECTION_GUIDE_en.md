# Connecting a local Windows MemoryWeft to ChatGPT

This document explains how to connect a local MemoryWeft (MWeft) MCP server
running on a Windows PC to ChatGPT as a custom app.

## 1. Connection architecture

The MCP server provided by MemoryWeft Manager is, by default, a local
`stdio` server. ChatGPT cannot connect to that process directly, so it must
be converted into an HTTPS MCP endpoint as follows.

```text
MWeft stdio MCP
  -> Supergateway (Streamable HTTP)
  -> Cloudflare Tunnel (public HTTPS)
  -> ChatGPT custom MCP app
```

## 2. Prerequisites

- Windows PowerShell
- MemoryWeft Manager and a created project
- Node.js
- Cloudflared
- A ChatGPT account that can create custom MCP apps

Check whether they are installed.

```powershell
node --version
npx.cmd --version
cloudflared --version
```

The PowerShell execution policy may block `npx`. This document always uses
`npx.cmd` instead of `npx`.

If Cloudflared is not installed, install it with the following command.

```powershell
winget install --id Cloudflare.cloudflared
```

After installation, close and reopen PowerShell, then verify.

```powershell
cloudflared --version
```

## 3. Get the values from MemoryWeft Manager "Show config"

Do not write the MWeft executable path and domain settings by hand — take
them from MemoryWeft Manager.

Menu location:

```text
MemoryWeft Manager
-> Settings
-> Domain settings
-> MCP install (AI client)
-> ChatGPT Desktop
-> Show config
```

`Show config` displays JSON like the following.

```json
{
  "mcpServers": {
    "mweft": {
      "command": "D:\\...\\Mweft-install-path\\k2g-mcp.exe",
      "args": [],
      "env": {
        "K2G_USER_MEMORY_SAVE_GROUP": "default",
        "K2G_USER_MEMORY_SAVE_DOMAIN": "default_domain",
        "K2G_USER_SEARCH_TARGETS": "default_domain",
        "K2G_MCP_LAZY_INIT": "true",
        "EMBEDDING_PROVIDER": "onnx",
        "EMBEDDING_MODEL": "BAAI/bge-m3",
        "EMBEDDING_ONNX_PATH":"C:\\Mweft-install-path\\models\\bge-m3-onnx",
        "EMBEDDING_DIM": "1024",
        "DATA_DIR": "D:\\AI\\project-path"
      }
    }
  }
}
```

You do not paste this entire JSON into ChatGPT. Only the following values are
used in the later PowerShell commands.

- `command`: path to `k2g-mcp.exe`
- `K2G_USER_MEMORY_SAVE_GROUP`: save group
- `K2G_USER_MEMORY_SAVE_DOMAIN`: save domain
- `K2G_USER_SEARCH_TARGETS`: search target domain
- embedding settings
- `DATA_DIR`: project data folder

> The `\\` in JSON is an escaped representation. In PowerShell paths, use a
> single backslash.

```text
JSON:       D:\\AI\\MemoryWeftTest
PowerShell: D:\AI\project-path
```

Use the save domain and search target exactly as shown in Manager. For
example, if your existing memories are in the `Test` domain, do not change it
to `default` arbitrarily.

## 4. Set environment variables

In a new PowerShell window, apply the values from `Show config`.

```powershell
$env:K2G_DOTENV_FILE=""
$env:K2G_USER_MEMORY_SAVE_GROUP="default"
$env:K2G_USER_MEMORY_SAVE_DOMAIN="Test"
$env:K2G_USER_SEARCH_TARGETS="Test"
$env:K2G_MCP_LAZY_INIT="true"

$env:EMBEDDING_PROVIDER="onnx"
$env:EMBEDDING_MODEL="BAAI/bge-m3"
$env:EMBEDDING_ONNX_PATH="C:\Mweft-install-path\models\bge-m3-onnx"
$env:EMBEDDING_DIM="1024"
$env:DATA_DIR="D:\AI\project-path"
```

The values above are examples. In particular, the domain and paths must use
your own `Show config` values. `K2G_DOTENV_FILE=""` means "do not use a `.mwf`
file", and `EMBEDDING_PROVIDER` / `EMBEDDING_ONNX_PATH` should be used exactly
as shown in `Show config` (the portable build uses `onnx`).

## 5. Verify the MWeft executable

Check that the `command` path from `Show config` actually exists.

```powershell
Test-Path "C:\...\Mweft-install-path\runtime\venv\Scripts\k2g-mcp.exe"
```

The result must be `True`.

## 6. Run the Streamable HTTP server

First, move into the project folder.

```powershell
Set-Location "D:\AI\project-path"
```

Run the following as a single line. Replace the path after `--stdio` with the
`command` value from your `Show config`.

```powershell
npx.cmd -y supergateway --stdio '"C:\...\Mweft-install-path\runtime\venv\Scripts\k2g-mcp.exe"' --outputTransport streamableHttp --port 8000 --streamableHttpPath /mcp --healthEndpoint /healthz
```

When it starts correctly, you will see logs similar to the following.

```text
[supergateway] Listening on port 8000
[supergateway] StreamableHttp endpoint: http://localhost:8000/mcp
```

Do not close this PowerShell window.

## 7. Check the local server status

Open a new PowerShell window and run:

```powershell
Invoke-WebRequest http://localhost:8000/healthz
```

It is working if you see `200` and `ok` as below.

```text
StatusCode : 200
Content    : ok
```

## 8. Create the HTTPS tunnel

In a new PowerShell window, run:

```powershell
cloudflared tunnel --url http://localhost:8000
```

After a moment, a temporary HTTPS address like the following appears.

```text
https://random-name.trycloudflare.com
```

The final MCP address to register in ChatGPT is that address with `/mcp`
appended.

```text
https://random-name.trycloudflare.com/mcp
```

Do not close the PowerShell window running Cloudflared either. Restarting the
temporary tunnel may change the address.

## 9. Register the MWeft app in ChatGPT

In ChatGPT, go to the following menu.

```text
Settings
-> Apps
-> Advanced settings
-> Enable developer mode
-> Create app
```

Enter the following.

| Field | Value |
| --- | --- |
| Name | `MWeft` |
| Description | `An MCP system that stores and searches personal long-term memory` |
| Connection | `Server URL` |
| Server URL | `https://random-name.trycloudflare.com/mcp` |
| Authentication | `No authentication` |

Review the warnings, check the consent box, then click `Create`.

You do not enter the MemoryWeft Manager JSON into ChatGPT. On the ChatGPT app
creation screen, enter only the HTTPS address issued by Cloudflare with the
`/mcp` path.

## 10. Test the connection

In a new ChatGPT conversation, enable the MWeft app and make a request like:

```text
Use MWeft's mweft_search tool to search my saved memories.
```

On accounts where write tools are allowed, you can also try:

```text
Use MWeft's mweft_remember tool to save this content.
```

## 11. Common errors

### `npx.ps1` execution policy error

Symptom:

```text
PSSecurityException
UnauthorizedAccess
```

Fix:

```powershell
npx.cmd --version
```

After this, use `npx.cmd` instead of `npx` in all Supergateway commands too.

### `cloudflared` not found

Fix:

```powershell
winget install --id Cloudflare.cloudflared
```

After installing, reopen PowerShell and verify.

```powershell
cloudflared --version
```

### Cannot create the app in ChatGPT

Check the following.

- Is the Supergateway PowerShell window still running?
- Does `http://localhost:8000/healthz` return `200 OK`?
- Is the Cloudflared PowerShell window still running?
- Does the ChatGPT URL end with `/mcp`?
- Has the Cloudflare temporary address changed due to a restart?
- Is authentication set to `No authentication`, not `OAuth`?
- Is `K2G_DOTENV_FILE` set to `""` (empty string)? (prevents `.mwf config file not found`)

## 12. How to shut down

When testing is done, press `Ctrl+C` in each of the two PowerShell windows.

1. The Cloudflared window
2. The Supergateway window

Once both processes stop, MWeft can no longer be reached from outside.

## 13. Security note

The combination of a `trycloudflare.com` temporary tunnel and
`No authentication` is suitable only for short connection tests. Anyone who
knows the URL can read MWeft's memories or run write tools, so the address
must not be shared.

For continuous operation, the following setup is recommended.

- A fixed Cloudflare Tunnel
- Access control such as Cloudflare Access
- OAuth or a separate authentication layer
- Separating read-tool and write-tool permissions
- Validating first in a separate test domain rather than your personal memory
