---
name: obsidian-claudian-gpt-config
description: Configure GPT/Codex and Claudian in an Obsidian vault on Windows. Use when the user asks to set up a new Obsidian knowledge base with BRAT/Claudian/GPT, fix Claudian after installing or upgrading it, repair "Claudian plugin failed to load", diagnose BOM JSON parse errors, or find/configure codex.exe for Obsidian Claudian.
---

# Obsidian Claudian GPT Config

This skill is self-contained. It does not require bundled scripts. When migrating this skill to another environment, copying only this `SKILL.md` is enough.

## Required Inputs

Before editing anything, obtain:

- Vault path, for example an Obsidian knowledge base folder.
- `codex.exe` path. If the user says they do not know, find the real executable.
- Scenario:
  - `new`: configuring a newly created vault.
  - `upgrade`: configuring after Claudian was installed or upgraded.

Do not ask the user to choose `claudian` or `realclaudian`. Determine the active plugin ID from the vault: prefer the one already enabled in `community-plugins.json`; otherwise use the only installed Claudian plugin folder. If both folders exist and neither is enabled, stop and explain that the vault has an ambiguous plugin state.

## Critical Rules

- Always write Obsidian and Claudian JSON as UTF-8 without BOM. Do not use Windows PowerShell `Set-Content -Encoding UTF8` for these files.
- Use `[System.Text.UTF8Encoding]::new($false)` and `[System.IO.File]::WriteAllText(...)`.
- Treat `claudian` and `realclaudian` as separate plugin IDs/folders, not as interchangeable names.
- Do not enable both plugin IDs at the same time unless the user explicitly asks.
- BRAT repository may still be `YishenTu/claudian` even when the installed plugin ID is `realclaudian`.
- Do not manage Claudian version numbers during GPT/Codex configuration. Read manifest versions only for diagnostics/reporting.
- Never print API keys or copied environment variables in the final answer.
- Before changing live files, inspect current state and preserve user-created files. Do not delete plugin folders just to switch IDs; prefer changing `community-plugins.json`.

## Workflow

1. Inspect the vault:
   - `.obsidian/community-plugins.json`
   - `.obsidian/plugins/*/manifest.json`
   - `.obsidian/plugins/obsidian42-brat/data.json`
   - `.claudian/claudian-settings.json`
   - `.obsidian/plugins/<plugin-id>/data.json`

2. Check JSON encoding and parse state:
   - Detect BOM by testing first bytes `EF BB BF`.
   - Parse each JSON file.
   - If the Obsidian console says `Unexpected token` followed by an invisible character at the start of JSON, repair BOM first.

3. Resolve `codex.exe`:
   - If the user supplied a path, verify `Test-Path` and run `codex.exe --version`.
   - If the user does not know the path, search likely locations under:
     - `%APPDATA%\npm\node_modules\@openai\codex`
     - `%APPDATA%\npm`
     - `%LOCALAPPDATA%`
     - the vault `.claudian\bin`
   - Prefer the real native executable ending in `...\vendor\x86_64-pc-windows-msvc\bin\codex.exe`.
   - Avoid paths containing mojibake or garbled Chinese characters.

4. Resolve and apply the active plugin ID:
   - If `community-plugins.json` already enables `claudian` or `realclaudian`, use that ID.
   - If neither is enabled and only one matching plugin folder exists, use that installed ID.
   - If both folders exist and neither is enabled, stop and report the ambiguous plugin state.
   - Write `.obsidian/community-plugins.json` to include `obsidian42-brat` plus the resolved plugin ID.

5. Configure BRAT without managing versions:
   - Keep `pluginList` as `["YishenTu/claudian"]`.
   - Do not set or infer a Claudian version unless the user explicitly asked for version pinning.
   - Set `updateAtStartup` and `updateThemesAtStartup` to `false` while stabilizing the vault.
   - Write as UTF-8 without BOM.

6. Configure Claudian/Codex:
   - Set `.claudian/claudian-settings.json`:
     - `settingsProvider`: `codex`
     - `model`: `gpt-5.5`
     - `lastCustomModel`: `gpt-5.5`
     - `permissionMode`: `yolo` unless the user requests safer behavior
     - `effortLevel`: `high`
     - `thinkingBudget`: `high`
     - `serviceTier`: `default`
     - `providerConfigs.codex.enabled`: `true`
     - `providerConfigs.codex.safeMode`: `workspace-write`
     - `providerConfigs.codex.cliPath`: resolved `codex.exe`
     - `providerConfigs.codex.customModels`: include `gpt-5.5`, `gpt-5.4`, `gpt-5.4-mini`, `gpt-5.3-codex`, `gpt-5.2`
     - `savedProviderModel.codex`: `gpt-5.5`
   - Preserve unrelated existing settings and environment snippets. Do not expose secrets.

7. Configure active plugin data:
   - Write `.obsidian/plugins/<plugin-id>/data.json` with a tab whose `draftModel` is `gpt-5.5`.

8. Validate:
   - All edited JSON files parse successfully.
   - All edited JSON files have `HasBom = False`.
   - The active plugin manifest ID matches the resolved plugin ID.
   - `community-plugins.json` enables the resolved plugin ID.
   - `codex.exe --version` works.

## Self-Contained PowerShell

Use this inline script for the repeatable configuration. Paste/run it as one PowerShell script after setting `$VaultPath`, `$CodexPath`, and `$Scenario`.

```powershell
param(
  [Parameter(Mandatory=$true)][string]$VaultPath,
  [Parameter(Mandatory=$true)][string]$CodexPath,
  [ValidateSet('new','upgrade')][string]$Scenario = 'upgrade'
)

$ErrorActionPreference = 'Stop'
$Utf8NoBom = [System.Text.UTF8Encoding]::new($false)

function Read-JsonFile($Path) {
  $text = [System.IO.File]::ReadAllText($Path).TrimStart([char]0xFEFF)
  if ([string]::IsNullOrWhiteSpace($text)) { return [pscustomobject]@{} }
  return $text | ConvertFrom-Json
}

function Write-JsonNoBom($Path, $Object, [int]$Depth = 100) {
  $dir = Split-Path -Parent $Path
  if ($dir -and -not (Test-Path -LiteralPath $dir)) {
    New-Item -ItemType Directory -Path $dir | Out-Null
  }
  $json = $Object | ConvertTo-Json -Depth $Depth
  [System.IO.File]::WriteAllText($Path, $json, $Utf8NoBom)
}

function Test-JsonNoBom($Path) {
  $bytes = [System.IO.File]::ReadAllBytes($Path)
  $hasBom = $bytes.Length -ge 3 -and $bytes[0] -eq 0xEF -and $bytes[1] -eq 0xBB -and $bytes[2] -eq 0xBF
  $jsonOk = $true
  try { $null = [System.IO.File]::ReadAllText($Path) | ConvertFrom-Json } catch { $jsonOk = $false }
  [pscustomobject]@{ Path = $Path; HasBom = $hasBom; JsonOk = $jsonOk }
}

function Find-CodexExe {
  $roots = @(
    (Join-Path $env:APPDATA 'npm\node_modules\@openai\codex'),
    (Join-Path $env:APPDATA 'npm'),
    $env:LOCALAPPDATA,
    (Join-Path $VaultPath '.claudian\bin')
  ) | Where-Object { $_ -and (Test-Path -LiteralPath $_) }

  foreach ($root in $roots) {
    $match = Get-ChildItem -LiteralPath $root -Recurse -Filter codex.exe -ErrorAction SilentlyContinue |
      Where-Object { $_.FullName -match '\\vendor\\x86_64-pc-windows-msvc\\bin\\codex\.exe$' } |
      Select-Object -First 1
    if ($match) { return $match.FullName }
  }

  foreach ($root in $roots) {
    $match = Get-ChildItem -LiteralPath $root -Recurse -Filter codex.exe -ErrorAction SilentlyContinue |
      Select-Object -First 1
    if ($match) { return $match.FullName }
  }

  throw 'codex.exe not found. Ask the user to install Codex or provide the exact path.'
}

function Resolve-ClaudianPluginId {
  $communityPath = Join-Path $VaultPath '.obsidian\community-plugins.json'
  $installed = @(@('realclaudian', 'claudian') | Where-Object {
    Test-Path -LiteralPath (Join-Path $VaultPath ".obsidian\plugins\$_\manifest.json")
  })

  if (Test-Path -LiteralPath $communityPath) {
    try {
      $enabled = Read-JsonFile $communityPath
      foreach ($id in @('realclaudian', 'claudian')) {
        if ($enabled -contains $id -and $installed -contains $id) { return $id }
      }
    } catch {
      # Encoding/JSON validation later will report details; continue with installed folders.
    }
  }

  if ($installed.Count -eq 1) { return $installed[0] }
  if ($installed.Count -eq 0) {
    throw 'No claudian or realclaudian plugin folder found. Install Claudian through BRAT first.'
  }

  throw 'Both claudian and realclaudian are installed, but neither is enabled in community-plugins.json. Enable the intended Claudian plugin in Obsidian or remove the inactive duplicate before running this configuration.'
}

if (-not (Test-Path -LiteralPath $VaultPath)) {
  throw "Vault path does not exist: $VaultPath"
}

$unknownChinese = [string]::Concat([char]0x4E0D, [char]0x77E5, [char]0x9053)
if ($CodexPath -eq 'unknown' -or $CodexPath -eq $unknownChinese) {
  $CodexPath = Find-CodexExe
}

if (-not (Test-Path -LiteralPath $CodexPath)) {
  throw "codex.exe path does not exist: $CodexPath"
}

$activeId = Resolve-ClaudianPluginId
$pluginDir = Join-Path $VaultPath ".obsidian\plugins\$activeId"
$manifestPath = Join-Path $pluginDir 'manifest.json'
if (-not (Test-Path -LiteralPath $manifestPath)) {
  throw "Selected plugin folder is missing: $pluginDir. Install that Claudian plugin through BRAT first."
}

$communityPath = Join-Path $VaultPath '.obsidian\community-plugins.json'
[System.IO.File]::WriteAllText($communityPath, "[`r`n  `"obsidian42-brat`",`r`n  `"$activeId`"`r`n]", $Utf8NoBom)

$bratPath = Join-Path $VaultPath '.obsidian\plugins\obsidian42-brat\data.json'
if (Test-Path -LiteralPath $bratPath) {
  $brat = Read-JsonFile $bratPath
} else {
  $brat = [pscustomobject]@{}
}
$brat | Add-Member -NotePropertyName pluginList -NotePropertyValue @('YishenTu/claudian') -Force
$brat | Add-Member -NotePropertyName updateAtStartup -NotePropertyValue $false -Force
$brat | Add-Member -NotePropertyName updateThemesAtStartup -NotePropertyValue $false -Force
$brat | Add-Member -NotePropertyName enableAfterInstall -NotePropertyValue $true -Force
Write-JsonNoBom $bratPath $brat

$settingsPath = Join-Path $VaultPath '.claudian\claudian-settings.json'
if (Test-Path -LiteralPath $settingsPath) {
  $settings = Read-JsonFile $settingsPath
} else {
  $settings = [pscustomobject]@{}
}

$settings | Add-Member -NotePropertyName settingsProvider -NotePropertyValue 'codex' -Force
$settings | Add-Member -NotePropertyName model -NotePropertyValue 'gpt-5.5' -Force
$settings | Add-Member -NotePropertyName lastCustomModel -NotePropertyValue 'gpt-5.5' -Force
$settings | Add-Member -NotePropertyName permissionMode -NotePropertyValue 'yolo' -Force
$settings | Add-Member -NotePropertyName effortLevel -NotePropertyValue 'high' -Force
$settings | Add-Member -NotePropertyName thinkingBudget -NotePropertyValue 'high' -Force
$settings | Add-Member -NotePropertyName serviceTier -NotePropertyValue 'default' -Force
if (-not $settings.providerConfigs) { $settings | Add-Member -NotePropertyName providerConfigs -NotePropertyValue ([pscustomobject]@{}) -Force }
if (-not $settings.providerConfigs.codex) { $settings.providerConfigs | Add-Member -NotePropertyName codex -NotePropertyValue ([pscustomobject]@{}) -Force }
$settings.providerConfigs.codex | Add-Member -NotePropertyName enabled -NotePropertyValue $true -Force
$settings.providerConfigs.codex | Add-Member -NotePropertyName safeMode -NotePropertyValue 'workspace-write' -Force
$settings.providerConfigs.codex | Add-Member -NotePropertyName cliPath -NotePropertyValue $CodexPath -Force
$settings.providerConfigs.codex | Add-Member -NotePropertyName customModels -NotePropertyValue "gpt-5.5`r`ngpt-5.4`r`ngpt-5.4-mini`r`ngpt-5.3-codex`r`ngpt-5.2" -Force
$settings.providerConfigs.codex | Add-Member -NotePropertyName reasoningSummary -NotePropertyValue 'detailed' -Force
$settings.providerConfigs.codex | Add-Member -NotePropertyName environmentVariables -NotePropertyValue '' -Force
$settings.providerConfigs.codex | Add-Member -NotePropertyName environmentHash -NotePropertyValue '' -Force
if (-not $settings.savedProviderModel) { $settings | Add-Member -NotePropertyName savedProviderModel -NotePropertyValue ([pscustomobject]@{}) -Force }
$settings.savedProviderModel | Add-Member -NotePropertyName codex -NotePropertyValue 'gpt-5.5' -Force
if (-not $settings.savedProviderEffort) { $settings | Add-Member -NotePropertyName savedProviderEffort -NotePropertyValue ([pscustomobject]@{}) -Force }
$settings.savedProviderEffort | Add-Member -NotePropertyName codex -NotePropertyValue 'high' -Force
if (-not $settings.savedProviderServiceTier) { $settings | Add-Member -NotePropertyName savedProviderServiceTier -NotePropertyValue ([pscustomobject]@{}) -Force }
$settings.savedProviderServiceTier | Add-Member -NotePropertyName codex -NotePropertyValue 'default' -Force
Write-JsonNoBom $settingsPath $settings

$pluginDataPath = Join-Path $pluginDir 'data.json'
$pluginData = [pscustomobject]@{
  tabManagerState = [pscustomobject]@{
    openTabs = @([pscustomobject]@{
      draftModel = 'gpt-5.5'
      tabId = 'tab-gpt55'
      conversationId = $null
    })
    activeTabId = 'tab-gpt55'
  }
}
Write-JsonNoBom $pluginDataPath $pluginData 30

$validationFiles = @($communityPath, $bratPath, $settingsPath, $pluginDataPath, $manifestPath)
$validation = $validationFiles | ForEach-Object { Test-JsonNoBom $_ }
$manifest = Read-JsonFile $manifestPath

[pscustomobject]@{
  Scenario = $Scenario
  ActivePluginId = $activeId
  ManifestPluginVersion = $manifest.version
  CodexPath = $CodexPath
  CodexPathExists = (Test-Path -LiteralPath $CodexPath)
  SettingsProvider = 'codex'
  Model = 'gpt-5.5'
  Validation = $validation
}
```

## Troubleshooting Signals

- `failed to read JSON ... Unexpected token` at the first character: JSON file likely has BOM. Rewrite UTF-8 without BOM.
- `Plugin failure: claudian ... JSON.parse` or `Plugin failure: realclaudian ... JSON.parse`: usually BOM in `.claudian/claudian-settings.json` or plugin `data.json`.
- Plugin exists but command is missing: check `community-plugins.json`; it may enable the wrong or inactive plugin ID.
- Codex fails or path is garbled: replace with verified native `codex.exe` path and run `--version`.
- Obsidian removes the plugin from enabled list after startup: inspect Developer Console for the first red error, then repair the failing JSON/config.

## Final Response

Report only the important facts:

- resolved plugin ID
- enabled plugins
- Codex path verified or not
- default provider/model
- BOM/JSON validation result
- whether the user needs to restart Obsidian
