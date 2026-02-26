# Tasks: List Microsoft Planner Tasks in PowerShell

**Branch**: `001-list-planner-tasks`
**Input**: Design documents from `specs/001-list-planner-tasks/`
**Prerequisites**: spec.md ✅ | plan.md (template only — technical context derived from conversation)

**Tests**: Not explicitly requested — no test tasks generated.

**Organization**: Tasks grouped by phase and user story to enable independent implementation.

## Format: `[ID] [P?] [Story?] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Maps to user story from spec.md (e.g., [US1])

---

## Phase 1: Setup (Module Scaffolding)

**Purpose**: Create the PowerShell module structure before any logic is written.

- [ ] T001 Create root module folder `PlannerTasks/` at repository root
- [ ] T002 [P] Create module manifest `PlannerTasks/PlannerTasks.psd1` with metadata (Author, Description, PowerShellVersion = '7.0', FunctionsToExport = @())
- [ ] T003 [P] Create root module loader `PlannerTasks/PlannerTasks.psm1` that dot-sources all files under `Public/` and `Private/` recursively
- [ ] T004 [P] Create folder structure: `PlannerTasks/Public/`, `PlannerTasks/Private/Auth/`, `PlannerTasks/Private/Api/`, `PlannerTasks/Private/Helpers/`

**Checkpoint**: `Import-Module ./PlannerTasks/PlannerTasks.psd1` loads without errors.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core private infrastructure that ALL public functions depend on. Must be complete before User Story phases begin.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [ ] T005 [P] Create `PlannerTasks/Private/Auth/Connect-MsGraphInteractive.ps1` — implements device-code OAuth2 flow against Azure AD (`https://login.microsoftonline.com/{tenantId}/oauth2/v2.0/devicecode` and token endpoint) using only `Invoke-RestMethod`; returns an access token string; accepts `-TenantId` and `-ClientId` parameters; prompts user to open browser and enter device code; outputs progress with `Write-Information`
- [ ] T006 [P] Create `PlannerTasks/Private/Api/Invoke-PlannerGraphRequest.ps1` — wraps `Invoke-RestMethod` for Microsoft Graph Planner API calls; accepts `-AccessToken`, `-Uri`, `-Method` parameters; handles HTTP errors and surfaces user-friendly messages via `Write-Error` with `-ErrorAction`; returns raw API response object
- [ ] T007 Create `PlannerTasks/Private/Helpers/ConvertTo-PlannerTaskObject.ps1` — converts a single raw Graph API task JSON object into a `[pscustomobject]` with properties: `Title` (string), `DueDateTime` ([datetime] or `$null` if absent), `Status` (string mapped from percentComplete: 0='NotStarted', 50='InProgress', 100='Completed'), `TaskId` (string), `PlanId` (string); add `PSTypeName = 'PlannerTask'` for display formatting
- [ ] T008 Create `PlannerTasks/Private/Helpers/Write-PlannerUserMessage.ps1` — helper that accepts `-Message` (string) and `-Type` ('Error', 'Warning', 'Information') and writes to the appropriate PowerShell stream (`Write-Error`, `Write-Warning`, `Write-Information`) with consistent prefix `[PlannerTasks]`

**Checkpoint**: All four private functions are importable and individually testable in isolation.

---

## Phase 3: User Story 1 — List tasks from a Planner plan (Priority: P1) 🎯 MVP

**Goal**: User can run `Get-PlannerTask` in the terminal and get back an array of `[pscustomobject]` Planner tasks from a specified plan.

**Independent Test**: Run `Get-PlannerTask` with a valid plan ID → objects returned with `Title`, `DueDateTime`, `Status` properties visible in terminal. Run with invalid ID → friendly error shown, no crash.

### Implementation for User Story 1

- [ ] T009 [US1] Create `PlannerTasks/Public/Get-PlannerTask.ps1` — skeleton with `[CmdletBinding()]`, `param()` block (see T010), and `begin`/`process`/`end` structure; add `#Requires -Version 7.0` at top
- [ ] T010 [US1] Add parameter block to `Get-PlannerTask.ps1`: `-PlanId` [string] (mandatory, prompt if omitted via `[Parameter(Mandatory)]`), `-TenantId` [string] (mandatory, prompt if omitted), `-ClientId` [string] (mandatory, prompt if omitted), `-Status` [ValidateSet('NotStarted','InProgress','Completed','All')] defaulting to 'All', `-DueAfter` [datetime] optional, `-DueBefore` [datetime] optional
- [ ] T011 [US1] Implement auth flow in `Get-PlannerTask.ps1` `begin` block: call `Connect-MsGraphInteractive -TenantId $TenantId -ClientId $ClientId`; store returned access token; if auth fails, call `Write-PlannerUserMessage -Message "Authentication failed. Check your TenantId and ClientId." -Type Error` and return
- [ ] T012 [US1] Implement Graph API call in `Get-PlannerTask.ps1` `process` block: call `Invoke-PlannerGraphRequest -AccessToken $token -Uri "https://graph.microsoft.com/v1.0/planner/plans/$PlanId/tasks" -Method GET`; if result is `$null` or response has no `value`, call `Write-PlannerUserMessage -Message "No tasks found for plan '$PlanId'." -Type Information` and return empty array
- [ ] T013 [US1] Implement object conversion in `Get-PlannerTask.ps1` `process` block: pipe each item in `$response.value` through `ConvertTo-PlannerTaskObject`; collect results into `[System.Collections.Generic.List[pscustomobject]]`
- [ ] T014 [US1] Implement client-side filtering in `Get-PlannerTask.ps1` `process` block: filter list by `-Status` (if not 'All'); filter by `-DueAfter` and `-DueBefore` if provided (skip items where `DueDateTime` is `$null` when date filters are active, and emit a `Write-Warning`); output filtered list to pipeline
- [ ] T015 [US1] Add edge-case handling in `Get-PlannerTask.ps1`: wrap Graph call in `try/catch`; on network error call `Write-PlannerUserMessage -Message "Network error contacting Microsoft Graph: $_" -Type Error`; on 403/404 HTTP error call `Write-PlannerUserMessage -Message "Plan '$PlanId' not found or access denied." -Type Error`
- [ ] T016 [US1] Update `PlannerTasks/PlannerTasks.psd1`: set `FunctionsToExport = @('Get-PlannerTask')`; verify `RootModule = 'PlannerTasks.psm1'`; set `RequiredAssemblies = @()` (no third-party assemblies required)

**Checkpoint**: `Import-Module ./PlannerTasks/PlannerTasks.psd1` then `Get-PlannerTask` prompts for PlanId, TenantId, ClientId; authenticates interactively; returns array of `[pscustomobject]` with `Title`, `DueDateTime`, `Status` properties; errors display as friendly messages.

---

## Phase 4: Polish & Cross-Cutting Concerns

**Purpose**: Documentation, help, and final validation.

- [ ] T017 [P] Add comment-based help to `PlannerTasks/Public/Get-PlannerTask.ps1`: `.SYNOPSIS`, `.DESCRIPTION`, `.PARAMETER` for each param, `.EXAMPLE` showing basic usage and filtered usage, `.OUTPUTS [pscustomobject[]]`
- [ ] T018 [P] Add comment-based help to `PlannerTasks/Private/Auth/Connect-MsGraphInteractive.ps1`: `.SYNOPSIS`, `.PARAMETER TenantId`, `.PARAMETER ClientId`, `.OUTPUTS [string] Access token`
- [ ] T019 Create `PlannerTasks/README.md` with: prerequisites (PowerShell 7+, Azure AD App Registration with Planner.Read.All scope), installation steps (`Import-Module`), quick-start usage examples for `Get-PlannerTask`, filtering examples, and output format description
- [ ] T020 Validate full module: run `Import-Module ./PlannerTasks/PlannerTasks.psd1 -Force` and `Get-Command -Module PlannerTasks` to confirm `Get-PlannerTask` is exported; run `Get-Help Get-PlannerTask -Full` to confirm help renders correctly

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — start immediately. T002, T003, T004 can run in parallel after T001.
- **Foundational (Phase 2)**: Depends on Phase 1 completion. T005, T006, T007, T008 can all run in parallel.
- **User Story 1 (Phase 3)**: Depends on Phase 2 completion. Tasks within Phase 3 are sequential (T009 → T010 → T011 → T012 → T013 → T014 → T015 → T016).
- **Polish (Phase 4)**: Depends on Phase 3 completion. T017, T018, T019 can run in parallel. T020 runs last.

### Within User Story 1

```
T009 (skeleton) → T010 (params) → T011 (auth) → T012 (API call)
                                                → T013 (conversion)
                                                → T014 (filtering)
                                                → T015 (error handling)
                                                → T016 (manifest update)
```

### Parallel Opportunities

```powershell
# Phase 1 parallel (after T001):
Task: "T002 - Create PlannerTasks.psd1"
Task: "T003 - Create PlannerTasks.psm1"
Task: "T004 - Create folder structure"

# Phase 2 parallel (all four at once):
Task: "T005 - Connect-MsGraphInteractive.ps1"
Task: "T006 - Invoke-PlannerGraphRequest.ps1"
Task: "T007 - ConvertTo-PlannerTaskObject.ps1"
Task: "T008 - Write-PlannerUserMessage.ps1"

# Phase 4 parallel (after T016):
Task: "T017 - Help for Get-PlannerTask.ps1"
Task: "T018 - Help for Connect-MsGraphInteractive.ps1"
Task: "T019 - Create README.md"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Module scaffolding
2. Complete Phase 2: Private infrastructure (auth, API, helpers)
3. Complete Phase 3: `Get-PlannerTask` public function
4. **STOP and VALIDATE**: Run `Get-PlannerTask` against a real Planner plan
5. Ship as usable module

### File Structure (Final)

```text
PlannerTasks/
├── PlannerTasks.psd1                          # Module manifest
├── PlannerTasks.psm1                          # Root loader (dot-sources Public/ + Private/)
├── README.md                                  # Usage docs
├── Public/
│   └── Get-PlannerTask.ps1                    # Exported function
└── Private/
    ├── Auth/
    │   └── Connect-MsGraphInteractive.ps1     # OAuth2 device-code flow
    ├── Api/
    │   └── Invoke-PlannerGraphRequest.ps1     # Graph API wrapper
    └── Helpers/
        ├── ConvertTo-PlannerTaskObject.ps1    # JSON → [pscustomobject]
        └── Write-PlannerUserMessage.ps1       # Consistent user messaging
```

---

## Notes

- All auth uses pure `Invoke-RestMethod` against Azure AD endpoints — no MSAL or Microsoft.Graph module required
- `[pscustomobject]` output is pipeline-friendly: users can pipe to `Where-Object`, `Sort-Object`, `Format-Table`, `Export-Csv`, etc.
- `PSTypeName = 'PlannerTask'` on output objects enables future `.ps1xml` formatting without breaking changes
- Interactive prompting via `[Parameter(Mandatory)]` is native PowerShell — no extra code needed
- No storage, no caching, no secrets stored — access token lives only in session memory
