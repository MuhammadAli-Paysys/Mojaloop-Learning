# Mojaloop Core Test Harness: One-File AI Deployment

## For the participant (30 seconds)

1. Install **Docker Desktop** (WSL 2 backend, Linux containers) and **Git**. If WSL is missing, run `wsl --install` in an *Administrator* PowerShell and restart Windows.
2. Open Docker Desktop and wait until it says the engine is running.
3. Open **Claude Code** (or any Claude with terminal access) in an empty folder, attach or paste this file, and say:

   > Follow MOJALOOP-AI-DEPLOY.md exactly and deploy everything. Only stop to ask me if something needs me (admin rights, restart, BIOS).

That's it. Claude does the rest and reports back when the Testing Toolkit is ready.

---

## INSTRUCTIONS FOR CLAUDE (the agent)

You are deploying the upstream **Mojaloop Core Test Harness** locally with Docker. Work autonomously, run commands yourself, and fix problems using this document. Commands are **Windows PowerShell**. On macOS/Linux use bash for the same `git` and `docker` commands and make the Phase 2 edits with your file-edit tool.

### Rules

- Do **not** change image versions, `.env`, or anything outside the two edits in Phase 2.
- Do **not** run `docker system prune`, `docker volume rm`, `docker compose down -v`, or reset Docker.
- Do **not** stop or remove containers you did not create. Identify the conflict and ask first.
- Always run compose from inside the `ml-core-test-harness` folder with the profiles shown.
- Ask the user only for things you cannot do: installing software, admin/UAC prompts, Windows restart, enabling BIOS virtualization, starting Docker Desktop.
- Do not tell the user it worked until Phase 5 checks pass.

Define this once per session and reuse it:

```powershell
$P = @('--profile','all-services','--profile','ttk-provisioning','--profile','ttk-tests')
```

### Phase 0: Preflight

```powershell
git --version
wsl --status
docker version
docker compose version
docker info --format '{{.OSType}}'
```

- `docker info` must print `linux` and `docker version` must show both Client and Server. If the engine is not reachable, ask the user to start Docker Desktop and wait, then retry.
- If Git or Docker is missing, ask the user to install it. Do not try to install silently.
- Check resources:

```powershell
"{0:N1} GB RAM" -f ((Get-CimInstance Win32_ComputerSystem).TotalPhysicalMemory/1GB)
"{0:N1} GB free on C:" -f ((Get-PSDrive C).Free/1GB)
```

  Recommended: about 16 GB RAM and 20 GB or more free disk. Below 8 GB RAM, warn the user that the stack will probably fail, and continue only if they agree. Below 20 GB free disk, warn them.
- Check for conflicts:

```powershell
Get-NetTCPConnection -LocalPort 9660,9550,9440 -State Listen -ErrorAction SilentlyContinue
docker ps -a --format '{{.Names}}'
```

  If those ports are in use, or containers from an earlier harness exist (for example `mojaloop-testing-toolkit`, `central-ledger`), tell the user what you found and ask before touching anything. Only one full harness can run per machine because container names and the `mojaloop-net` network are fixed.

### Phase 1: Clone and pin

If `ml-core-test-harness` already exists in the working folder, reuse it and just run the `git checkout` line. Otherwise:

```powershell
git -c core.autocrlf=false clone https://github.com/mojaloop/ml-core-test-harness.git
Set-Location -LiteralPath '.\ml-core-test-harness'
git checkout 21875204e04e508c8606babe92a08105ee736f40
```

A detached HEAD message is expected. Keep the tracked `.env` as is. `core.autocrlf=false` preserves LF endings, which the container shell scripts need.

### Phase 2: Apply the two required fixes (scripted, idempotent)

Run from inside `ml-core-test-harness`. Safe to run more than once.

```powershell
$ErrorActionPreference = 'Stop'
$utf8 = New-Object System.Text.UTF8Encoding($false)

# Fix 1: central-settlement health check (only inside that service block)
$f = Join-Path (Get-Location) 'docker-compose.yml'
$lines = [System.IO.File]::ReadAllText($f).Split("`n")
$inBlock = $false; $changed = 0
for ($i = 0; $i -lt $lines.Count; $i++) {
  $l = $lines[$i].TrimEnd("`r")
  if ($l -match '^  central-settlement:\s*$') { $inBlock = $true; continue }
  if ($inBlock -and $l -match '^  \S') { $inBlock = $false }
  if ($inBlock -and $lines[$i] -match '127\.0\.0\.1:3001/health') {
    $lines[$i] = $lines[$i] -replace '127\.0\.0\.1:3001/health','127.0.0.1:3007/v2/health'
    $changed++
  }
}
[System.IO.File]::WriteAllText($f, ($lines -join "`n"), $utf8)
"docker-compose.yml: $changed line(s) changed (1 expected on first run, 0 if already patched)"

# Fix 2: add the deferred settlement Kafka topic after topic-notification-event
$k = Join-Path (Get-Location) 'docker/kafka/scripts/provision.sh'
$text = [System.IO.File]::ReadAllText($k)
if ($text -notmatch 'topic-deferredsettlement-close') {
  $re = [regex]::new('(?m)^([ \t]*)"topic-notification-event"[ \t]*$')
  $text = $re.Replace($text, { param($m) $m.Value + "`n" + $m.Groups[1].Value + '"topic-deferredsettlement-close"' }, 1)
  [System.IO.File]::WriteAllText($k, $text, $utf8)
}
Select-String -Path $k -Pattern 'topic-deferredsettlement-close'
```

Verify both edits:

```powershell
git diff -- docker-compose.yml docker/kafka/scripts/provision.sh
Select-String -Path docker-compose.yml -Pattern '3007/v2/health'
docker compose @P config --quiet
```

Expected: exactly one changed line in `docker-compose.yml` (3001/health to 3007/v2/health, inside `central-settlement`) and one added line in `provision.sh`. `config --quiet` must return with no error.

If the script changed nothing and the diff is empty, the file layout differs from expectations. Make the two edits manually with your file-edit tool, keeping LF line endings, then re-run the verification. Do not touch health checks of any other service.

### Phase 3: Optional offline images

If the workshop host gave the user an image archive (for example `mojaloop-images.tar`), load it first to skip most downloads:

```powershell
docker load -i .\mojaloop-images.tar
```

Skip this phase if there is no archive.

### Phase 4: Pull and deploy

```powershell
docker compose @P pull
docker compose @P up -d
```

If `pull` fails, check internet or proxy access and retry up to 3 times. Image pulls on shared venue Wi-Fi can be slow, so be patient. Do not change image tags.

### Phase 5: Wait and verify (do not skip)

First startup takes several minutes. `up -d` returning does **not** mean the tests finished. Poll every 30 seconds for up to about 20 minutes:

```powershell
$deadline = (Get-Date).AddMinutes(20)
do {
  Start-Sleep -Seconds 30
  $out = docker compose @P ps -a ttk-tests | Out-String
  $out
} until (($out -match 'Exited \((\d+)\)') -or ((Get-Date) -gt $deadline))
```

Then verify:

```powershell
docker compose @P ps -a
docker compose --profile ttk-provisioning --profile ttk-tests logs --tail 100 ttk-provisioning ttk-tests
```

Pass criteria:

- Long-running services are running, and those with health checks are healthy.
- `kafka-provisioning`, `ttk-provisioning` and `ttk-tests` show `Exited (0)`. This is normal for finished one-shot containers.
- The `ttk-tests` report shows **zero failed assertions**. Reports are also in the repo's `reports` folder.

If anything fails, use the recovery table below, fix, and re-verify.

### Phase 6: Final report to the user

Reply with a short table: each check (Docker, clone, 2 fixes, services healthy, provisioning, tests) with pass or fail. Then give these links:

| URL | Purpose |
| --- | --- |
| http://localhost:9660 | Testing Toolkit UI |
| http://localhost:9660/admin/reports | Provisioning and test reports |
| http://localhost:9660/mobilesimulator | Sample mobile simulator |

Tell the user how to run a transfer again:

```powershell
docker compose --profile ttk-tests run --rm --no-deps ttk-tests
```

Or in the UI: **Test Runner**, then **Collection Manager**, then **Import Folder**, choose `docker/ml-testing-toolkit/test-cases/collections/tests`, select `p2p.json`, click **Run**.

---

## Recovery table (for the agent)

| Symptom | Action |
| --- | --- |
| Docker engine not reachable | Ask the user to start Docker Desktop and wait; confirm Linux containers mode and WSL status. |
| Missing Virtual Machine Platform / WSL | Ask the user to run `wsl --install` in an Administrator PowerShell, restart Windows, reopen Docker Desktop. If it still fails, virtualization may be disabled in BIOS; tell the user. |
| `central-settlement` unhealthy | Re-check the `3007/v2/health` fix is inside that service only. Read its logs. |
| Deferred settlement topic missing | Edit alone does not create it. Re-run `docker compose @P up -d` so `kafka-provisioning` runs again after Kafka is ready, then re-check. |
| `^M` or `bad interpreter` in logs | The `.sh` file has CRLF. Convert to LF (re-run the Phase 2 script, or delete the clone and re-clone with `core.autocrlf=false`), then redeploy. |
| `port is already allocated` or name conflict | Another stack is using it. Identify the owner, report to the user, and ask before stopping anything. |
| Containers killed or very slow | Low RAM or Docker/WSL limits. Check `docker stats --no-stream` and free disk. Suggest closing heavy apps. Optionally, with the user's consent, raise WSL memory in `%UserProfile%\.wslconfig` (`[wsl2]` then `memory=8GB`), run `wsl --shutdown`, and restart Docker Desktop. |
| `exec format error` or platform/architecture errors (ARM Macs or Windows on ARM) | Images may need amd64 emulation. Report this to the user rather than changing image versions. |
| Image pull fails | Check internet/proxy, retry, or use the offline archive from Phase 3. Never upgrade tags. |
| `ttk-tests` never starts | `ttk-provisioning` probably failed. Read its logs, fix the cause, then re-run `docker compose @P up -d`. |
| Mobile simulator keeps loading | First confirm the functional test passes. Then check toolkit logs and provisioning. |

Read-only diagnostics:

```powershell
docker compose --profile all-services ps -a
docker compose --profile all-services logs --tail 100 central-settlement
docker compose --profile all-services logs --tail 100 account-lookup-service
docker compose --profile all-services logs --tail 100 mojaloop-testing-toolkit
docker stats --no-stream
```

---

## Everyday commands (for the participant)

Stop everything (keeps data):

```powershell
docker compose --profile all-services stop
```

Start again after a laptop restart (open Docker Desktop first; do **not** re-provision):

```powershell
docker compose --profile all-services up -d
docker compose --profile all-services ps -a
```

Never delete volumes or reset Docker if you want to keep your workshop data.
