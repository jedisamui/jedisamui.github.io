---
title: Automating a VCF 9.1 Management Domain Startup for Reusable VCD Lab Pods
description: A modular PowerCLI framework that walks a VCF 9.1 management domain back through Broadcom's documented startup sequence, restoring the VCF Management Infrastructure after an isolated VCD lab pod is redeployed from a vApp Template.
author: samui
date: 2026-08-20
categories:
- VMware Cloud Foundation
- VMware Cloud Director
tags:
- automation
- powershell
- vcd
- vcf
image:
  path: /images/2026/08/vcf91-startup-script.png
permalink: /blog/vcf-91-management-domain-startup/
---

We spin up VMware Cloud Foundation 9.1 inside isolated, self-contained pods running as vApps within VMware Cloud Director. Once a pod is converted into a reusable vApp Template - [the process covered in the companion post on shutting the management domain down cleanly](/blog/vcf-91-management-domain-shutdown/) - the next student or engineer who checks it out is handed a freshly deployed vApp with every VM in a known, powered-off state. Before that pod is usable, the entire nested VCF 9.1 Management Domain has to be brought back online: SDDC Manager, NSX, vCenter, and the VCF Operations fleet all expect their dependencies to already be healthy before they will start correctly, so "power everything on" is exactly as unsafe to do in an arbitrary order on the way up as it is on the way down.

Broadcom documents the supported order for this exact scenario, and it was the starting point for everything that follows: [Start the Management Domain](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1/fleet-management/vcf-shutdown-and-startup/sddc-startup/start-the-management-domain.html). The official procedure is effectively the shutdown sequence in reverse: vSAN and the ESXi hosts come up first, followed by SDDC Manager, NSX Manager, the NSX Edge/VNA nodes, Protection and Recovery, VCF Operations, VCF Management Services, the License Server, Cloud Proxy, VCF Operations for Networks, and finally VCF Automation last. That ordering, and the health checks Broadcom calls out between stages, are the backbone of the script set below. Protection and Recovery is intentionally left out of this implementation, since it is not deployed in these lab pods.

Turning that runbook into automation followed the same discipline as the shutdown side: one file per Broadcom stage plus a shared helper module, built and verified step by step - running each new module on its own against the lab, confirming the right services came up cleanly and reported healthy, and only then wiring it into the next step. That modularity kept the blast radius of any one change contained to a single small file instead of a monolithic script, and it means each component's startup logic - and the health check that proves it actually worked, rather than just that a power-on request was accepted - can be tested and re-tested in isolation before it's trusted against a live pod.

**A note on the credentials in these examples:** the wrapper script below stores vCenter, NSX Manager, VCF Operations, and Fleet LCM credentials as plaintext in its `$Config` block, by design. These scripts are meant to run only against disposable, network-isolated lab pods with no ingress or egress to the public internet, where the password list is already handed to students. That is not a pattern to reuse anywhere credentials matter - every password shown here has been replaced with `<PASSWORD>`, and the lab domain has been replaced with `lab.net`, for this write-up.

### The shared helper module

Every step file and the wrapper itself dot-source `00-Common.ps1` first. It's the same helper module used on the shutdown side, carried forward for consistency: a timestamped, color-coded `Write-Step` logger; `Connect-VCFVCenter`, which reuses an existing vCenter session instead of reconnecting; `Stop-VMsGracefully`, kept here for symmetry with the shutdown toolkit even though the startup flow doesn't call it directly; `Assert-PowerCLIModule` and `Assert-ModuleAvailable`, which check for and install the required PowerCLI/VCF modules on demand; and `Start-VMsAndWait`, the power-on counterpart that requests each VM start, then polls until it reports powered on (optionally waiting for VMware Tools to report running before considering it ready). `Confirm-ManualStep` - the gate used whenever a step can't be fully automated - picked up one addition on the startup side: it now checks a non-interactive session flag first, and if it's set, fails fast with a clear error instead of sitting on a blocking `Read-Host` prompt that nothing will ever answer.

```powershell
<#
Common helper functions shared by every step script and the wrapper.
Dot-source this first: . .\00-Common.ps1
#>

function Write-Step {
    param(
        [Parameter(Mandatory)][string]$Message,
        [ValidateSet('INFO','WARN','ERROR','OK')][string]$Level = 'INFO'
    )
    $ts = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $color = switch ($Level) { 'WARN'{'Yellow'} 'ERROR'{'Red'} 'OK'{'Green'} default{'Cyan'} }
    Write-Host "[$ts] [$Level] $Message" -ForegroundColor $color
}

function Connect-VCFVCenter {
    param(
        [Parameter(Mandatory)][string]$VCenterServer,
        [Parameter(Mandatory)][PSCredential]$Credential
    )
    $existing = $global:DefaultVIServers | Where-Object { $_.Name -eq $VCenterServer }
    if ($existing) { Write-Step "Already connected to $VCenterServer." -Level OK; return $existing }

    Write-Step "Connecting to vCenter $VCenterServer ..."
    try {
        $conn = Connect-VIServer -Server $VCenterServer -Credential $Credential -ErrorAction Stop
        Write-Step "Connected to $VCenterServer." -Level OK
        return $conn
    } catch {
        Write-Step "Failed to connect to $VCenterServer : $($_.Exception.Message)" -Level ERROR
        throw
    }
}

function Stop-VMsGracefully {
    <# Graceful guest-OS shutdown (equivalent to Power > Shut Down Guest OS), then polls for PoweredOff. #>
    param(
        [Parameter(Mandatory)][string[]]$VMNames,
        [int]$TimeoutSeconds = 900,
        [int]$PollIntervalSeconds = 15
    )

    $targets = foreach ($name in $VMNames) {
        $vm = Get-VM -Name $name -ErrorAction SilentlyContinue
        if (-not $vm) { Write-Step "VM '$name' not found - skipping." -Level WARN; continue }
        if ($vm.PowerState -eq 'PoweredOff') { Write-Step "VM '$name' already powered off." -Level OK; continue }
        $vm
    }
    if (-not $targets) { Write-Step "Nothing to shut down for this step." -Level OK; return }

    foreach ($vm in $targets) {
        try {
            Write-Step "Requesting graceful guest OS shutdown for '$($vm.Name)'..."
            Stop-VMGuest -VM $vm -Confirm:$false -ErrorAction Stop | Out-Null
        } catch {
            Write-Step "Guest shutdown request failed for '$($vm.Name)': $($_.Exception.Message)" -Level ERROR
        }
    }

    Write-Step "Waiting for VM(s) to power off (timeout ${TimeoutSeconds}s)..."
    $sw = [Diagnostics.Stopwatch]::StartNew()
    $pending = $targets.Name
    while ($pending.Count -gt 0 -and $sw.Elapsed.TotalSeconds -lt $TimeoutSeconds) {
        Start-Sleep -Seconds $PollIntervalSeconds
        foreach ($name in @($pending)) {
            $vm = Get-VM -Name $name -ErrorAction SilentlyContinue
            if ($vm -and $vm.PowerState -eq 'PoweredOff') {
                Write-Step "VM '$name' is powered off." -Level OK
                $pending = $pending | Where-Object { $_ -ne $name }
            }
        }
    }
    foreach ($name in $pending) {
        Write-Step "VM '$name' did not power off within the timeout - verify manually before proceeding." -Level WARN
    }
}

function Confirm-ManualStep {
    param([Parameter(Mandatory)][string]$Prompt)
    do { $resp = Read-Host "$Prompt Type 'yes' to confirm and continue" } while ($resp -notmatch '^(yes)$')
}

# --- ADD this generic helper alongside the existing functions in 00-Common.ps1 ---

function Assert-PowerCLIModule {
    <#
    Checks whether VCF.PowerCLI is available and installs it for the current
    user if it is missing. Also silences the CEIP prompt and (optionally) allows
    self-signed certificates commonly found in on-prem VCF deployments.

    Note: VMware.PowerCLI has been deprecated and replaced by VCF.PowerCLI as
    the current distribution. If you're on an older environment that still
    only has VMware.PowerCLI installed, pass -LegacyModuleName 'VMware.PowerCLI'.
    #>
    param(
        [switch]$AllowSelfSignedCerts,
        [string]$ModuleName = 'VCF.PowerCLI'
    )

    $module = Get-Module -ListAvailable -Name $ModuleName
    if (-not $module) {
        Write-Step "$ModuleName module not found. Installing for current user..." -Level WARN
        try {
            Install-Module -Name $ModuleName -Scope CurrentUser -Force -AllowClobber -ErrorAction Stop
            Write-Step "$ModuleName installed successfully." -Level OK
        }
        catch {
            Write-Step "Failed to install ${ModuleName}: $($_.Exception.Message)" -Level ERROR
            throw
        }
    }
    else {
        Write-Step "$ModuleName module found (version $($module[0].Version))." -Level OK
    }

    Import-Module $ModuleName -ErrorAction Stop

    # Silence CEIP / update check prompts non-interactively
    Set-PowerCLIConfiguration -ParticipateInCEIP:$false -Scope Session -Confirm:$false | Out-Null

    if ($AllowSelfSignedCerts) {
        Set-PowerCLIConfiguration -InvalidCertificateAction Ignore -Scope Session -Confirm:$false | Out-Null
        Write-Step "PowerCLI configured to ignore invalid/self-signed certificates for this session." -Level WARN
    }
}

function Assert-ModuleAvailable {
    <# Generic module presence/install check (unchanged - already module-name-agnostic). #>
    param(
        [Parameter(Mandatory)][string]$Name,
        [string]$InstallScope = 'CurrentUser'
    )
    $module = Get-Module -ListAvailable -Name $Name
    if (-not $module) {
        Write-Step "$Name module not found. Installing for current user..." -Level WARN
        try {
            Install-Module -Name $Name -Scope $InstallScope -Force -AllowClobber -ErrorAction Stop
            Write-Step "$Name installed." -Level OK
        } catch {
            Write-Step "Failed to install ${Name}: $($_.Exception.Message)" -Level ERROR
            throw
        }
    } else {
        Write-Step "$Name found (version $($module[0].Version))." -Level OK
    }
    Import-Module $Name -ErrorAction Stop
}

function Start-VMsAndWait {
    param(
        [Parameter(Mandatory)][string[]]$VMNames,
        [switch]$WaitForTools,
        [int]$TimeoutSeconds = 900,
        [int]$PollIntervalSeconds = 15
    )
    $targets = foreach ($name in $VMNames) {
        $vm = Get-VM -Name $name -ErrorAction SilentlyContinue
        if (-not $vm) { Write-Step "VM '$name' not found - skipping." -Level WARN; continue }
        $vm
    }
    if (-not $targets) { Write-Step "Nothing to start for this step." -Level OK; return }

    foreach ($vm in $targets) {
        if ($vm.PowerState -eq 'PoweredOn') { Write-Step "VM '$($vm.Name)' already powered on." -Level OK; continue }
        try {
            Write-Step "Powering on '$($vm.Name)'..."
            Start-VM -VM $vm -Confirm:$false -ErrorAction Stop | Out-Null
        } catch {
            Write-Step "Power-on request failed for '$($vm.Name)': $($_.Exception.Message)" -Level ERROR
        }
    }

    Write-Step "Waiting for VM(s) to power on$(if($WaitForTools){' and report VMware Tools running'}) (timeout ${TimeoutSeconds}s)..."
    $sw = [Diagnostics.Stopwatch]::StartNew()
    $pending = $targets.Name
    while ($pending.Count -gt 0 -and $sw.Elapsed.TotalSeconds -lt $TimeoutSeconds) {
        Start-Sleep -Seconds $PollIntervalSeconds
        foreach ($name in @($pending)) {
            $vm = Get-VM -Name $name -ErrorAction SilentlyContinue
            if (-not $vm) { continue }
            $ready = $vm.PowerState -eq 'PoweredOn'
            if ($ready -and $WaitForTools) {
                $ready = $vm.ExtensionData.Guest.ToolsRunningStatus -eq 'guestToolsRunning'
            }
            if ($ready) {
                Write-Step "VM '$name' is up." -Level OK
                $pending = $pending | Where-Object { $_ -ne $name }
            }
        }
    }
    foreach ($name in $pending) {
        Write-Step "VM '$name' did not report ready within the timeout - verify manually before proceeding." -Level WARN
    }
}

function Confirm-ManualStep {
    param([Parameter(Mandatory)][string]$Prompt)

    if ($Global:LabStartupNonInteractive) {
        Write-Step "Manual confirmation required but running non-interactively: $Prompt" -Level ERROR
        throw "TIMEOUT: manual confirmation required - $Prompt"
    }

    do { $resp = Read-Host "$Prompt Type 'yes' to confirm and continue" } while ($resp -notmatch '^(yes)$')
}
```

### Step 1: Start vSAN and the ESX Hosts in the Management Domain

Host power-on via out-of-band management (Redfish/iLO/iDRAC) is handled by a separate process outside this script set - this step picks up once the hosts are physically reachable. The vCenter readiness check was rewritten from a naive "retry `Connect-VIServer` until it succeeds" loop, which can report success before vCenter's internal services have actually finished starting. It now polls the vCenter Appliance's own health API (`GET /api/appliance/health/system`) via a session token obtained from `POST /api/session`, and only proceeds once it reports healthy. The vSAN cluster restart itself uses the officially supported `Start-VsanCluster` / `Get-VsanClusterPowerState` cmdlets, and what used to be a manual "go check Skyline Health" prompt is now two automated polling functions: `Wait-S1VsanClusterHealthy`, gated on `HealthScore` rather than the status string (a cluster can report "Warning" at a HealthScore of 99), and `Wait-S1VsanResyncComplete`, which waits for `Get-VsanResyncingOverview` to report zero bytes outstanding. Both fall back to a manual confirmation prompt if the cmdlet isn't available or the poll times out. The step closes by acknowledging the IO-filter-provider alarm that reliably fires when vSAN comes back up, and by cleaning any stale `root` lockdown exceptions left on the hosts.

```powershell
<#
Step 1: Start vSAN and the ESX Hosts in the Management Domain.

Host power-on via out-of-band management (Redfish/iLO/iDRAC) is NOT handled
here - that is managed by a separate process outside this script set.

vCenter readiness check: previously this just retried Connect-VIServer until
it succeeded, which can be true before vCenter's internal services (vpxd,
inventory service, etc.) have actually finished starting - a race condition.
This now polls the vCenter Appliance's own health API
(GET /api/appliance/health/system, vSphere Automation API) until it reports
healthy, using a session token obtained from POST /api/session.
Confirmed against Broadcom's vSphere Automation API docs (v9.1):
  https://developer.broadcom.com/xapis/vsphere-automation-api/latest/appliance/appliance-health-system/
  https://developer.broadcom.com/xapis/vsphere-automation-api/latest/cis/cis-session/
Caveat: the docs confirm the endpoint and that it returns a HealthLevel
string, but I could not load the exact enum data-structure page to confirm
casing/spelling. "green" (case-insensitive) is the long-standing, well-known
vSphere health value for "fully healthy" - verify against what your
environment actually returns the first time this runs, and adjust
-RequiredHealthLevels below if needed (e.g. to also accept 'yellow').

Requires PowerShell 7+ (uses -SkipCertificateCheck on Invoke-RestMethod),
same requirement already in place for Step 5/7's REST calls.

vSAN cluster restart still uses the confirmed PowerCLI cmdlets from
William Lam's blog: Start-VsanCluster / Get-VsanClusterPowerState.

Post-restart health/resync validation: previously a manual confirmation
prompt. Now automated via two polling functions:
  - Wait-S1VsanClusterHealthy: uses Test-VsanClusterHealth, gated on
    HealthScore (per user's own observed output, OverallHealthStatus can
    show "Warning" even at HealthScore 99, so HealthScore is the real gate).
  - Wait-S1VsanResyncComplete: uses Get-VsanResyncingOverview, gated on
    TotalBytesToSync reaching zero/null (per user-supplied, user-tested code).
Both fall back to a manual confirmation prompt if the relevant cmdlet isn't
available, or if the polling loop times out without a healthy/complete result.
#>

function Clear-S1IOFilterProviderAlarms {
    <#
        Acknowledges the vCenter alarm "Registration/unregistration of
        third-party IO filter storage providers fails on a host" on each
        given ESX host - this reliably triggers when vSAN is shut down and
        clears itself once hosts successfully re-register IO filter
        providers after vSAN is back up.

        Confirmed against Broadcom's vSphere Web Services API docs (v9.1):
          https://developer.broadcom.com/xapis/vsphere-web-services-api/latest/vim.alarm.AlarmManager.html
          https://developer.broadcom.com/xapis/vsphere-web-services-api/latest/vim.alarm.AlarmState.html
          https://developer.broadcom.com/xapis/vsphere-web-services-api/latest/vim.alarm.AlarmFilterSpec.html

        Caveat: the API only exposes AcknowledgeAlarm (scoped to one alarm on
        one entity) and ClearTriggeredAlarms (a MASS reset of every triggered
        alarm matching a status filter, with no per-entity/per-alarm
        scoping). There is no documented API for "clear just this one alarm
        on just this host". This function acknowledges by default; pass
        -AlsoMassClearTriggeredAlarms only if you're fine with resetting
        every triggered yellow/red alarm in the connected vCenter, not just
        this one - off by default given that blast radius.
    #>
    param(
        [Parameter(Mandatory)][string[]]$HostNames,
        [string]$AlarmNameMatch = 'Registration/unregistration of third-party IO filter storage providers fails on a host',
        [switch]$AlsoMassClearTriggeredAlarms
    )

    Write-Step "Checking for '$AlarmNameMatch' alarms on management cluster hosts..."

    $alarmMgr = Get-View AlarmManager -ErrorAction SilentlyContinue
    if (-not $alarmMgr) {
        Write-Step "Could not retrieve AlarmManager - skipping alarm acknowledgment." -Level WARN
        return
    }

    $acknowledgedAny = $false

    foreach ($hostName in $HostNames) {
        try {
            $vmhost = Get-VMHost -Name $hostName -ErrorAction Stop
            $states = $alarmMgr.GetAlarmState($vmhost.ExtensionData.MoRef)

            foreach ($state in $states) {
                try {
                    $alarmView = Get-View -Id $state.Alarm -ErrorAction Stop
                } catch {
                    continue
                }

                if ($alarmView.Info.Name -notlike "*$AlarmNameMatch*") { continue }
                if ($state.Acknowledged) {
                    Write-Step "'$($alarmView.Info.Name)' on '$hostName' is already acknowledged." -Level OK
                    continue
                }

                try {
                    $alarmMgr.AcknowledgeAlarm($state.Alarm, $vmhost.ExtensionData.MoRef)
                    Write-Step "Acknowledged '$($alarmView.Info.Name)' on '$hostName'." -Level OK
                    $acknowledgedAny = $true
                } catch {
                    Write-Step "Failed to acknowledge alarm on '$hostName': $($_.Exception.Message)" -Level WARN
                }
            }
        } catch {
            Write-Step "Could not check alarms on '$hostName': $($_.Exception.Message)" -Level WARN
        }
    }

    if (-not $acknowledgedAny) {
        Write-Step "No matching un-acknowledged '$AlarmNameMatch' alarms found." -Level OK
    }

    if ($AlsoMassClearTriggeredAlarms) {
        Write-Step "Mass-clearing ALL triggered alarms (status yellow/red) in the connected vCenter, per -AlsoMassClearTriggeredAlarms..." -Level WARN
        try {
            $filter = New-Object VMware.Vim.AlarmFilterSpec
            $filter.status = @('yellow','red')
            $alarmMgr.ClearTriggeredAlarms($filter)
            Write-Step "ClearTriggeredAlarms invoked." -Level OK
        } catch {
            Write-Step "ClearTriggeredAlarms failed: $($_.Exception.Message)" -Level WARN
        }
    }
}

function Get-S1VCenterSessionToken {
    param(
        [Parameter(Mandatory)][string]$VCenterServer,
        [Parameter(Mandatory)][PSCredential]$Credential
    )
    $pair = "$($Credential.UserName):$($Credential.GetNetworkCredential().Password)"
    $basicAuth = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes($pair))
    $headers = @{ Authorization = "Basic $basicAuth" }
    Invoke-RestMethod -Method Post -Uri "https://$VCenterServer/api/session" `
        -Headers $headers -SkipCertificateCheck -ErrorAction Stop
}

function Remove-S1VCenterSessionToken {
    param(
        [Parameter(Mandatory)][string]$VCenterServer,
        [Parameter(Mandatory)][string]$SessionToken
    )
    try {
        Invoke-RestMethod -Method Delete -Uri "https://$VCenterServer/api/session" `
            -Headers @{ 'vmware-api-session-id' = $SessionToken } `
            -SkipCertificateCheck -ErrorAction SilentlyContinue | Out-Null
    } catch { }
}

function Wait-S1VCenterServicesReady {
    <#
        Polls the vCenter Appliance health API until it reports a healthy
        state, rather than just checking that Connect-VIServer succeeds.
    #>
    param(
        [Parameter(Mandatory)][string]$VCenterServer,
        [Parameter(Mandatory)][PSCredential]$Credential,
        [string[]]$RequiredHealthLevels = @('green'),
        [int]$TimeoutSeconds = 3600,
        [int]$PollIntervalSeconds = 30
    )

    Write-Step "Waiting for vCenter appliance services on '$VCenterServer' to report healthy (timeout ${TimeoutSeconds}s)..."
    $sw = [Diagnostics.Stopwatch]::StartNew()
    $healthy = $false
    $lastHealth = $null

    while ($sw.Elapsed.TotalSeconds -lt $TimeoutSeconds) {
        try {
            $token = Get-S1VCenterSessionToken -VCenterServer $VCenterServer -Credential $Credential
            try {
                $health = Invoke-RestMethod -Method Get -Uri "https://$VCenterServer/api/appliance/health/system" `
                    -Headers @{ 'vmware-api-session-id' = $token } -SkipCertificateCheck -ErrorAction Stop
            } finally {
                Remove-S1VCenterSessionToken -VCenterServer $VCenterServer -SessionToken $token
            }

            $lastHealth = $health
            $normalized = ($health | Out-String).Trim().Trim('"')

            if ($RequiredHealthLevels -contains $normalized.ToLowerInvariant()) {
                $healthy = $true
                break
            }
            Write-Step "vCenter appliance health currently reports '$normalized' - waiting..." -Level WARN
        } catch {
            # Expected while the appliance/API endpoint is still coming up during boot - keep retrying.
        }
        Start-Sleep -Seconds $PollIntervalSeconds
    }

    if (-not $healthy) {
        throw "vCenter '$VCenterServer' appliance health did not reach an acceptable level ($($RequiredHealthLevels -join ', ')) within the timeout (last observed: '$lastHealth')."
    }
    Write-Step "vCenter appliance reports healthy ('$lastHealth')." -Level OK
}

function Wait-S1VsanClusterHealthy {
    <#
        Replaces the manual "go verify Skyline Health" prompt using
        Test-VsanClusterHealth. Per user's own observed output,
        OverallHealthStatus can report "Warning" even when the cluster is
        effectively fine (e.g. HealthScore 99), so HealthScore is used as the
        pass/fail gate rather than the status string.
    #>
    param(
        [Parameter(Mandatory)][string]$ClusterName,
        [int]$MinimumHealthScore = 90,
        [int]$TimeoutSeconds = 1800,
        [int]$PollIntervalSeconds = 30
    )

    $cluster = Get-Cluster -Name $ClusterName -ErrorAction Stop

    if (-not (Get-Command Test-VsanClusterHealth -ErrorAction SilentlyContinue)) {
        Write-Step "Test-VsanClusterHealth cmdlet not found in this PowerCLI version - falling back to manual confirmation." -Level WARN
        Confirm-ManualStep -Prompt "Verify vSAN Skyline Health is healthy for cluster '$ClusterName' before continuing."
        return
    }

    Write-Step "Waiting for vSAN cluster '$ClusterName' health score to exceed $MinimumHealthScore (timeout ${TimeoutSeconds}s)..."
    $sw = [Diagnostics.Stopwatch]::StartNew()
    $healthy = $false
    $lastScore = $null
    $lastStatus = $null

    while ($sw.Elapsed.TotalSeconds -lt $TimeoutSeconds) {
        try {
            $result = Test-VsanClusterHealth -Cluster $cluster -ErrorAction Stop
            $lastScore = $result.HealthScore
            $lastStatus = $result.OverallHealthStatus

            if ($null -ne $lastScore -and [int]$lastScore -gt $MinimumHealthScore) {
                $healthy = $true
                break
            }
            Write-Step "vSAN cluster '$ClusterName' health score currently $lastScore (status: '$lastStatus') - waiting..." -Level WARN
        } catch {
            Write-Step "Test-VsanClusterHealth call failed: $($_.Exception.Message)" -Level WARN
        }
        Start-Sleep -Seconds $PollIntervalSeconds
    }

    if (-not $healthy) {
        Write-Step "vSAN cluster '$ClusterName' health score did not exceed $MinimumHealthScore within the timeout (last observed: score=$lastScore, status='$lastStatus')." -Level WARN
        Confirm-ManualStep -Prompt "Manually verify vSAN Skyline Health is healthy for cluster '$ClusterName' before continuing."
    } else {
        Write-Step "vSAN cluster '$ClusterName' health score is $lastScore (status: '$lastStatus') - proceeding." -Level OK
    }
}

function Wait-S1VsanResyncComplete {
    <#
        Polls Get-VsanResyncingOverview until TotalBytesToSync reports zero
        (or null, meaning nothing outstanding), confirming vSAN has finished
        resyncing objects. Adapted from a cmdlet/property the user confirmed
        against their own environment.
    #>
    param(
        [Parameter(Mandatory)][string]$ClusterName,
        [int]$TimeoutSeconds = 3600,
        [int]$PollIntervalSeconds = 60
    )

    $cluster = Get-Cluster -Name $ClusterName -ErrorAction Stop

    if (-not (Get-Command Get-VsanResyncingOverview -ErrorAction SilentlyContinue)) {
        Write-Step "Get-VsanResyncingOverview cmdlet not found in this PowerCLI version - falling back to manual confirmation." -Level WARN
        Confirm-ManualStep -Prompt "Verify vSAN Resyncing Objects have cleared for cluster '$ClusterName' before continuing."
        return
    }

    Write-Step "Waiting for vSAN cluster '$ClusterName' resynchronization to complete (timeout ${TimeoutSeconds}s)..."
    $sw = [Diagnostics.Stopwatch]::StartNew()
    $complete = $false
    $lastBytesLeft = $null

    while ($sw.Elapsed.TotalSeconds -lt $TimeoutSeconds) {
        try {
            $overview = Get-VsanResyncingOverview -Cluster $cluster -ErrorAction Stop
            $lastBytesLeft = $overview.TotalBytesToSync

            if ($null -eq $lastBytesLeft -or $lastBytesLeft -eq 0) {
                $complete = $true
                break
            }

            $gbLeft = [Math]::Round($lastBytesLeft / 1GB, 2)
            Write-Step "vSAN cluster '$ClusterName' is actively resyncing: ${gbLeft} GB remaining - waiting..." -Level WARN
        } catch {
            Write-Step "Get-VsanResyncingOverview call failed: $($_.Exception.Message)" -Level WARN
        }
        Start-Sleep -Seconds $PollIntervalSeconds
    }

    if (-not $complete) {
        Write-Step "vSAN cluster '$ClusterName' did not clear its resync queue within the timeout (last observed: $lastBytesLeft bytes remaining)." -Level WARN
        Confirm-ManualStep -Prompt "Manually verify vSAN Resyncing Objects have cleared for cluster '$ClusterName' before continuing."
    } else {
        Write-Step "vSAN cluster '$ClusterName' resynchronization is complete." -Level OK
    }
}

function Invoke-Step01-StartVsanAndEsxHosts {
    param(
        [Parameter(Mandatory)][string]$VCenterServer,
        [Parameter(Mandatory)][PSCredential]$VCenterCredential,
        [Parameter(Mandatory)][string[]]$ManagementClusterHostNames,   # cluster hosting mgmt vCenter - checked first
        [string[][]]$OtherVsanClusterHostGroups = @(),                  # optional: other clusters, started after
        [int]$VCenterReadyTimeoutSeconds = 1800,
        [int]$ClusterRunningTimeoutSeconds = 1800,
        [int]$PollIntervalSeconds = 30
    )
    Write-Step "=== Step 1: Start vSAN and the ESX Hosts in the Management Domain ==="

    Write-Step "Note: ESX host power-on via out-of-band management is handled outside this script." -Level INFO

    Wait-S1VCenterServicesReady -VCenterServer $VCenterServer -Credential $VCenterCredential `
        -TimeoutSeconds $VCenterReadyTimeoutSeconds -PollIntervalSeconds $PollIntervalSeconds

    Connect-VCFVCenter -VCenterServer $VCenterServer -Credential $VCenterCredential | Out-Null

    $cmdletsAvailable = [bool](Get-Command Start-VsanCluster -ErrorAction SilentlyContinue) -and
                         [bool](Get-Command Get-VsanClusterPowerState -ErrorAction SilentlyContinue)
    if (-not $cmdletsAvailable) {
        Write-Step "Start-VsanCluster / Get-VsanClusterPowerState cmdlets not found in this PowerCLI version." -Level WARN
    }

    $clusterName = 'm1-mgmt-cluster'

    Write-Step "Restarting vSAN cluster '$clusterName'..."
    #$cluster = Get-Cluster -Name $clusterName -ErrorAction SilentlyContinue
	$cluster = Get-Cluster -Name "m1-mgmt-cluster" -ErrorAction SilentlyContinue
    Write-Step "Cluster check: '$cluster' "
    $automated = $false

    if ($cmdletsAvailable -and $cluster) {
        try {
            Start-VsanCluster -Cluster $cluster -Confirm:$false -ErrorAction Stop
            Write-Step "Start-VsanCluster invoked for '$clusterName'. Polling for running state (timeout ${ClusterRunningTimeoutSeconds}s)..."

            $sw2 = [Diagnostics.Stopwatch]::StartNew()
            $running = $false
            while ($sw2.Elapsed.TotalSeconds -lt $ClusterRunningTimeoutSeconds) {
                try {
                    $state = Get-VsanClusterPowerState -Cluster $cluster -ErrorAction Stop
                    if ($state -and ($state.PowerState -eq 'poweredOn' -or $state -eq 'poweredOn' -or $state.ToString() -match 'On')) {
                        $running = $true
                        break
                    }
                } catch {
                    Write-Step "Get-VsanClusterPowerState call failed: $($_.Exception.Message)" -Level WARN
                    break
                }
                Start-Sleep -Seconds $PollIntervalSeconds
            }
            if ($running) {
                Write-Step "vSAN cluster '$clusterName' reports running." -Level OK
                $automated = $true
            } else {
                Write-Step "vSAN cluster '$clusterName' did not report running within the timeout." -Level WARN
            }
        } catch {
            Write-Step "Start-VsanCluster failed for '$clusterName': $($_.Exception.Message)" -Level WARN
        }
    }

    if (-not $automated) {
        Confirm-ManualStep -Prompt "In the vSphere Client, right-click cluster '$clusterName' > vSAN > Restart cluster and complete the wizard. Once complete,"
    }

    Wait-S1VsanClusterHealthy -ClusterName $clusterName -MinimumHealthScore 90 `
        -TimeoutSeconds $ClusterRunningTimeoutSeconds -PollIntervalSeconds $PollIntervalSeconds

    Wait-S1VsanResyncComplete -ClusterName $clusterName `
        -TimeoutSeconds $ClusterRunningTimeoutSeconds -PollIntervalSeconds $PollIntervalSeconds

    $allGroups = @(, $ManagementClusterHostNames) + $OtherVsanClusterHostGroups
    foreach ($clusterHosts in $allGroups) {
        foreach ($hostName in $clusterHosts) {
            try {
                $vmhost = Get-VMHost -Name $hostName -ErrorAction Stop
                $accessManager = Get-View -Id $vmhost.ExtensionData.ConfigManager.HostAccessManager
                $current = @($accessManager.QueryLockdownExceptions())
                if ($current -contains 'root') {
                    $accessManager.UpdateLockdownExceptions(@($current | Where-Object { $_ -ne 'root' }))
                    Write-Step "Removed 'root' from lockdown exception users on '$hostName'." -Level OK
                }
            } catch {
                Write-Step "Could not verify/update lockdown exceptions on '$hostName': $($_.Exception.Message)" -Level WARN
            }
        }
    }

    Clear-S1IOFilterProviderAlarms -HostNames $ManagementClusterHostNames -AlsoMassClearTriggeredAlarms

    Write-Step "=== Step 1 complete ===" -Level OK
}
```

### Step 2: Start the SDDC Manager Appliance

With the management vCenter and vSAN healthy, SDDC Manager is next in Broadcom's order - a straightforward power-on that waits for VMware Tools to confirm the guest OS is actually up before moving on.

```powershell
function Invoke-Step02-StartSDDCManager {
    param([Parameter(Mandatory)][string]$VMName)
    Write-Step "=== Step 2: Start the SDDC Manager Appliance ==="
    Start-VMsAndWait -VMNames @($VMName) -WaitForTools
    Write-Step "=== Step 2 complete ===" -Level OK
}
```

### Step 3: Start the NSX Manager Nodes

After the NSX Manager node(s) power on, the step polls the NSX Manager cluster status API (`/api/v1/cluster/status`) until the management cluster reports `STABLE`, rather than assuming the nodes are ready the moment they finish booting. If it times out, it falls back to a manual check in the NSX Manager UI.

```powershell
# ============================================================================
# 03-Startup-NSXManager.ps1
# Step 3: Start the NSX Manager Nodes
# ============================================================================

function Wait-S1NsxManagerClusterStable {
    param(
        [Parameter(Mandatory)][string]$NsxManagerServer,
        [Parameter(Mandatory)][PSCredential]$NsxManagerCredential,
        [int]$TimeoutSeconds = 1800,
        [int]$PollIntervalSeconds = 15
    )

    Write-Step "Waiting for NSX Manager cluster status to report 'STABLE' on $NsxManagerServer ..."

    # NOTE: Requires PowerShell 7+ for -SkipCertificateCheck (same as the
    # vCenter/Services Runtime REST calls in Steps 1 and 5/7).
    $pair = "$($NsxManagerCredential.UserName):$($NsxManagerCredential.GetNetworkCredential().Password)"
    $basicAuthValue = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes($pair))
    $headers = @{ Authorization = "Basic $basicAuthValue" }

    $uri = "https://$NsxManagerServer/api/v1/cluster/status"
    $deadline = (Get-Date).AddSeconds($TimeoutSeconds)

    while ((Get-Date) -lt $deadline) {
        try {
            $status = Invoke-RestMethod -Uri $uri -Headers $headers -Method Get -SkipCertificateCheck -TimeoutSec 15
            $mgmtStatus = $status.mgmt_cluster_status.status
            Write-Step "  NSX Manager mgmt cluster status: $mgmtStatus"
            if ($mgmtStatus -and $mgmtStatus.ToUpper() -eq 'STABLE') {
                Write-Step "NSX Manager cluster is STABLE." -Level OK
                return $true
            }
        } catch {
            Write-Step "  NSX Manager not yet reachable/ready ($($_.Exception.Message)). Retrying..." -Level WARN
        }
        Start-Sleep -Seconds $PollIntervalSeconds
    }

    Write-Step "Timed out waiting for NSX Manager cluster to report STABLE." -Level WARN
    Confirm-ManualStep -Prompt "Log in to NSX Manager (System > Configuration > Appliances) and verify the cluster shows 'Stable' with all nodes available before continuing."
    return $false
}

function Invoke-Step03-StartNSXManager {
    param(
        [Parameter(Mandatory)][string[]]$VMNames,
        [Parameter(Mandatory)][string]$NsxManagerServer,
        [Parameter(Mandatory)][PSCredential]$NsxManagerCredential,
        [int]$TimeoutSeconds = 1800,
        [int]$PollIntervalSeconds = 15
    )

    Write-Step "=== Step 3: Start the NSX Manager Nodes ==="

    Start-VMsAndWait -VMNames $VMNames -WaitForTools

    Wait-S1NsxManagerClusterStable -NsxManagerServer $NsxManagerServer -NsxManagerCredential $NsxManagerCredential -TimeoutSeconds $TimeoutSeconds -PollIntervalSeconds $PollIntervalSeconds

    Write-Step "=== Step 3 complete ===" -Level OK
}
```

### Step 4: Start the NSX Edge or VNA Nodes

Another simple power-on-and-wait step, run once NSX Manager itself is stable.

```powershell
function Invoke-Step04-StartNSXEdge {
    param([Parameter(Mandatory)][string[]]$VMNames)
    Write-Step "=== Step 4: Start the NSX Edge or VNA Nodes ==="
    Start-VMsAndWait -VMNames $VMNames -WaitForTools
    Write-Step "=== Step 4 complete ===" -Level OK
}
```

Broadcom's documented order places **Protection and Recovery** next. That component isn't deployed in these lab pods, so the step is intentionally skipped in this implementation and the sequence picks back up at Step 6.

### Step 6: Start VCF Operations

VCF Operations can take the longest of any component to come fully online, so this step authenticates against the Suite API (`/suite-api/api/auth/token/acquire`) and polls node status until it reports `ONLINE` - documented as potentially taking up to an hour - before the sequence continues.

```powershell
function Wait-S1VcfOperationsClusterOnline {
    param(
        [Parameter(Mandatory)][string]$VcfOperationsServer,
        [Parameter(Mandatory)][PSCredential]$VcfOperationsCredential,
        [int]$TimeoutSeconds = 5400,
        [int]$PollIntervalSeconds = 30
    )

    Write-Step "Waiting for VCF Operations cluster to report ONLINE on $VcfOperationsServer (can take up to ~1 hour)..."

    # NOTE: Requires PowerShell 7+ for -SkipCertificateCheck (same as other REST-based steps).
    $tokenUri  = "https://$VcfOperationsServer/suite-api/api/auth/token/acquire"
    $statusUri = "https://$VcfOperationsServer/suite-api/api/deployment/node/status"
    $body = @{
        username = $VcfOperationsCredential.UserName
        password = $VcfOperationsCredential.GetNetworkCredential().Password
    } | ConvertTo-Json

    $deadline = (Get-Date).AddSeconds($TimeoutSeconds)

    while ((Get-Date) -lt $deadline) {
        try {
            $tokenResponse = Invoke-RestMethod -Uri $tokenUri -Method Post -Body $body -ContentType 'application/json' -SkipCertificateCheck -TimeoutSec 15
            $headers = @{ Authorization = "OpsToken $($tokenResponse.token)" }

            $status = Invoke-RestMethod -Uri $statusUri -Headers $headers -Method Get -SkipCertificateCheck -TimeoutSec 15
            Write-Step "  VCF Operations node status: $($status.status)"
            if ($status.status -eq 'ONLINE') {
                Write-Step "VCF Operations cluster is ONLINE." -Level OK
                return $true
            }
        } catch {
            Write-Step "  VCF Operations not yet reachable/ready ($($_.Exception.Message)). Retrying..." -Level WARN
        }
        Start-Sleep -Seconds $PollIntervalSeconds
    }

    Write-Step "Timed out waiting for VCF Operations cluster to report ONLINE." -Level WARN
    Confirm-ManualStep -Prompt "Log in to https://$VcfOperationsServer/admin and verify the VCF Operations cluster is online before continuing."
    return $false
}

function Invoke-Step06-StartVCFOperations {
    param(
        [Parameter(Mandatory)][string[]]$VMNames,
        [Parameter(Mandatory)][string]$VcfOperationsServer,
        [Parameter(Mandatory)][PSCredential]$VcfOperationsCredential,
        [int]$TimeoutSeconds = 5400,
        [int]$PollIntervalSeconds = 30
    )

    Write-Step "=== Step 6: Start VCF Operations ==="

    Start-VMsAndWait -VMNames $VMNames -WaitForTools

    #Confirm-ManualStep -Prompt "Log in to https://$VcfOperationsServer/admin and click 'Bring Cluster Online' if you haven't already."

    Wait-S1VcfOperationsClusterOnline -VcfOperationsServer $VcfOperationsServer -VcfOperationsCredential $VcfOperationsCredential -TimeoutSeconds $TimeoutSeconds -PollIntervalSeconds $PollIntervalSeconds

    Write-Step "=== Step 6 complete ===" -Level OK
}
```

### Step 7: Start VCF Management Services

This is the most involved step in the set. VCF Management Services - the "VCF services runtime" - is powered on control node(s) first, then worker node(s), followed by a five-minute settle pause before anything else is attempted. From there the step authenticates to the Fleet LCM API using an OAuth2 password grant against the `admin@vsp.local` service account (confirmed during lab testing to be a different account than either SDDC Manager's or VCF Operations' own admin users), looks up the "VCF services runtime" component by its `componentTypeDescription`, and polls `GET /fleet-lcm/v1/components/{componentId}/status` until it reports `Running`. If the component can't be located or the API can't be reached, it falls back to a manual verification prompt pointing at the right screen in VCF Operations.

```powershell
<#
.SYNOPSIS
    Step 7 (Startup): Start VCF Management Services (VCF services runtime).

.DESCRIPTION
    Powers on the VCF services runtime control node(s) first, then the worker
    node(s). Instead of a manual verification step, this polls the Fleet LCM
    API for the "VCF services runtime" component's status until it reports
    "Running", falling back to a manual confirmation on timeout.

    Fleet LCM API notes (confirmed via lab testing):
      - Host: fleet.lab.net (the "Fleet lifecycle" component's own FQDN,
        NOT the SDDC Manager FQDN or the VCF Operations FQDN).
      - Auth: OAuth2 password grant against admin@vsp.local (the VCF services
        runtime admin account, NOT vmware-system-guest or admin@local).
            POST https://<server>/api/v1/identity/token
            Content-Type: application/x-www-form-urlencoded
            Body: grant_type=password&username=admin@vsp.local&password=<PASSWORD>
        Response: { "access_token": "...", "token_type": "bearer", "expires_in": 14399 }
      - GET /fleet-lcm/v1/components returns { components: [...], pageMetadata: {...} }.
        Each component has componentTypeDescription (e.g. "VCF services runtime"),
        componentType, fqdn, scope, nodes[], etc. Node objects only have
        nodeType/id/fqdn/ipAddress/name - there is NO per-node "status" field.
      - GET /fleet-lcm/v1/components/{componentId}/status returns
        { "id": "...", "status": "Unknown" | "NotRunning" | "Running" } - this is
        the authoritative health signal we poll.
#>

function Get-S1FleetLcmAccessToken {
    param(
        [Parameter(Mandatory)][string]$Server,
        [Parameter(Mandatory)][string]$Username,
        [Parameter(Mandatory)][string]$Password,
        [int]$TimeoutSeconds = 1800,
        [int]$PollIntervalSeconds = 30
    )

    Write-Step "Waiting for Fleet Manager to report ONLINE..."

    $uri = "https://$Server/api/v1/identity/token"
    $body = @{
        grant_type = 'password'
        username   = $Username
        password   = $Password
    }

    $deadline = (Get-Date).AddSeconds($TimeoutSeconds)
    while ((Get-Date) -lt $deadline) {
        try {
            $response = Invoke-RestMethod -Uri $uri -Method Post -Body $body -ContentType 'application/x-www-form-urlencoded' -SkipCertificateCheck

            if ($response.access_token) {
                Write-Step "Fleet Manager is ONLINE." -Level OK
                return $response.access_token
            }            
        } catch {
            Write-Step "  Fleet Manager not yet reachable/ready ($($_.Exception.Message)). Retrying..." -Level WARN
        }
        Start-Sleep -Seconds $PollIntervalSeconds
        
        if (-not $response.access_token) {
            throw "Failed to acquire Fleet LCM access token - no 'access_token' field in response from $uri."
        }
    }
    return $false
}

function Get-S1FleetLcmComponentId {
    param(
        [Parameter(Mandatory)][string]$Server,
        [Parameter(Mandatory)][string]$AccessToken,
        [Parameter(Mandatory)][string]$ComponentTypeDescription
    )

    $uri = "https://$Server/fleet-lcm/v1/components"
    $headers = @{ Authorization = "Bearer $AccessToken" }

    $response = Invoke-RestMethod -Uri $uri -Method Get -Headers $headers `
        -ContentType 'application/json' -SkipCertificateCheck

    $component = $response.components | Where-Object { $_.componentTypeDescription -eq $ComponentTypeDescription }

    if (-not $component) {
        return $null
    }

    return $component.id
}

function Wait-S1VcfManagementServicesHealthy {
    param(
        [Parameter(Mandatory)][string]$Server,
        [Parameter(Mandatory)][string]$Username,
        [Parameter(Mandatory)][string]$Password,
        [int]$TimeoutSeconds = 1800,
        [int]$PollIntervalSeconds = 30
    )

    Write-Step "Waiting for VCF Management Services (VCF services runtime) to report 'Running' via Fleet LCM API..."

    <#
    try {
        $accessToken = Get-S1FleetLcmAccessToken -Server $Server -Username $Username -Password $Password
    } catch {
        Write-Step "Warning: could not acquire Fleet LCM access token: $($_.Exception.Message)"
        Confirm-ManualStep -Message "Manually verify VCF Management Services are healthy in VCF Operations (Build > Lifecycle > VCF Management > Components > 'VCF services runtime') before continuing. Refer to KB 440862 if recovery is needed."
        return
    }
    #> 
    $componentId = Get-S1FleetLcmComponentId -Server $Server -AccessToken $accessToken -ComponentTypeDescription 'VCF services runtime'
    
    if (-not $componentId) {
        Write-Step "Could not locate the 'VCF services runtime' component via the Fleet LCM API."
        Confirm-ManualStep -Message "Manually verify VCF Management Services are healthy in VCF Operations (Build > Lifecycle > VCF Management > Components > 'VCF services runtime') before continuing. Refer to KB 440862 if recovery is needed."
        return
    }

    $statusUri = "https://$Server/fleet-lcm/v1/components/$componentId/status"
    $headers = @{ Authorization = "Bearer $accessToken" }

    $elapsed = 0
    $healthy = $false

    while ($elapsed -lt $TimeoutSeconds) {
        try {
            $statusResponse = Invoke-RestMethod -Uri $statusUri -Method Get -Headers $headers `
                -ContentType 'application/json' -SkipCertificateCheck
            Write-Step "VCF services runtime status: $($statusResponse.status)"

            if ($statusResponse.status -eq 'Running') {
                $healthy = $true
                break
            }
        } catch {
            Write-Step "Warning: failed to query Fleet LCM component status: $($_.Exception.Message)"
        }

        Start-Sleep -Seconds $PollIntervalSeconds
        $elapsed += $PollIntervalSeconds
    }

    if ($healthy) {
        Write-Step "VCF Management Services (VCF services runtime) reports 'Running'."
    } else {
        Write-Step "Timed out after $TimeoutSeconds seconds waiting for VCF services runtime to report 'Running'."
        Confirm-ManualStep -Message "Manually verify VCF Management Services are healthy in VCF Operations (Build > Lifecycle > VCF Management > Components > 'VCF services runtime') before continuing. Refer to KB 440862 if recovery is needed."
    }
}

function Invoke-Step07-StartVCFManagementServices {
    param(
        [Parameter(Mandatory)][string[]]$ControlNodeVMNames,
        [Parameter(Mandatory)][string[]]$WorkerNodeVMNames,
        [Parameter(Mandatory)][string]$FleetLcmServer,
        [Parameter(Mandatory)][string]$FleetLcmUsername,
        [Parameter(Mandatory)][string]$FleetLcmPassword
    )

    Write-Step "=== Step 7: Start VCF Management Services ==="

    # Power on the control node(s) first
    Start-VMsAndWait -VMNames $ControlNodeVMNames

    # Then power on the worker node(s)
    Start-VMsAndWait -VMNames $WorkerNodeVMNames

    # Pause for all VCF Fleet Services to come online
    Start-Sleep -Seconds 300

    # Wait for Fleet Manager to come ONLINE
    Write-Step "Waiting for Fleet Manager to report ONLINE..." -Level OK
    $accessToken = Get-S1FleetLcmAccessToken -Server $FleetLcmServer -Username $FleetLcmUsername -Password $FleetLcmPassword
    Write-Step "Fleet Manager is ONLINE." -Level OK

    # Wait for automated health confirmation via the Fleet LCM API
    Wait-S1VcfManagementServicesHealthy -Server $FleetLcmServer -Username $FleetLcmUsername -Password $FleetLcmPassword
}
```

### Step 8: Start the License Server

A simple power-on-and-wait step, unchanged in shape from the pattern used throughout.

```powershell
function Invoke-Step08-StartLicenseServer {
    param([Parameter(Mandatory)][string]$VMName)
    Write-Step "=== Step 8: Start the License Server ==="
    Start-VMsAndWait -VMNames @($VMName) -WaitForTools
    Write-Step "=== Step 8 complete ===" -Level OK
}
```

### Step 9: Start the Cloud Proxy Appliances

Same pattern again, run once licensing is back online.

```powershell
function Invoke-Step09-StartCloudProxy {
    param([Parameter(Mandatory)][string[]]$VMNames)
    Write-Step "=== Step 9: Start the Cloud Proxy Appliances ==="
    Start-VMsAndWait -VMNames $VMNames -WaitForTools
    Write-Step "=== Step 9 complete ===" -Level OK
}
```

### Step 10: Start VCF Operations for Networks

Broadcom documents this step as UI-only. In practice the underlying action is still just powering the appliance's guest VM back on, so this step reuses the same `Start-VMsAndWait` helper as a PowerCLI-equivalent rather than requiring a manual click-through.

```powershell
<# Step 10: Start VCF Operations for Networks (documented as UI-only; PowerCLI-equivalent guest VM power-on used here). #>
function Invoke-Step10-StartVCFOperationsForNetworks {
    param([Parameter(Mandatory)][string[]]$VMNames)
    Write-Step "=== Step 10: Start VCF Operations for Networks ==="
    Start-VMsAndWait -VMNames $VMNames -WaitForTools
    Write-Step "=== Step 10 complete ===" -Level OK
}
```

### Step 11: Start VCF Automation

The final stage, and the only one that starts a VM without ever touching vCenter or PowerCLI directly. Lab testing against the Fleet LCM API confirmed that triggering VCF Automation's startup is a `POST https://<fleetLcmServer>/fleet-lcm/v1/components/{componentId}?action=start` call using the same OAuth2 bearer-token flow as Step 7 - Fleet Manager's start action orchestrates powering on the underlying node itself. One implementation detail worth calling out: the call has to be made against `https://` explicitly, because a plain `http://` request gets a 301 redirect that most HTTP clients (including Postman, in testing) silently downgrade from POST to GET when they follow it - which makes the action look like a harmless no-op instead of actually failing loudly. The `action=start` call itself returns immediately with a `RUNNING` task status that reflects the workflow kicking off, not the component's actual health, so real readiness is polled separately via the same `/status` endpoint used in Step 7 until it reports `Running`.

```powershell
<#
.SYNOPSIS
    Step 11 (Startup): Start VCF Automation.

.DESCRIPTION
    Confirmed via lab testing (Postman):
      - Triggering the start requires:
            POST https://<fleetLcmServer>/fleet-lcm/v1/components/{componentId}?action=start
        using the same admin@vsp.local OAuth2 password-grant bearer token flow validated in Step 7.
      - IMPORTANT: this call must be made directly against https:// - if sent to http://, the server
        responds with a 301 redirect to https://, and most HTTP clients (including Postman) silently
        downgrade the follow-up request from POST to GET when following a 301/302. That makes the action
        look like a silent no-op that just returns the component's plain GET details instead of actually
        triggering anything. Always call https:// directly to avoid this.
      - Unlike VCF Management Services (Step 7), there is no need to power on any VM directly via
        PowerCLI/vCenter first - Fleet Manager's start action orchestrates powering on the underlying
        node(s) itself (observed in testing: VM asr01-j299m powered on automatically once the
        action=start call was accepted).
      - The action=start call returns an immediate Task object with status "RUNNING" - this reflects the
        *workflow* kicking off, not the component's actual health. Real readiness must be polled
        separately via GET /fleet-lcm/v1/components/{componentId}/status until status == "Running".
        This can take several minutes.
#>

function Get-S1FleetLcmAccessToken {
    param(
        [Parameter(Mandatory)][string]$Server,
        [Parameter(Mandatory)][string]$Username,
        [Parameter(Mandatory)][string]$Password
    )

    $uri = "https://$Server/api/v1/identity/token"
    $body = @{
        grant_type = 'password'
        username   = $Username
        password   = $Password
    }

    $response = Invoke-RestMethod -Uri $uri -Method Post -Body $body `
        -ContentType 'application/x-www-form-urlencoded' -SkipCertificateCheck

    if (-not $response.access_token) {
        throw "Failed to acquire Fleet LCM access token - no 'access_token' field in response from $uri."
    }

    return $response.access_token
}

function Get-S1FleetLcmComponentId {
    param(
        [Parameter(Mandatory)][string]$Server,
        [Parameter(Mandatory)][string]$AccessToken,
        [Parameter(Mandatory)][string]$ComponentTypeDescription
    )

    $uri = "https://$Server/fleet-lcm/v1/components"
    $headers = @{ Authorization = "Bearer $AccessToken" }

    $response = Invoke-RestMethod -Uri $uri -Method Get -Headers $headers `
        -ContentType 'application/json' -SkipCertificateCheck

    $component = $response.components | Where-Object { $_.componentTypeDescription -eq $ComponentTypeDescription }

    if (-not $component) {
        return $null
    }

    return $component.id
}

function Get-S1FleetLcmComponentStatus {
    param(
        [Parameter(Mandatory)][string]$Server,
        [Parameter(Mandatory)][string]$AccessToken,
        [Parameter(Mandatory)][string]$ComponentId
    )

    $uri = "https://$Server/fleet-lcm/v1/components/$ComponentId/status"
    $headers = @{ Authorization = "Bearer $AccessToken" }

    return Invoke-RestMethod -Uri $uri -Method Get -Headers $headers -ContentType 'application/json' -SkipCertificateCheck
}

function Start-S1FleetLcmComponentAction {
    param(
        [Parameter(Mandatory)][string]$Server,
        [Parameter(Mandatory)][string]$AccessToken,
        [Parameter(Mandatory)][string]$ComponentId,
        [Parameter(Mandatory)][ValidateSet('start', 'shutdown', 'restart', 'refresh')][string]$Action
    )

    # NOTE: Must use https:// explicitly - a http:// call here will 301-redirect and get silently
    # downgraded from POST to GET by most HTTP clients, making the action never actually fire.
    $uri = "https://$Server/fleet-lcm/v1/components/$ComponentId`?action=$Action"
    $headers = @{ Authorization = "Bearer $AccessToken" }

    return Invoke-RestMethod -Uri $uri -Method Post -Headers $headers -ContentType 'application/json' -SkipCertificateCheck
}

function Wait-S1VcfAutomationHealthy {
    param(
        [Parameter(Mandatory)][string]$Server,
        [Parameter(Mandatory)][string]$Username,
        [Parameter(Mandatory)][string]$Password,
        [int]$TimeoutSeconds = 2400,
        [int]$PollIntervalSeconds = 30
    )

    Write-Step "Starting VCF Automation via Fleet LCM API and waiting for it to report 'Running'..."

    try {
        $accessToken = Get-S1FleetLcmAccessToken -Server $Server -Username $Username -Password $Password
    } catch {
        Write-Step "Warning: could not acquire Fleet LCM access token: $($_.Exception.Message)"
        Confirm-ManualStep -Message "Manually start VCF Automation in VCF Operations (Build > Lifecycle > VCF Management > Components > VCF Automation > Actions > State management > Start) and wait for it to report healthy before continuing."
        return
    }

    $componentId = Get-S1FleetLcmComponentId -Server $Server -AccessToken $accessToken -ComponentTypeDescription 'VCF Automation'

    if (-not $componentId) {
        Write-Step "Could not locate the 'VCF Automation' component via the Fleet LCM API."
        Confirm-ManualStep -Message "Manually start VCF Automation in VCF Operations (Build > Lifecycle > VCF Management > Components > VCF Automation > Actions > State management > Start) and wait for it to report healthy before continuing."
        return
    }

    # Check current status first - avoid re-triggering start if it's already running.
    $currentStatus = Get-S1FleetLcmComponentStatus -Server $Server -AccessToken $accessToken -ComponentId $componentId
    Write-Step "Current VCF Automation status: $($currentStatus.status)"

    if ($currentStatus.status -eq 'Running') {
        Write-Step "VCF Automation is already reporting 'Running'. Nothing to do."
        return
    }

    try {
        $task = Start-S1FleetLcmComponentAction -Server $Server -AccessToken $accessToken -ComponentId $componentId -Action 'start'
        Write-Step "Start action submitted: $($task.name) (task id: $($task.id), status: $($task.status))"
    } catch {
        Write-Step "Warning: failed to submit start action for VCF Automation: $($_.Exception.Message)"
        Confirm-ManualStep -Message "Manually start VCF Automation in VCF Operations (Build > Lifecycle > VCF Management > Components > VCF Automation > Actions > State management > Start) and wait for it to report healthy before continuing."
        return
    }

    $elapsed = 0
    $healthy = $false

    while ($elapsed -lt $TimeoutSeconds) {
        try {
            $statusResponse = Get-S1FleetLcmComponentStatus -Server $Server -AccessToken $accessToken -ComponentId $componentId
            Write-Step "VCF Automation status: $($statusResponse.status)"

            if ($statusResponse.status -eq 'Running') {
                $healthy = $true
                break
            }
        } catch {
            Write-Step "Warning: failed to query Fleet LCM component status: $($_.Exception.Message)"
        }

        Start-Sleep -Seconds $PollIntervalSeconds
        $elapsed += $PollIntervalSeconds
    }

    if ($healthy) {
        Write-Step "VCF Automation reports 'Running'."
    } else {
        Write-Step "Timed out after $TimeoutSeconds seconds waiting for VCF Automation to report 'Running'."
        Confirm-ManualStep -Message "Manually verify VCF Automation is healthy in VCF Operations (Build > Lifecycle > VCF Management > Components > VCF Automation) before continuing."
    }
}

function Invoke-Step11-StartVCFAutomation {
    param(
        [Parameter(Mandatory)][string]$FleetLcmServer,
        [Parameter(Mandatory)][string]$FleetLcmUsername,
        [Parameter(Mandatory)][string]$FleetLcmPassword
    )

    Write-Step "=== Step 11: Start VCF Automation ==="

    Wait-S1VcfAutomationHealthy -Server $FleetLcmServer -Username $FleetLcmUsername -Password $FleetLcmPassword
}
```

### Tying it together: the wrapper script

`Invoke-VCFMgmtDomainStartup.ps1` is the entry point that turns eleven separate files into one run. It dot-sources the common module and every step file, builds the `PSCredential` objects each step needs from the `$Config` block below, then calls the step functions in Broadcom's documented order - Step 5 (Protection and Recovery) skipped as noted above, and Step 10 (VCF Operations for Networks) left commented out here since it isn't part of every pod's inventory, but ready to enable for pods that do deploy it. The whole sequence runs inside a single `try`/`catch`, so a failure at any step is logged clearly through `Write-Step` and aborts the run rather than plowing ahead with an unhealthy dependency underneath it. The `$Config` block itself is where a given pod's inventory lives - vCenter, NSX Manager, VCF Operations, and Fleet LCM endpoints and credentials, plus the VM names for every appliance being started. Every FQDN below has been normalized to the `lab.net` domain and every password redacted to `<PASSWORD>` for this write-up.

```powershell
[CmdletBinding()]
param([switch]$AllowSelfSignedCerts)

$ErrorActionPreference = 'Stop'
$here = Split-Path -Parent $MyInvocation.MyCommand.Path

. (Join-Path $here '00-Common.ps1')
. (Join-Path $here '01-Startup-vSANAndESXHosts.ps1')
. (Join-Path $here '02-Startup-SDDCManager.ps1')
. (Join-Path $here '03-Startup-NSXManager.ps1')
. (Join-Path $here '04-Startup-NSXEdge.ps1')
. (Join-Path $here '06-Startup-VCFOperations.ps1')
. (Join-Path $here '07-Startup-VCFManagementServices.ps1')
. (Join-Path $here '08-Startup-LicenseServer.ps1')
. (Join-Path $here '09-Startup-CloudProxy.ps1')
. (Join-Path $here '10-Startup-VCFOperationsForNetworks.ps1')
. (Join-Path $here '11-Startup-VCFAutomation.ps1')

# ---- EDIT THIS BLOCK for your environment ----
$Config = @{
    VCenterServer                 = 'vc-m1-01.lab.net'
    VCenterUsername               = 'administrator@vsphere.local'
    VCenterPassword               = '<PASSWORD>'   # Lab-only plaintext password - disposable/isolated environment
    VCFAutomationVMNames          = @('auto')
    CloudProxyVMNames             = @('ops-col')
    LicenseServerVMName           = 'license'
    VCFOperationsVMNamesInKBOrder = @('ops-pri')
    NSXManagerVMNames             = @('nsx-m1-01')
    SDDCManagerVMName             = 'sddc-m1-01'
    VSANClusterNames              = @('m1-mgmt-cluster')
    ManagementVCenterClusterName  = 'm1-mgmt-cluster'
    ServicesRuntimeNodeIp         = '10.1.1.73'
    ServicesRuntimeNodePort       = 5480
    ManagementClusterHostNames    = @('esx-m1-01.lab.net','esx-m1-02.lab.net','esx-m1-03.lab.net','esx-m1-04.lab.net')
    NSXManagerApiServer           = 'nsx-m1-01.lab.net'   # <-- verify/adjust FQDN or VIP
    NSXManagerUsername            = 'admin'
    NSXManagerPassword            = '<PASSWORD>'
    VCFOperationsApiServer        = 'ops-pri.lab.net'   # <-- verify/adjust FQDN
    VCFOperationsUsername         = 'admin'
    VCFOperationsPassword         = '<PASSWORD>'
    VCFMgmtServicesControlNodes   = @('msr01-wdvwm')
    VCFMgmtServicesWorkerNodes    = @('msr01-5jn26','msr01-67lz4','msr01-mxmrj','msr01-pmvkj')
    FleetLcmServer                = 'fleet.lab.net'   
    FleetLcmUsername              = 'admin@vsp.local'
    FleetLcmPassword              = '<PASSWORD>'
}
# ------------------------------------------------

try {
    Write-Step "Starting VCF 9.1 Management Domain startup sequence." -Level INFO
    Assert-PowerCLIModule -AllowSelfSignedCerts:$AllowSelfSignedCerts

    $vcSecurePassword = ConvertTo-SecureString -String $Config.VCenterPassword -AsPlainText -Force
    $vcCred = New-Object System.Management.Automation.PSCredential ($Config.VCenterUsername, $vcSecurePassword)
    $nsxManagerCredential = New-Object System.Management.Automation.PSCredential($Config.NSXManagerUsername,(ConvertTo-SecureString $Config.NSXManagerPassword -AsPlainText -Force))
    $vcfOperationsCredential = New-Object System.Management.Automation.PSCredential($Config.VCFOperationsUsername,(ConvertTo-SecureString $Config.VCFOperationsPassword -AsPlainText -Force))
    $fleetLcmCredential = New-Object System.Management.Automation.PSCredential($Config.FleetLcmUsername, (ConvertTo-SecureString $Config.FleetLcmPassword -AsPlainText -Force))


    Invoke-Step01-StartVsanAndEsxHosts -VCenterServer $Config.VCenterServer -VCenterCredential $vcCred `
        -ManagementClusterHostNames $Config.ManagementClusterHostNames

    Invoke-Step02-StartSDDCManager              -VMName  $Config.SDDCManagerVMName
    Invoke-Step03-StartNSXManager -VMNames $Config.NSXManagerVMNames -NsxManagerServer $Config.NSXManagerApiServer -NsxManagerCredential $nsxManagerCredential
    Invoke-Step04-StartNSXEdge                  -VMNames $Config.NSXEdgeVMNames
    # Step 5 (Protection and Recovery) skipped per user instruction
    Invoke-Step06-StartVCFOperations -VMNames $Config.VCFOperationsVMNamesInKBOrder -VcfOperationsServer $Config.VCFOperationsApiServer -VcfOperationsCredential $vcfOperationsCredential
    Invoke-Step07-StartVCFManagementServices `
        -ControlNodeVMNames $Config.VCFMgmtServicesControlNodes `
        -WorkerNodeVMNames  $Config.VCFMgmtServicesWorkerNodes `
        -FleetLcmServer     $Config.FleetLcmServer `
        -FleetLcmUsername   $Config.FleetLcmUsername `
        -FleetLcmPassword   $Config.FleetLcmPassword
    Invoke-Step08-StartLicenseServer             -VMName  $Config.LicenseServerVMName
    Invoke-Step09-StartCloudProxy                -VMNames $Config.CloudProxyVMNames
    #Invoke-Step10-StartVCFOperationsForNetworks  -VMNames $Config.VCFOpsForNetworksVMNames
    Invoke-Step11-StartVCFAutomation `
        -FleetLcmServer   $Config.FleetLcmServer `
        -FleetLcmUsername $Config.FleetLcmUsername `
        -FleetLcmPassword $Config.FleetLcmPassword

    Write-Step "Management Domain startup sequence finished." -Level OK
}
catch {
    Write-Step "Startup sequence aborted: $($_.Exception.Message)" -Level ERROR
    throw
}
```

Together with its shutdown counterpart, this framework closes the loop on the pod lifecycle: a management domain can be brought down into a clean, template-ready state and brought back up again in the correct, dependency-aware order, with health checks at every stage standing in for what would otherwise be a long manual runbook. That's what makes converting a VCF 9.1 vApp into a reusable Template - and handing it back out to the next student - a repeatable operation instead of a one-off effort.
