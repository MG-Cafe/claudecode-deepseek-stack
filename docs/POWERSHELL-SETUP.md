# PowerShell Setup (Windows)

Full setup for Windows users running Claude Code via PowerShell.

## Prerequisites

- Claude Code installed (`claude --version`)
- DeepSeek account + API key from `platform.deepseek.com`
- Bun installed (`bun --version`) — used to run ccusage

## 1. Create Config Directory

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config\mg-deepseek"
icacls "$env:USERPROFILE\.config\mg-deepseek" /inheritance:r /grant:r "${env:USERNAME}:(OI)(CI)F"
```

## 2. Create Settings Files

Save both to `~/.config/mg-deepseek/`. Replace `sk-YOUR-KEY` with your DeepSeek key.

**`deepseek-pro-settings.json`** (v4-pro, Opus equivalent):
```json
{
  "env": {
    "CLAUDE_CODE_USE_VERTEX": "",
    "ANTHROPIC_VERTEX_PROJECT_ID": "",
    "CLOUD_ML_REGION": "",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "",
    "ANTHROPIC_BASE_URL": "https://api.deepseek.com/anthropic",
    "ANTHROPIC_AUTH_TOKEN": "sk-YOUR-KEY",
    "ANTHROPIC_API_KEY": "sk-YOUR-KEY"
  }
}
```

**`deepseek-flash-settings.json`** (v4-flash, Haiku equivalent): same structure, same key.

## 3. Install ccusage

```powershell
bun install -g ccusage
```

## 4. Add to PowerShell Profile

Find your profile path: `echo $PROFILE`. Add the following:

```powershell
function Get-WeeklySpend {
    try {
        $monday = (Get-Date).AddDays(-[int](Get-Date).DayOfWeek + 1).ToString("yyyyMMdd")
        $raw = ccusage weekly -j --offline -s $monday 2>&1
        $jsonStr = ($raw | Where-Object { $_ -notmatch 'WARN|INFO|Fetching|Loaded|Resolving|Resolved|Saved|\[ccusage\]' }) -join ""
        $json = $jsonStr | ConvertFrom-Json
        return [math]::Round($json.totals.totalCost, 2)
    } catch { return 0 }
}

function Get-ThrottleStatus {
    $spend = Get-WeeklySpend
    $budget = 700  # set to 80% of your weekly Anthropic cap
    return @{ Spend = $spend; Throttled = ($spend -gt $budget) }
}

function ds-pro {
    claude --bare --settings "$env:USERPROFILE\.config\mg-deepseek\deepseek-pro-settings.json" --model deepseek-v4-pro --dangerously-skip-permissions @args
}
function ds-flash {
    claude --bare --settings "$env:USERPROFILE\.config\mg-deepseek\deepseek-flash-settings.json" --model deepseek-v4-flash --dangerously-skip-permissions @args
}
function claude-sonnet {
    claude --model claude-sonnet-4-6 --dangerously-skip-permissions @args
}
function claude-opus {
    $status = Get-ThrottleStatus
    if ($status.Throttled) {
        Write-Host "[$([math]::Round($status.Spend,2))/wk > budget] Throttled → DeepSeek v4-pro"
        ds-pro @args
    } else {
        claude --model claude-opus-4-6 --dangerously-skip-permissions @args
    }
}
function cs {
    $status = Get-ThrottleStatus
    if ($status.Throttled) {
        Write-Host "[$([math]::Round($status.Spend,2))/wk > budget] Throttled → DeepSeek v4-pro"
        ds-pro @args
    } else {
        claude --model claude-sonnet-4-6 --dangerously-skip-permissions @args
    }
}
function deepseek-pro   { claude --bare --settings "$env:USERPROFILE\.config\mg-deepseek\deepseek-pro-settings.json" --model deepseek-v4-pro @args }
function deepseek-flash { claude --bare --settings "$env:USERPROFILE\.config\mg-deepseek\deepseek-flash-settings.json" --model deepseek-v4-flash @args }
function rotate-deepseek-key {
    param([string]$NewKey)
    $files = @(
        "$env:USERPROFILE\.config\mg-deepseek\deepseek-pro-settings.json",
        "$env:USERPROFILE\.config\mg-deepseek\deepseek-flash-settings.json"
    )
    foreach ($file in $files) {
        $content = Get-Content $file -Raw
        $content = $content -replace 'sk-[a-f0-9]+', $NewKey
        Set-Content $file $content
        Write-Host "Updated: $file"
    }
}
function usage {
    $status = Get-ThrottleStatus
    $pct = if ($status.Spend -gt 0) { [math]::Round(($status.Spend / 875) * 100, 1) } else { 0 }
    $bar = "#" * [int]($pct / 5)
    $throttled = if ($status.Throttled) { " [THROTTLED -> DeepSeek]" } else { "" }
    Write-Host "Week: `$$($status.Spend) / ~`$875$throttled"
    Write-Host "[$bar] $pct%"
}
```

## 5. Verify

```powershell
. $PROFILE
usage
ds-flash -p "say: works"
ds-pro -p "say: works"
```

## Rotate Key

```powershell
rotate-deepseek-key sk-YOURNEWKEY
```
