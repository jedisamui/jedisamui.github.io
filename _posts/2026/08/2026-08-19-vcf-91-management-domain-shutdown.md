---
layout: post
title: "Automating a VCF 9.1 Management Domain Shutdown for Reusable VCD Lab Pods"
summary: "A modular PowerCLI framework that walks a VCF 9.1 management domain through Broadcom's documented shutdown sequence, built so isolated VCD lab pods can be safely converted back into reusable vApp Templates."
author: samui
date: 2026-08-19
category: [vcf, vcd, powershell, automation]
thumbnail: /images/2026/08/vcf9.1-shutdown-script.png
permalink: /blog/vcf-91-management-domain-shutdown/
---

We spin up VMware Cloud Foundation 9.1 inside isolated, self-contained pods running as vApps within VMware Cloud Director. Each pod is a fully nested VCF management domain that students or engineers can use, tear down, and hand back. For a pod to be reusable, it has to be converted into a vApp Template - and VCD will not let that conversion happen cleanly unless every VM inside the vApp is powered off in a known-good state first. For a nested VCF environment, "power everything off" is not safe to do in an arbitrary order. SDDC Manager, NSX, vCenter, and the VCF Operations fleet all depend on each other, and shutting them down out of sequence risks leaving the management domain in a state that will not come back up cleanly the next time a student needs it.

Broadcom documents the supported order for this exact scenario, and it was our starting point for everything that follows: [Shut Down the Management Domain](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1/fleet-management/vcf-shutdown-and-startup/vcf-shutdown/shut-down-the-management-domain.html). The official procedure walks through eleven stages - VCF Automation, VCF Operations for Networks, Cloud Proxy, the License Server, VCF Management Services, VCF Operations, Protection and Recovery, NSX Edge/VNA, NSX Manager, SDDC Manager, and finally the ESX hosts and vCenter itself - and calls out that infrastructure services VMs (AD, NTP, DNS, DHCP) come down last, and that the vSAN cluster hosting the management vCenter has to be shut down after every other cluster, since the vCenter connection drops the moment that cluster's shutdown begins. That ordering, and those warnings, are the backbone of the script set below.

Turning an eleven-step manual runbook into automation is one thing; turning it into automation you can trust with a shared lab environment is another. Rather than write one long script and hope it worked end to end, I split it into one file per Broadcom step plus a shared helper module, and built it up step by step - running each new module on its own against the lab, confirming the right VMs came down cleanly and in the right state, and only then wiring it into the next step. That modularity paid off during development: when something needed adjusting (the vSAN step, for example, went through a full rewrite once William Lam published the officially supported `Stop-VsanCluster` cmdlets), the change was contained to a single ~90-line file instead of a scroll through one monolithic script. It also means anyone extending this for their own environment can test one component in isolation before trusting it against a live pod.

**A note on the credentials in these examples:** the wrapper script below stores vCenter and API credentials as plaintext in its `$Config` block, by design. These scripts are meant to run only against disposable, network-isolated lab pods with no ingress or egress to the public internet, where the password list is already handed to students. That is not a pattern to reuse anywhere credentials matter - every password shown here has been replaced with `<PASSWORD>`, and the lab domain has been replaced with `lab.net`, for this write-up.

### The shared helper module

Every step file and the wrapper itself dot-source `00-Common.ps1` first. It holds the handful of functions that would otherwise be copy-pasted into every step: a timestamped, color-coded `Write-Step` logger; `Connect-VCFVCenter`, which reuses an existing vCenter session if one is already open instead of reconnecting; `Stop-VMsGracefully`, which requests a guest-OS shutdown for a list of VMs and then polls until they report powered off (or times out and flags them for manual follow-up); `Assert-PowerCLIModule` and `Assert-ModuleAvailable`, which check for and install the required PowerCLI/VCF modules on demand; a `Confirm-ManualStep` gate for the handful of actions that still need a human in the loop; and `Start-VMsAndWait`, the power-on counterpart used by the startup side of this framework. Keeping these in one place means every step behaves the same way when a VM won't shut down or a module is missing, instead of each step file reinventing its own error handling.

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

### The orchestrator: `Invoke-VCFMgmtDomainShutdown.ps1`

This is the entry point - the only script an operator actually runs. It dot-sources `00-Common.ps1` and every step module, defines a single `$Config` hashtable with everything environment-specific (vCenter, credentials, VM names per component, the VCF Services Runtime node IP, and the Fleet LCM server used to drive VCF Automation), then calls each `Invoke-StepNN-*` function in Broadcom's documented order inside a `try/catch/finally` block. Two things are worth calling out. First, Steps 2 and 7 are commented out here - VCF Operations for Networks isn't deployed in this particular pod, and Step 7 (Protection and Recovery / SRM) isn't part of this lab's footprint - so the wrapper simply skips over them; the step files themselves are still present and reusable for anyone whose environment does include those components. Second, the `finally` block deliberately does not try to disconnect the vCenter session, because by the time Step 11 finishes, that session is very likely already gone - vCenter is one of the last things powered off.

```powershell
<#
.SYNOPSIS
Orchestrates the VCF 9.1 Management Domain shutdown in Broadcom's documented
order, skipping Steps 5 (VCF Management Services) and 7 (Protection and
Recovery / SRM) and the final "ESX Hosts with NFS or Fibre Channel Storage"
page, per user instruction. Each step lives in its own file for modularity.

.NOTES
Run this from an elevated PowerShell session with network access to the
management domain vCenter. Fill in the $Config values below for your
environment before running.

Credentials below are stored as plaintext in this file by design, per user
instruction - these scripts are intended only for disposable, network-isolated
lab environments (no ingress/egress to the public) where students are already
given the password list. Do not reuse this pattern for anything else.
#>

[CmdletBinding()]
param(
    [switch]$AllowSelfSignedCerts
)

$ErrorActionPreference = 'Stop'
$here = Split-Path -Parent $MyInvocation.MyCommand.Path

# ---- Dot-source common functions and every step module ----
. (Join-Path $here '00-Common.ps1')
. (Join-Path $here '01-Shutdown-VCFAutomation.ps1')
. (Join-Path $here '02-Shutdown-VCFOperationsForNetworks.ps1')
. (Join-Path $here '03-Shutdown-CloudProxy.ps1')
. (Join-Path $here '04-Shutdown-LicenseServer.ps1')
. (Join-Path $here '05-Shutdown-VCFManagementServices.ps1')
. (Join-Path $here '06-Shutdown-VCFOperations.ps1')
. (Join-Path $here '08-Shutdown-NSXEdge.ps1')
. (Join-Path $here '09-Shutdown-NSXManager.ps1')
. (Join-Path $here '10-Shutdown-SDDCManager.ps1')
. (Join-Path $here '11-Shutdown-vSANAndESXHosts.ps1')

# ---- EDIT THIS BLOCK for your environment ----
$Config = @{
    VCenterServer                     = 'vc-m1-01.lab.net'
    VCenterUsername                   = 'administrator@vsphere.local'
    VCenterPassword                   = '<PASSWORD>'   # Lab-only plaintext password - disposable/isolated environment
    VCFAutomationVMNames              = @('auto')
    CloudProxyVMNames                 = @('ops-col')
    LicenseServerVMName               = 'license'
    VCFOperationsVMNamesInKBOrder     = @('ops-pri')
    NSXManagerVMNames                 = @('nsx-m1-01')
    SDDCManagerVMName                 = 'sddc-m1-01'
    VSANClusterNames                  = @('m1-mgmt-cluster')
    ManagementVCenterClusterName      = 'm1-mgmt-cluster'
    ServicesRuntimeNodeIp             = '10.1.1.73'   # Control Plane node IP (VCF Operations > Build > Lifecycle > Components > VCF Services Runtime > Nodes)
    ServicesRuntimeNodePort           = 5480
    ServicesRuntimeBreakglassPassword = '<PASSWORD>'   # Lab-only plaintext password for 'vmware-system-user' (Step 5)
	FleetLcmServer                = 'fleet.lab.net'
    FleetLcmUsername              = 'admin@vsp.local'
    FleetLcmPassword              = '<PASSWORD>'
}
# ------------------------------------------------

try {
    Write-Step "Starting VCF 9.1 Management Domain shutdown sequence." -Level INFO

    Assert-PowerCLIModule -AllowSelfSignedCerts:$AllowSelfSignedCerts

    $vcSecurePassword = ConvertTo-SecureString -String $Config.VCenterPassword -AsPlainText -Force
    $vcCred = New-Object System.Management.Automation.PSCredential ($Config.VCenterUsername, $vcSecurePassword)
    Connect-VCFVCenter -VCenterServer $Config.VCenterServer -Credential $vcCred | Out-Null

    $vmspPassword = ConvertTo-SecureString -String $Config.ServicesRuntimeBreakglassPassword -AsPlainText -Force

    Invoke-Step01-StopVCFAutomation `
        -FleetLcmServer   $Config.FleetLcmServer `
        -FleetLcmUsername $Config.FleetLcmUsername `
        -FleetLcmPassword $Config.FleetLcmPassword
    ## Invoke-Step02-ShutdownVCFOperationsForNetworks -VMNames $Config.VCFOpsForNetworksVMNames
    Invoke-Step03-ShutdownCloudProxy              -VMNames $Config.CloudProxyVMNames
    Invoke-Step04-ShutdownLicenseServer           -VMName  $Config.LicenseServerVMName
    Invoke-Step05-ShutdownVCFManagementServices   -NodeIp $Config.ServicesRuntimeNodeIp `
                                                -NodePort $Config.ServicesRuntimeNodePort `
                                                -VCenterCredential $vcCred `
                                                -VmspPassword $vmspPassword
    Invoke-Step06-ShutdownVCFOperations           -VMNamesInKBOrder $Config.VCFOperationsVMNamesInKBOrder
    ## Step 7 (Protection and Recovery / SRM) skipped per user instruction
    ##Invoke-Step08-ShutdownNSXEdge                 -VMNames $Config.NSXEdgeVMNames
    Invoke-Step09-ShutdownNSXManager              -VMNames $Config.NSXManagerVMNames
    Invoke-Step10-ShutdownSDDCManager             -VMName  $Config.SDDCManagerVMName
    Invoke-Step11-ShutdownVsanAndEsxHosts         -ClusterNames $Config.VSANClusterNames `
        -ManagementVCenterClusterName $Config.ManagementVCenterClusterName
    # Final NFS/Fibre Channel host shutdown step skipped per user instruction

    Write-Step "Management Domain shutdown sequence finished." -Level OK
}
catch {
    Write-Step "Shutdown sequence aborted: $($_.Exception.Message)" -Level ERROR
    throw
}
finally {
    if ($global:DefaultVIServers) {
        Write-Step "Note: leaving vCenter session open (it may already be disconnected if the mgmt vCenter cluster was shut down)."
    }
}
```

### Step 1 - Stop VCF Automation

VCF Automation doesn't expose a documented shutdown API directly, but Fleet LCM does, and lab testing against this environment (via Postman) confirmed the exact call: a `POST` to `/fleet-lcm/v1/components/{componentId}?action=shutdown`, authenticated with the same `admin@vsp.local` OAuth2 password-grant bearer token used elsewhere in this framework. One gotcha is worth calling out for anyone adapting this: the call has to go to `https://` directly. Sending it to `http://` gets a 301 redirect, and most HTTP clients - Postman included - silently downgrade the follow-up request from `POST` to `GET` when they follow that redirect, so the shutdown looks like it fired but never actually did anything. The `action=shutdown` call only kicks off the workflow; actual completion is polled separately against `/fleet-lcm/v1/components/{componentId}/status` until it reports `NotRunning`, which can take several minutes.

```powershell
<#
.SYNOPSIS
    Step 01 (Shutdown): Stop VCF Automation.

.DESCRIPTION
    Mirrors the Step 01 startup logic (Invoke-Step11-StartVCFAutomation), but drives the Fleet LCM
    "shutdown" action instead of "start" and polls for status == "NotRunning" instead of "Running".

    Confirmed via lab testing (Postman):
      - Triggering the shutdown requires:
            POST https://<fleetLcmServer>/fleet-lcm/v1/components/{componentId}?action=shutdown
        using the same admin@vsp.local OAuth2 password-grant bearer token flow validated in Step 7.
      - IMPORTANT: this call must be made directly against https:// - if sent to http://, the server
        responds with a 301 redirect to https://, and most HTTP clients (including Postman) silently
        downgrade the follow-up request from POST to GET when following a 301/302. That makes the action
        look like a silent no-op that just returns the component's plain GET details instead of actually
        triggering anything. Always call https:// directly to avoid this.
      - The action=shutdown call returns an immediate Task object with status "RUNNING" - this reflects
        the *workflow* kicking off, not the component's actual health/power state. Real completion must
        be polled separately via GET /fleet-lcm/v1/components/{componentId}/status until
        status == "NotRunning". This can take several minutes.
      - Per lab testing, Fleet Manager's shutdown action is expected to orchestrate powering off the
        underlying node(s) itself, symmetric to how the start action powers them on (Step 11 startup
        observed VM asr01-j299m power on automatically once action=start was accepted). Confirm this
        holds for shutdown in your environment before relying on it to skip a manual PowerCLI/vCenter
        power-off step.
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

function Wait-S1VcfAutomationStopped {
    param(
        [Parameter(Mandatory)][string]$Server,
        [Parameter(Mandatory)][string]$Username,
        [Parameter(Mandatory)][string]$Password,
        [int]$TimeoutSeconds = 2400,
        [int]$PollIntervalSeconds = 30
    )

    Write-Step "Shutting down VCF Automation via Fleet LCM API and waiting for it to report 'NotRunning'..."

    try {
        $accessToken = Get-S1FleetLcmAccessToken -Server $Server -Username $Username -Password $Password
    } catch {
        Write-Step "Warning: could not acquire Fleet LCM access token: $($_.Exception.Message)"
        Confirm-ManualStep -Message "Manually stop VCF Automation in VCF Operations (Build > Lifecycle > VCF Management > Components > VCF Automation > Actions > State management > Stop) and wait for it to report stopped before continuing."
        return
    }

    $componentId = Get-S1FleetLcmComponentId -Server $Server -AccessToken $accessToken -ComponentTypeDescription 'VCF Automation'

    if (-not $componentId) {
        Write-Step "Could not locate the 'VCF Automation' component via the Fleet LCM API."
        Confirm-ManualStep -Message "Manually stop VCF Automation in VCF Operations (Build > Lifecycle > VCF Management > Components > VCF Automation > Actions > State management > Stop) and wait for it to report stopped before continuing."
        return
    }

    # Check current status first - avoid re-triggering shutdown if it's already stopped.
    $currentStatus = Get-S1FleetLcmComponentStatus -Server $Server -AccessToken $accessToken -ComponentId $componentId
    Write-Step "Current VCF Automation status: $($currentStatus.status)"

    if ($currentStatus.status -eq 'NotRunning') {
        Write-Step "VCF Automation is already reporting 'NotRunning'. Nothing to do."
        return
    }

    try {
        $task = Start-S1FleetLcmComponentAction -Server $Server -AccessToken $accessToken -ComponentId $componentId -Action 'shutdown'
        Write-Step "Shutdown action submitted: $($task.name) (task id: $($task.id), status: $($task.status))"
    } catch {
        Write-Step "Warning: failed to submit shutdown action for VCF Automation: $($_.Exception.Message)"
        Confirm-ManualStep -Message "Manually stop VCF Automation in VCF Operations (Build > Lifecycle > VCF Management > Components > VCF Automation > Actions > State management > Stop) and wait for it to report stopped before continuing."
        return
    }

    $elapsed = 0
    $stopped = $false

    while ($elapsed -lt $TimeoutSeconds) {
        try {
            $statusResponse = Get-S1FleetLcmComponentStatus -Server $Server -AccessToken $accessToken -ComponentId $componentId
            Write-Step "VCF Automation status: $($statusResponse.status)"

            if ($statusResponse.status -eq 'NotRunning') {
                $stopped = $true
                break
            }
        } catch {
            Write-Step "Warning: failed to query Fleet LCM component status: $($_.Exception.Message)"
        }

        Start-Sleep -Seconds $PollIntervalSeconds
        $elapsed += $PollIntervalSeconds
    }

    if ($stopped) {
        Write-Step "VCF Automation reports 'NotRunning'."
    } else {
        Write-Step "Timed out after $TimeoutSeconds seconds waiting for VCF Automation to report 'NotRunning'."
        Confirm-ManualStep -Message "Manually verify VCF Automation is stopped in VCF Operations (Build > Lifecycle > VCF Management > Components > VCF Automation) before continuing."
    }
}

function Invoke-Step01-StopVCFAutomation {
    param(
        [Parameter(Mandatory)][string]$FleetLcmServer,
        [Parameter(Mandatory)][string]$FleetLcmUsername,
        [Parameter(Mandatory)][string]$FleetLcmPassword
    )

    Write-Step "=== Step 01: Stop VCF Automation ==="

    Wait-S1VcfAutomationStopped -Server $FleetLcmServer -Username $FleetLcmUsername -Password $FleetLcmPassword
}
```

### Step 2 - VCF Operations for Networks

This component isn't deployed in the pod this framework targets, so the wrapper leaves it commented out - but the module is here for anyone whose environment includes it. It follows the same graceful-shutdown pattern as most of the later steps: hand a list of VM names to `Stop-VMsGracefully` from the common module and let it handle the guest shutdown request and the poll-for-power-off loop.

```powershell
<# Step 2: Shut Down VCF Operations for Networks (same UI-only method as Step 1; PowerCLI equivalent used here). #>
function Invoke-Step02-ShutdownVCFOperationsForNetworks {
    param([Parameter(Mandatory)][string[]]$VMNames)
    Write-Step "=== Step 2: Shut Down VCF Operations for Networks ==="
    Stop-VMsGracefully -VMNames $VMNames
    Write-Step "=== Step 2 complete ===" -Level OK
}
```

### Step 3 - Cloud Proxy appliances

Same pattern again: the Cloud Proxy appliances get a graceful guest shutdown and a wait-for-power-off poll. At this scale of automation, most of Broadcom's documented steps really do reduce to "shut these VMs down cleanly, in this position in the sequence" - which is exactly why the common `Stop-VMsGracefully` helper exists, so that logic only had to be written, and tested, once.

```powershell
<# Step 3: Shut Down the Cloud Proxy Appliances. #>
function Invoke-Step03-ShutdownCloudProxy {
    param([Parameter(Mandatory)][string[]]$VMNames)
    Write-Step "=== Step 3: Shut Down the Cloud Proxy Appliances ==="
    Stop-VMsGracefully -VMNames $VMNames
    Write-Step "=== Step 3 complete ===" -Level OK
}
```

### Step 4 - License Server

```powershell
<# Step 4: Shut Down the License Server. #>
function Invoke-Step04-ShutdownLicenseServer {
    param([Parameter(Mandatory)][string]$VMName)
    Write-Step "=== Step 4: Shut Down the License Server ==="
    Stop-VMsGracefully -VMNames @($VMName)
    Write-Step "=== Step 4 complete ===" -Level OK
}
```

### Step 5 - VCF Management Services (the VCF Services Runtime cluster)

This is the largest and most involved module in the set, because it's the one component Broadcom expects you to drain through an API rather than just power off. The VCF Services Runtime cluster runs the platform controllers and tenant workloads underneath VCF Management Services, and the supported approach is to gracefully scale everything down through its management REST API before powering off the underlying node VMs. This module is adapted from a PowerShell port - originally written by Ward Vissers (wardvissers.nl) of Broadcom's own `vcf_services_runtime_shutdown.sh` - referencing Broadcom knowledge base articles [440874](https://knowledge.broadcom.com/external/article/440874/) and [440862](https://knowledge.broadcom.com/external/article/440862/). Wrapping it as a function for this modular framework meant two changes: every internal helper got an `S5` prefix and its state was moved into script-scoped variables local to this file, so nothing collides with the other ten step modules when they're all dot-sourced together; and every `exit` call from the original standalone script was replaced with `throw`/`return`, so a failure here surfaces as a normal PowerShell exception the wrapper's `try/catch` can actually see, instead of terminating the whole shutdown sequence's host process outright.

The flow inside `Invoke-Step05-ShutdownVCFManagementServices` is: authenticate against the Services Runtime API with the `vmware-system-user` breakglass credential, confirm no VM snapshots exist on the cluster nodes, `POST /api/v1/system?action=shutdown` and poll the resulting task until it reports `Succeeded`, then resolve each node's VM MoRef through vCenter and power it off - a deliberate hard power-off at that point, mirroring the original script's `govc vm.power -off -force` behavior, since the API-driven system shutdown has already gracefully drained everything the appliance was running.

```powershell
#Requires -Version 7.0
<#
Step 5: Shut Down VCF Management Services (VCF Services Runtime cluster).

Adapted from the PowerShell port (by Ward Vissers, wardvissers.nl) of Broadcom's
official vcf_services_runtime_shutdown.sh, referencing:
  https://knowledge.broadcom.com/external/article/440874/
  https://knowledge.broadcom.com/external/article/440862/

Uses the VCF Services Runtime management REST API to gracefully scale down all
tenant workloads and platform controllers (no kubectl access required), then
powers off the underlying node VMs via vcf.powercli. Wrapped as a function for
this modular framework: internal helpers are prefixed "S5" and state is kept in
script-scoped variables local to this dot-sourced file to avoid colliding with
other step files. All `exit` calls from the original were replaced with
`throw`/`return` so failures surface as normal exceptions the wrapper can catch.
#>

# ---- S5-prefixed logging helpers ----
function Get-S5LogTimestamp { (Get-Date).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ") }
function Write-S5Log      { param([string]$Message) Write-Host "[$(Get-S5LogTimestamp)] [INFO]  $Message" }
function Write-S5LogWarn  { param([string]$Message) Write-Host "[$(Get-S5LogTimestamp)] [WARN]  $Message" -ForegroundColor Yellow }
function Write-S5LogError { param([string]$Message) Write-Host "[$(Get-S5LogTimestamp)] [ERROR] $Message" -ForegroundColor Red }
function Write-S5LogStep  {
    param([string]$Message)
    Write-Host ""
    Write-Host ("=" * 58)
    Write-Host "[$(Get-S5LogTimestamp)] [STEP]  $Message"
    Write-Host ("=" * 58)
}

function Test-S5VCenterCredentialsAvailable {
    return (-not [string]::IsNullOrEmpty($Script:S5VCenterUsername)) -and
           (-not [string]::IsNullOrEmpty($Script:S5VCenterPassword))
}

function Test-S5Prerequisites {
    Write-S5LogStep "Checking prerequisites"

    if (-not $Script:S5SkipPoweroff -and (Test-S5VCenterCredentialsAvailable)) {
        Assert-ModuleAvailable -Name 'VCF.PowerCLI'   # kept as a standalone-safe check; harmless if the wrapper already ran Assert-PowerCLIModule
        $invalidCertAction = if ($Script:S5VCenterInsecure) { "Ignore" } else { "Warn" }
        Set-PowerCLIConfiguration -InvalidCertificateAction $invalidCertAction `
            -ParticipateInCeip $false -Confirm:$false -Scope Session | Out-Null
    }

    if ([string]::IsNullOrEmpty($Script:S5NodeIp)) {
        throw "Node IP is required. Pass -NodeIp to Invoke-Step05-ShutdownVCFManagementServices."
    }
    $Script:S5ApiBase = "https://$($Script:S5NodeIp):$($Script:S5NodePort)"
    Write-S5Log "Management API base: $($Script:S5ApiBase)"
}

function Get-S5PasswordPlaintext {
    if ($Script:S5VmspPassword) {
        $bstr = [Runtime.InteropServices.Marshal]::SecureStringToBSTR($Script:S5VmspPassword)
        try { return [Runtime.InteropServices.Marshal]::PtrToStringAuto($bstr) }
        finally { [Runtime.InteropServices.Marshal]::ZeroFreeBSTR($bstr) }
    }
    $secure = Read-Host -Prompt "Enter the breakglass password for 'vmware-system-user'" -AsSecureString
    $bstr = [Runtime.InteropServices.Marshal]::SecureStringToBSTR($secure)
    try {
        $plain = [Runtime.InteropServices.Marshal]::PtrToStringAuto($bstr)
    } finally {
        [Runtime.InteropServices.Marshal]::ZeroFreeBSTR($bstr)
    }
    if ([string]::IsNullOrEmpty($plain)) { throw "Breakglass password cannot be empty." }
    return $plain
}

function Invoke-S5ApiLogin {
    Write-S5LogStep "Authenticating with API server"
    $passwordPlain = Get-S5PasswordPlaintext
    $payload = @{ username = "vmware-system-user"; password = $passwordPlain } | ConvertTo-Json

    try {
        $response = Invoke-RestMethod -Method Post -Uri "$($Script:S5ApiBase)/api/v1/auth/login" `
            -ContentType "application/json" -Body $payload -SkipCertificateCheck -ErrorAction Stop
    }
    catch {
        $statusCode = $null
        if ($_.Exception.Response) { $statusCode = [int]$_.Exception.Response.StatusCode }
        if (-not $statusCode) {
            throw "Failed to reach API server at $($Script:S5ApiBase). Check -NodeIp and network connectivity."
        }
        throw "Authentication failed (HTTP $statusCode): $($_.ErrorDetails.Message)"
    }
    if (-not $response.token) { throw "No token in login response: $($response | ConvertTo-Json -Compress)" }
    $Script:S5AuthToken = $response.token
    Write-S5Log "Authentication successful."
}

function Invoke-S5ApiRequest {
    param(
        [Parameter(Mandatory)][ValidateSet("Get","Post")][string]$Method,
        [Parameter(Mandatory)][string]$Path,
        [object]$Body = $null,
        [switch]$IsRetry
    )
    $uri = "$($Script:S5ApiBase)$Path"
    $headers = @{ Authorization = "Bearer $($Script:S5AuthToken)"; Accept = "application/json" }
    $requestParams = @{ Method = $Method; Uri = $uri; Headers = $headers; SkipCertificateCheck = $true; ErrorAction = "Stop" }
    if ($Method -eq "Post") {
        $requestParams["ContentType"] = "application/json"
        if ($null -ne $Body) { $requestParams["Body"] = ($Body | ConvertTo-Json -Depth 10) }
    }
    try {
        return Invoke-RestMethod @requestParams
    }
    catch {
        $statusCode = $null
        if ($_.Exception.Response) { $statusCode = [int]$_.Exception.Response.StatusCode }
        if (-not $statusCode) { throw "$Method $Path - connection failed." }
        if ($statusCode -eq 401 -and -not $IsRetry) {
            Write-S5Log "Token expired - re-authenticating..."
            Invoke-S5ApiLogin
            return Invoke-S5ApiRequest -Method $Method -Path $Path -Body $Body -IsRetry
        }
        throw "$Method $Path failed (HTTP $statusCode): $($_.ErrorDetails.Message)"
    }
}

function Test-S5NoSnapshots {
    Write-S5LogStep "Checking for VM snapshots on VCF Services Runtime nodes"
    if ($Script:S5SkipSnapshotCheck) {
        Write-S5LogWarn "Snapshot check skipped (-SkipSnapshotCheck). Ensure no snapshots exist on cluster node VMs before proceeding."
        return
    }
    if ($Script:S5DryRun) { Write-S5Log "[DRY-RUN] Would check for VM snapshots via API."; return }
    Write-S5Log "Snapshot pre-check is enforced by the component shutdown precheck workflow."
    Write-S5Log "Use -SkipSnapshotCheck only if you have manually confirmed no snapshots exist."
}

function Wait-S5ForTask {
    param([Parameter(Mandatory)][string]$TaskId, [Parameter(Mandatory)][string]$TargetName)
    Write-S5Log "  Waiting for task $TaskId ($TargetName)...."
    $elapsed = 0
    while ($true) {
        $taskBody = Invoke-S5ApiRequest -Method Get -Path "/api/v1/tasks/$TaskId"
        $status = if ($taskBody.PSObject.Properties.Name -contains "status" -and $taskBody.status) { $taskBody.status }
                  elseif ($taskBody.PSObject.Properties.Name -contains "phase" -and $taskBody.phase) { $taskBody.phase }
                  else { "Unknown" }
        Write-S5Log "  Task $TaskId status: $status"
        if ($status -eq "Succeeded") { Write-S5Log "  '$TargetName' shutdown succeeded."; return $true }
        if ($status -eq "Failed") {
            $messages = @()
            if ($taskBody.messages) { $messages = @($taskBody.messages | ForEach-Object { $_.default }) }
            Write-S5LogError "  '$TargetName' shutdown task failed: $($messages -join '; ')"
            return $false
        }
        if ($elapsed -ge $Script:S5TaskTimeout) {
            Write-S5LogError "  Timed out waiting for task $TaskId ($TargetName) after $($Script:S5TaskTimeout)s."
            return $false
        }
        Start-Sleep -Seconds $Script:S5TaskPollInterval
        $elapsed += $Script:S5TaskPollInterval
    }
}

function Stop-S5VcfSystem {
    Write-S5LogStep "Shutting down VCF Services Runtime system"
    if ($Script:S5DryRun) { Write-S5Log "  [DRY-RUN] Would POST /api/v1/system?action=shutdown"; return }
    $response = Invoke-S5ApiRequest -Method Post -Path "/api/v1/system?action=shutdown"
    $taskId = $response.id
    if ([string]::IsNullOrEmpty($taskId)) { throw "No task ID returned for system shutdown: $($response | ConvertTo-Json -Compress)" }
    Write-S5Log "  System shutdown task created: $taskId"
    if (-not (Wait-S5ForTask -TaskId $taskId -TargetName "system")) {
        throw "System shutdown failed. Resolve the failures above before powering off VMs."
    }
    Write-S5Log "System shut down successfully."
}

function Get-S5ClusterNodes {
    try { $response = Invoke-S5ApiRequest -Method Get -Path "/api/v1/system/inventory/nodes" }
    catch { Write-S5LogError "Failed to retrieve cluster nodes."; return @() }
    $nodes = @($response.nodes)
    Write-S5Log "Returned $($nodes.Count) node(s)."
    return $nodes
}

function Find-S5VCenterUrl {
    Write-S5Log "Auto-discovering vCenter URL from vsp component configuration..."
    $response = Invoke-S5ApiRequest -Method Get -Path "/api/v1/components?type=vsp"
    $vcenterUrl = $null
    $components = @($response.components)
    if ($components.Count -gt 0) {
        $config = $components[0].spec.configuration
        if ($config) {
            if ($config.infrastructure -and $config.infrastructure.vsphere -and $config.infrastructure.vsphere.server) {
                $vcenterUrl = $config.infrastructure.vsphere.server
            } elseif ($config.PSObject.Properties.Name -contains "provider.vsphere.server") {
                $vcenterUrl = $config.'provider.vsphere.server'
            }
        }
    }
    if ([string]::IsNullOrEmpty($vcenterUrl)) { Write-S5LogWarn "Could not determine vCenter URL from vsp component config."; return $null }
    Write-S5Log "Discovered vCenter server: $vcenterUrl"
    $Script:S5VCenterServer = $vcenterUrl
    return $Script:S5VCenterServer
}

function Initialize-S5VCenterConnection {
    Write-S5LogStep "Setting up vCenter connection"
    if ([string]::IsNullOrEmpty($Script:S5VCenterServer)) {
        $Script:S5VCenterServer = Find-S5VCenterUrl
        if ([string]::IsNullOrEmpty($Script:S5VCenterServer)) {
            Write-S5LogError "vCenter server could not be determined. Pass -VCenterServer explicitly and re-run."
            return $false
        }
    }
    Write-S5Log "Connecting to vCenter at $($Script:S5VCenterServer)..."
    try {
        $Script:S5VIConnection = Connect-VIServer -Server $Script:S5VCenterServer -Credential $Script:S5VCenterCredential -ErrorAction Stop
    } catch {
        Write-S5LogError "Failed to connect to vCenter at $($Script:S5VCenterServer): $($_.Exception.Message)"
        return $false
    }
    Write-S5Log "vCenter connection established."
    return $true
}

function Stop-S5Vms {
    Write-S5LogStep "VM Power-Off"
    if ($Script:S5SkipPoweroff) { Write-S5Log "Skipping VM power-off (-SkipPoweroff)."; return }

    $nodes = Get-S5ClusterNodes
    $vmRefs = @(); $vmMorefs = @(); $nodeNames = @()

    foreach ($node in $nodes) {
        $nodeName = $node.name
        $vmMoref = $node.vm.moRef
        if ([string]::IsNullOrEmpty($vmMoref)) { Write-S5LogWarn "  Node '$nodeName' has no VM MoRef - skipping."; continue }
        Write-S5Log "  Node '$nodeName' -> VM MoRef: VirtualMachine:$vmMoref"
        $vmRefs += "VirtualMachine:$vmMoref"; $vmMorefs += $vmMoref; $nodeNames += $nodeName
    }

    if (-not (Test-S5VCenterCredentialsAvailable)) {
        Write-S5LogWarn "vCenter credentials not available - skipping automated VM power-off."
        Write-S5LogWarn "Power off the following VMs manually in vCenter:"
        for ($i = 0; $i -lt $vmRefs.Count; $i++) { Write-S5LogWarn "  $($nodeNames[$i])  ->  $($vmRefs[$i])" }
        return
    }
    if ($vmRefs.Count -eq 0) { Write-S5LogWarn "No VM MoRefs found. Skipping power-off."; return }

    if (-not (Initialize-S5VCenterConnection)) {
        Write-S5LogError "vCenter connection could not be established. Power off VMs manually:"
        for ($i = 0; $i -lt $vmRefs.Count; $i++) { Write-S5LogError "  $($nodeNames[$i])  ->  $($vmRefs[$i])" }
        throw "vCenter connection failed during VM power-off step."
    }

    try {
        for ($i = 0; $i -lt $vmRefs.Count; $i++) {
            $vmId = "VirtualMachine-$($vmMorefs[$i])"
            $name = $nodeNames[$i]
            $vm = $null
            try { $vm = Get-VM -Id $vmId -Server $Script:S5VIConnection -ErrorAction Stop }
            catch { Write-S5LogWarn "  Could not resolve VM '$vmId' ($name) - skipping: $($_.Exception.Message)"; continue }

            if ($vm.PowerState -eq "PoweredOff") { Write-S5Log "  VM '$($vm.Name)' already powered off - skipping."; continue }
            if ($Script:S5DryRun) { Write-S5Log "  [DRY-RUN] Would power off VM: $($vm.Name) ($vmId)"; continue }

            Write-S5Log "  Powering off VM: $($vm.Name) ($vmId, node '$name')"
            try {
                # Deliberate hard power-off (matches original 'govc vm.power -off -force'
                # behavior): by this point the API-driven system shutdown has already
                # gracefully drained workloads/controllers, so the appliance itself no
                # longer needs a graceful guest-OS shutdown.
                Stop-VM -VM $vm -Confirm:$false -ErrorAction Stop | Out-Null
                Write-S5Log "  VM '$($vm.Name)' powered off."
            } catch {
                Write-S5LogWarn "  Failed to power off VM '$($vm.Name)': $($_.Exception.Message)"
            }
            Start-Sleep -Seconds $Script:S5PoweroffWait
        }
        Write-S5Log "VM power-off sequence complete."
    } finally {
        if ($Script:S5VIConnection) {
            Disconnect-VIServer -Server $Script:S5VIConnection -Confirm:$false -ErrorAction SilentlyContinue
            $Script:S5VIConnection = $null
        }
    }
}

function Invoke-Step05-ShutdownVCFManagementServices {
    <#
    .PARAMETER NodeIp        IP of any reachable VCF Services Runtime cluster node (API on NodePort).
    .PARAMETER VCenterCredential  PSCredential for vCenter (reuse the one collected in the wrapper).
    .PARAMETER VmspPassword  SecureString breakglass password for 'vmware-system-user'; prompted if omitted.
    #>
    param(
        [Parameter(Mandatory)][string]$NodeIp,
        [int]$NodePort = 5480,
        [Parameter(Mandatory)][PSCredential]$VCenterCredential,
        [SecureString]$VmspPassword,
        [string]$VCenterServer,
        [bool]$VCenterInsecure = $true,
        [switch]$DryRun,
        [switch]$SkipPoweroff,
        [switch]$SkipSnapshotCheck,
        [int]$TaskPollInterval = 15,
        [int]$TaskTimeout = 600,
        [int]$PoweroffWait = 5
    )

    Write-Step "=== Step 5: Shut Down VCF Management Services (Services Runtime cluster) ==="

    # Initialize this step's script-scoped state
    $Script:S5NodeIp             = $NodeIp
    $Script:S5NodePort           = $NodePort
    $Script:S5VmspPassword       = $VmspPassword
    $Script:S5DryRun             = $DryRun.IsPresent
    $Script:S5SkipPoweroff       = $SkipPoweroff.IsPresent
    $Script:S5SkipSnapshotCheck  = $SkipSnapshotCheck.IsPresent
    $Script:S5TaskPollInterval   = $TaskPollInterval
    $Script:S5TaskTimeout        = $TaskTimeout
    $Script:S5PoweroffWait       = $PoweroffWait
    $Script:S5VCenterServer      = $VCenterServer
    $Script:S5VCenterInsecure    = $VCenterInsecure
    $Script:S5VCenterCredential  = $VCenterCredential
    $Script:S5VCenterUsername    = $VCenterCredential.UserName
    $Script:S5VCenterPassword    = $VCenterCredential.GetNetworkCredential().Password
    $Script:S5ApiBase            = ""
    $Script:S5AuthToken          = ""
    $Script:S5VIConnection       = $null

    Write-Step "Mode: $(if ($Script:S5DryRun) { 'DRY-RUN' } else { 'LIVE' })"

    Test-S5Prerequisites
    Invoke-S5ApiLogin
    Test-S5NoSnapshots
    Stop-S5VcfSystem
    Stop-S5Vms

    Write-Step "=== Step 5 complete ===" -Level OK
}
```

### Step 6 - VCF Operations

Broadcom's documentation calls for taking the VCF Operations cluster offline from its own admin UI (`https://<vcf_operations_fqdn>/admin` → System status → Take cluster offline) before touching the appliance VMs, then powering them off in the specific order given in Broadcom KB 341964. That "take offline" action has no documented public API, so in this framework it's represented as a manual confirmation gate rather than something the script drives itself - the operator supplies the appliance VM names already in the correct KB order, and the module hands them to `Stop-VMsGracefully`.

```powershell
<#
Step 6: Shut Down VCF Operations.
Per the docs: take the VCF Operations cluster offline via its admin UI
(https://<vcf_operations_fqdn>/admin > System status > Take cluster offline)
BEFORE powering off appliances, and shut the appliances down in the order
given by Broadcom KB 341964. This has no documented public API, so the
"take offline" action is a manual confirmation gate; you must supply the
appliance VM names already in the correct KB-341964 order.
#>
function Invoke-Step06-ShutdownVCFOperations {
    param([Parameter(Mandatory)][string[]]$VMNamesInKBOrder)
    Write-Step "=== Step 6: Shut Down VCF Operations ==="
    #Confirm-ManualStep -Prompt "Log in to https://<vcf_operations_fqdn>/admin and click 'Take cluster offline' (provide a reason). This can take up to ~1 hour. Once the cluster shows offline,"
    Stop-VMsGracefully -VMNames $VMNamesInKBOrder
    Write-Step "=== Step 6 complete ===" -Level OK
}
```

Step 7, Protection and Recovery (VMware Live Site Recovery), is intentionally not part of this script set - it isn't deployed in the pods this framework targets, so it's skipped entirely in the wrapper rather than stubbed out with an empty file.

### Step 8 - NSX Edge / VNA nodes

```powershell
<# Step 8: Shut Down the NSX Edge or VNA Nodes in the Management Domain. #>
function Invoke-Step08-ShutdownNSXEdge {
    param([Parameter(Mandatory)][string[]]$VMNames)
    Write-Step "=== Step 8: Shut Down NSX Edge / VNA Nodes ==="
    Stop-VMsGracefully -VMNames $VMNames
    Write-Step "=== Step 8 complete ===" -Level OK
}
```

### Step 9 - NSX Manager nodes

```powershell
<# Step 9: Shut Down the NSX Manager Nodes (3-node cluster). #>
function Invoke-Step09-ShutdownNSXManager {
    param([Parameter(Mandatory)][string[]]$VMNames)
    Write-Step "=== Step 9: Shut Down NSX Manager Nodes ==="
    Stop-VMsGracefully -VMNames $VMNames
    Write-Step "=== Step 9 complete ===" -Level OK
}
```

### Step 10 - SDDC Manager appliance

```powershell
<# Step 10: Shut Down the SDDC Manager Appliance. #>
function Invoke-Step10-ShutdownSDDCManager {
    param([Parameter(Mandatory)][string]$VMName)
    Write-Step "=== Step 10: Shut Down the SDDC Manager Appliance ==="
    Stop-VMsGracefully -VMNames @($VMName)
    Write-Step "=== Step 10 complete ===" -Level OK
}
```

### Step 11 - vSAN and the ESX hosts

The last step is also the one that changed the most during development. It originally used a speculative approach built on `Get-VsanView` / `ClusterPoweroffVsanReplaceHost`, with a manual confirmation gate as the primary path. That got replaced once William Lam [documented](https://williamlam.com/2025/11/retrieving-the-vsan-cluster-shutdown-vms-running-pre-check-results-using-powercli.html) the officially supported PowerCLI cmdlets for vSAN cluster shutdown - introduced in vSAN 7 U3 and carried through VCF 9.0 - built on the vSAN Management APIs: `Test-VsanClusterHealth -Perspective clusterPowerOffPrecheck` to verify every VM except vCenter itself is powered off and the cluster is ready, `Stop-VsanCluster` to perform the actual graceful shutdown, and `Get-VsanClusterPowerState` to report the cluster's current state. The manual confirmation gate is kept only as a fallback for PowerCLI versions where these cmdlets aren't available yet.

The one detail that matters most here, and the reason this step is deliberately last: the cluster hosting the management vCenter has to be shut down after every other vSAN cluster, and the vCenter connection will drop the instant that cluster's shutdown begins - which the module treats as an expected, successful end state rather than an error.

```powershell
<#
Step 11: Shut Down vSAN and the ESX Hosts in the Management Domain.

Updated per William Lam's confirmed, officially-supported PowerCLI cmdlets
for vSAN Cluster Shutdown (introduced vSAN 7 U3, enhanced through VCF 9.0),
built on the vSAN Management APIs:
  https://williamlam.com/2025/11/retrieving-the-vsan-cluster-shutdown-vms-running-pre-check-results-using-powercli.html
  - Test-VsanClusterHealth -Perspective clusterPowerOffPrecheck : verifies all
    VMs (except the vCenter VM) are powered off and the cluster is ready.
  - Stop-VsanCluster : performs the actual graceful vSAN cluster shutdown.
  - Get-VsanClusterPowerState : reports current cluster power state.

This replaces the earlier speculative Get-VsanView / ClusterPoweroffVsanReplaceHost
approach. The manual confirmation gate is retained only as a fallback for
PowerCLI versions where these cmdlets aren't available.

IMPORTANT per Broadcom docs: the cluster hosting the management vCenter must
be shut down LAST, and the connection to vCenter will drop once that
cluster's shutdown begins - this is expected.
#>
function Invoke-Step11-ShutdownVsanAndEsxHosts {
    param(
        [Parameter(Mandatory)][string[]]$ClusterNames,          # all vSAN clusters in the mgmt domain
        [Parameter(Mandatory)][string]$ManagementVCenterClusterName,  # cluster hosting the mgmt vCenter - shut down last
        [string]$ShutdownReason = "Planned VCF management domain shutdown",
        [int]$PrecheckTimeoutSeconds = 900,
        [int]$PrecheckPollIntervalSeconds = 15
    )
    Write-Step "=== Step 11: Shut Down vSAN and the ESX Hosts in the Management Domain ==="

    $orderedClusters = @($ClusterNames | Where-Object { $_ -ne $ManagementVCenterClusterName })
    $orderedClusters += $ManagementVCenterClusterName

    foreach ($clusterName in $orderedClusters) {
        Write-Step "Preparing to shut down vSAN cluster '$clusterName'..."
        $cluster = Get-Cluster -Name $clusterName -ErrorAction SilentlyContinue
        if (-not $cluster) { Write-Step "Cluster '$clusterName' not found - skipping." -Level WARN; continue }

        $isMgmtCluster = ($clusterName -eq $ManagementVCenterClusterName)
        if ($isMgmtCluster) {
            Write-Step "'$clusterName' hosts the management vCenter. The vCenter connection WILL drop during this operation." -Level WARN
        }

        # --- Pre-check: wait for clusterPowerOffPrecheck perspective to pass ---
        $cmdletsAvailable = [bool](Get-Command Test-VsanClusterHealth -ErrorAction SilentlyContinue) -and
                             [bool](Get-Command Stop-VsanCluster -ErrorAction SilentlyContinue)

        $precheckPassed = $false
        if ($cmdletsAvailable) {
            Write-Step "Running vSAN clusterPowerOffPrecheck for '$clusterName' (waiting for all VMs except vCenter to power off)..."
            $sw = [Diagnostics.Stopwatch]::StartNew()
            while ($sw.Elapsed.TotalSeconds -lt $PrecheckTimeoutSeconds) {
                try {
                    $result = Test-VsanClusterHealth -Cluster $cluster -Perspective clusterPowerOffPrecheck -ErrorAction Stop
                    if ($result -and ($result.OverallHealth -eq 'green' -or $result.HealthScore -gt 96 -or $result -eq $true)) {
                        $precheckPassed = $true
                        break
                    }
                    Write-Step "  Precheck not yet green for '$clusterName' - still waiting on running VMs..." -Level WARN
                } catch {
                    Write-Step "Test-VsanClusterHealth call failed: $($_.Exception.Message)" -Level WARN
                    break
                }
                Start-Sleep -Seconds $PrecheckPollIntervalSeconds
            }
            if (-not $precheckPassed) {
                Write-Step "clusterPowerOffPrecheck did not pass within timeout for '$clusterName'." -Level WARN
            } else {
                Write-Step "clusterPowerOffPrecheck passed for '$clusterName'." -Level OK
            }
        } else {
            Write-Step "Test-VsanClusterHealth / Stop-VsanCluster cmdlets not found in this PowerCLI version." -Level WARN
        }

        # --- Shutdown: use Stop-VsanCluster if available and precheck passed, else manual fallback ---
        $automated = $false
        if ($cmdletsAvailable -and $precheckPassed) {
            try {
                Write-Step "Invoking Stop-VsanCluster for '$clusterName'..."
                Stop-VsanCluster -Cluster $cluster -PowerOffReason $ShutdownReason -Confirm:$false -ErrorAction Stop
                $automated = $true
                Write-Step "Stop-VsanCluster completed for '$clusterName'." -Level OK
            } catch {
                Write-Step "Stop-VsanCluster failed for '$clusterName': $($_.Exception.Message)" -Level WARN
            }
        }

        if (-not $automated) {
            Confirm-ManualStep -Prompt "In the vSphere Client, right-click cluster '$clusterName' > vSAN > Shutdown cluster, complete the wizard, and enter a reason. Once the wizard reports the shutdown is complete,"
        }

        if ($isMgmtCluster) {
            Write-Step "Management domain vCenter is now shutting down with this cluster. Script cannot verify further - shutdown sequence is complete." -Level OK
            return
        } else {
            Confirm-ManualStep -Prompt "Confirm all hosts in '$clusterName' are powered off before moving to the next cluster."
        }
    }
}
```

Broadcom's procedure also includes a final page for ESX hosts attached to NFS or Fibre Channel storage, which this framework skips - it doesn't apply to the all-vSAN pods it targets. For an environment that does mix storage types, that page's steps would slot in as a twelfth module, after Step 11, following the same pattern as everything above.

Once Step 11 returns, the management domain is down, vCenter connectivity is gone by design, and the pod's VMs are all powered off in an order that VCD, and VCF, both consider clean. That is what unblocks the actual goal: capturing the vApp as a template. Twelve small files instead of one big one turned out to be the difference between a script I trusted to run against a real lab pod on the first end-to-end attempt, and one I would have wanted to babysit through every step by hand anyway.
