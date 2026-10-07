# Workstation setup: Revit MCP over SSH

This guide connects Claude Code on a Linux machine to Revit running on a separate Windows workstation, using Autodesk's Revit Public MCP Server.

## How it works

The official Revit MCP server only supports the stdio transport: the AI client starts the server program as a local process and talks to it over stdin/stdout. To reach it from another machine, Claude Code on Linux starts the server through an SSH session instead. The MCP traffic travels inside that encrypted SSH connection, so the only open port is SSH (22).

```text
Linux                                   Windows workstation
-----                                   -------------------
Claude Code
   │  starts: ssh -T revit-ws revit-mcp.cmd
   ▼
ssh client  ───── SSH (port 22) ─────▶  sshd
                                          │
                                          ▼
                                        revit-mcp.cmd  (launcher in user profile)
                                          │
                                          ▼
                                        Revit MCP server .exe
                                          │  local connection
                                          ▼
                                        Revit 2027.2 (signed in, model open)
```

Steps are labelled with where they run: **[Win]** is the Revit workstation, **[Linux]** is the Claude Code machine.

## What you get

The official server is a Technical Preview and is **read-only**. It has seven tools:

| Tool | Purpose |
|---|---|
| `get_running_revit_instances` | List open Revit sessions |
| `query_model` | Filter elements by category, level, parameter |
| `get_element_data` | Read parameters of specific elements |
| `select_elements` | Select elements in the Revit UI |
| `zoom_to_elements` | Zoom the active view to elements |
| `open_view` | Open a view |
| `export_views` | Export views (images, schedules as CSV) |

It cannot change the model or trace MEP connectors. Those need a custom add-in and MCP server, which can use the same SSH route.

## Prerequisites

- Revit **2027.2** installed on the workstation, signed in, running, with a model open.
- Local admin rights on the workstation (to install the SSH server).
- Approval from IT security to run an SSH server on the workstation.
- Network access from the Linux machine to the workstation (same LAN or VPN).

---

## Step 1. [Win] Pre-checks

1. **Revit version.** In Revit, open Help > About. It must be 2027.2. If it is older, the official server is not available; stop and use a community server instead.
2. **Account type.** In Command Prompt, run `whoami` and note the output.
   - `azuread\...` means a Microsoft Entra ID account. SSH key login is not supported for these accounts; involve IT before continuing.
   - `domain\name` (domain account) or `pcname\name` (local account) is fine.
3. **Admin check.** Run `whoami /groups | findstr S-1-5-32-544`. If it prints a line, the account is in the Administrators group. This changes where the SSH key goes in Step 5.

## Step 2. [Win] Install the Revit MCP Server add-on

1. Sign in to your Autodesk Account. Go to **Products and Services > Revit > View Details > Extensions** and download **Revit 2027 MCP Server**.
2. Close Revit. Run the installer. Reopen Revit and open a model.
3. Find the server executable. Sources give different file names (`Autodesk.RevitMcpServer.Stdio.exe` or `RevitMCPServer.exe`), so search for it. Run this in a **normal (non-admin) PowerShell** window and keep the window open for Step 6:

```powershell
$exe = Get-ChildItem "C:\Program Files\Autodesk" -Recurse -Filter "*Mcp*.exe" -ErrorAction SilentlyContinue | Select-Object -First 1 -ExpandProperty FullName
$exe
```

Check the printed path. If it picked the wrong file, set it by hand, for example `$exe = "C:\Program Files\Autodesk\...\Autodesk.RevitMcpServer.Stdio.exe"`.

## Step 3. [Win] Install the OpenSSH Server

Run in **PowerShell as administrator**:

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
Get-NetFirewallRule -Name OpenSSH-Server-In-TCP
```

The last command should show an enabled rule. If the rule is missing, create it:

```powershell
New-NetFirewallRule -Name 'OpenSSH-Server-In-TCP' -DisplayName 'OpenSSH Server (sshd)' -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
```

If `Add-WindowsCapability` fails with an update or policy error, the PC probably gets updates from a company server (WSUS). Ask IT to allow the OpenSSH feature.

## Step 4. [Linux] Create a dedicated SSH key and host alias

Create a key used only for this connection. It has no passphrase because Claude Code starts the connection in the background and cannot type one. Keeping it separate means you can revoke it without touching other access.

```bash
ssh-keygen -t ed25519 -f ~/.ssh/revit_ws -N "" -C "kelix-revit-mcp"
```

Print the public key and copy the single line for Step 5:

```bash
cat ~/.ssh/revit_ws.pub
```

Add this entry to `~/.ssh/config` (create the file if it does not exist):

```text
Host revit-ws
    HostName <windows-hostname-or-ip>
    User <username@domain>
    IdentityFile ~/.ssh/revit_ws
    IdentitiesOnly yes
    BatchMode yes
    StrictHostKeyChecking accept-new
    ServerAliveInterval 30
```

For `User`, reverse the `whoami` output from Step 1. For example, `meinhardt\rohit.jain` becomes `rohit.jain@meinhardt`. For a local account, use just the name.

| Option | Why |
|---|---|
| `BatchMode yes` | Fail immediately instead of waiting for a password prompt nobody can answer |
| `StrictHostKeyChecking accept-new` | Accept the workstation's host key on first connect, reject it if it later changes |
| `ServerAliveInterval 30` | Detect a dropped connection instead of hanging |

## Step 5. [Win] Install the public key

Paste the line from Step 4 in place of `PASTE-KEY-LINE-HERE`.

**Standard user** (normal PowerShell):

```powershell
New-Item -Force -ItemType Directory -Path "$env:USERPROFILE\.ssh"
Add-Content -Path "$env:USERPROFILE\.ssh\authorized_keys" -Value 'PASTE-KEY-LINE-HERE'
```

**Administrator account** (PowerShell as administrator). For admin accounts, Windows OpenSSH ignores the per-user file and reads a shared file, which must be readable only by Administrators and SYSTEM:

```powershell
Add-Content -Path "$env:ProgramData\ssh\administrators_authorized_keys" -Value 'PASTE-KEY-LINE-HERE'
icacls.exe "$env:ProgramData\ssh\administrators_authorized_keys" /inheritance:r /grant "Administrators:F" /grant "SYSTEM:F"
```

Use `Add-Content` as shown. Writing the file with `>` or `Out-File` in Windows PowerShell saves it as UTF-16, which sshd cannot read.

## Step 6. [Win] Create the launcher

The server path contains spaces, and stdio must carry nothing except MCP messages. A two-line batch file in your profile folder solves both: `@echo off` stops cmd from echoing the command into the output stream.

Run in the **same PowerShell window as Step 2**, so `$exe` is still set:

```powershell
Set-Content -Path "$env:USERPROFILE\revit-mcp.cmd" -Value '@echo off', "`"$exe`""
```

SSH sessions start in the user profile folder, so the launcher can be called simply as `revit-mcp.cmd`.

## Step 7. [Linux] Test SSH

```bash
ssh revit-ws whoami
```

This must print the same account as `whoami` on the workstation. The MCP server can only reach Revit when it runs as the same Windows user who has Revit open.

```bash
ssh revit-ws 'tasklist /fi "imagename eq Revit.exe"'
```

This must list `Revit.exe`.

Optional stdio check: run the launcher directly. It should start and wait silently for input. Press Ctrl+C to exit. If it prints anything before you type, that text will break MCP.

```bash
ssh -T revit-ws revit-mcp.cmd
```

## Step 8. [Linux] Register the server in Claude Code

Run from this repository folder (the default `local` scope applies the server to this project only):

```bash
claude mcp add revit -- ssh -T revit-ws revit-mcp.cmd
```

```bash
claude mcp list
```

`revit` should show as connected. Start a new Claude Code session in this folder and ask it to list the running Revit instances.

## Step 9. [Win] Harden and keep it running

**Restrict SSH to the Linux machine.** Get the Linux IP with `hostname -I`, then on the workstation (admin PowerShell):

```powershell
Set-NetFirewallRule -Name OpenSSH-Server-In-TCP -RemoteAddress <linux-ip>
```

**Disable password login** once key login works:

1. Open `C:\ProgramData\ssh\sshd_config` as administrator.
2. Find the existing `#PasswordAuthentication yes` line and change it to `PasswordAuthentication no`. Edit it in place; do not append it at the end of the file, because settings after a `Match` block only apply to that block.
3. Run `Restart-Service sshd`.

**Prevent sleep.** Sleep drops the SSH connection:

```powershell
powercfg /change standby-timeout-ac 0
```

**Day-to-day rules for the workstation:**

- Stay signed in. Do not sign out. Locking the screen should be fine, but test it once.
- Keep Revit open with a model loaded. Queries against the Revit home screen return nothing.
- Keep Remote Desktop access. Revit only accepts API calls when idle, so any open dialog (warning, save prompt, crash report) blocks requests until someone closes it.
- Set Windows Update active hours. An automatic restart closes Revit.
- Export to a local folder. An SSH session opened with a key has no network credentials, so it cannot write to network shares.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Permission denied (publickey)` | Key in the wrong file (admin vs standard user) | Check Step 1 admin result and redo Step 5 |
| | Wrong `User` format | Use `name@domain` for domain accounts |
| | Entra ID (`azuread\`) account | Not supported for key login; involve IT |
| | `authorized_keys` saved as UTF-16 | Recreate with `Add-Content` |
| `Connection refused` / timeout | sshd stopped, firewall rule missing or restricted to another IP | `Get-Service sshd`, `Get-NetFirewallRule -Name OpenSSH-Server-In-TCP` |
| `claude mcp list` shows failed | Launcher prints text, or wrong exe path | Run the Step 7 stdio check; check `revit-mcp.cmd` contents |
| Connected, but no Revit instances found | Server cannot see Revit from the SSH logon session | See "Fallback" below |
| Requests hang | A Revit dialog is open, or Revit is busy | Close the dialog over Remote Desktop |
| View export fails | Target is a network share | Export to a local folder |

### Fallback: server cannot see Revit over SSH

Windows runs SSH logins in a separate logon session from the desktop. If the server connects but cannot find the running Revit, run the server inside the desktop session instead, behind a stdio-to-HTTP bridge that listens only on `localhost`. Then reach it from Linux through an SSH tunnel (`ssh -L`) and register it in Claude Code as an HTTP server. This keeps the port closed to the network.

## Revoking access

- [Win] Delete the key line from `authorized_keys` (or `administrators_authorized_keys`).
- [Win] To remove SSH entirely: `Stop-Service sshd`, `Set-Service sshd -StartupType Disabled`, `Disable-NetFirewallRule -Name OpenSSH-Server-In-TCP`.
- [Linux] `claude mcp remove revit`, delete the `revit-ws` entry in `~/.ssh/config`, delete `~/.ssh/revit_ws` and `~/.ssh/revit_ws.pub`.

## References

- [Autodesk MCP Server Help: Revit MCP Server](https://help.autodesk.com/view/ADSKMCP/ENU/?guid=ADSKMCP_RevitMcp_revit_mcp_server_html)
- [Autodesk Developer Blog: Revit 2027 MCP Server (Tech Preview)](https://blog.autodesk.io/revit-2027-mcp-server-release/)
- [CAD Forum: Revit MCP Server not showing in Claude (exe path)](https://www.cadforum.cz/en/why-isn-t-revit-mcp-server-showing-up-in-my-claude-ai-tool-tip15125)
- [Microsoft Learn: Get started with OpenSSH Server for Windows](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse)
- [Microsoft Learn: Key-based authentication in OpenSSH for Windows](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_keymanagement)
- [adity982/revit-model-mcp: SSH + stdio remote pattern](https://github.com/adity982/revit-model-mcp)
