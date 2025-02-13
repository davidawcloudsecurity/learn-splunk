## Troubleshooting
### Disable mgmt
```bash
# Disable management port to prevent remote (or local) config.
[httpServer]
disableDefaultPort = true
```
### Loop splunkd.log
```powershell
while ($true) {
    Clear-Host
    Get-Content "C:\program files\SplunkUniversalForwarder\var\log\splunk\splunkd.log" -tail 10
    Start-Sleep -Seconds 2
}
```
### Windows
```bash
C:\"Program Files"\SplunkUniversalForwarder\bin\splunk.exe restart
C:\"Program Files"\SplunkUniversalForwarder\bin\splunk.exe list forward-server
```
Network
```bash
netstat -ano | findstr ":8000"
netstat -ano | findstr ":8089"
```
### Outputs
```bash
[tcpout]
defaultGroup = default-autolb-group

[tcpout:default-autolb-group]
server = 192.168.12.200:9997

[tcpout-server://192.168.12.200:9997]
```
### Inputs
```bash
###### OS Logs ######
[WinEventLog://Application]
disabled = 0
index=<changeme>
start_from = oldest
current_only = 0
checkpointInterval = 5
renderXml=0
host=<changeme>

[WinEventLog://Security]
disabled = 0
index=<changeme>
start_from = oldest
current_only = 0
evt_resolve_ad_obj = 1
checkpointInterval = 5
blacklist1 = EventCode="4662" Message="Object Type:(?!\s*groupPolicyContainer)"
blacklist2 = EventCode="566" Message="Object Type:(?!\s*groupPolicyContainer)"
renderXml=0
host=<changeme>

[WinEventLog://System]
disabled = 0
index=<changeme>
start_from = oldest
current_only = 0
checkpointInterval = 5
renderXml=0
host=<changeme>
```
## 🔍 Splunk btool Debugging Commands
### 1️⃣ Check Active Forwarding Configuration
```powershell
& "C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" btool outputs list --debug
```
✅ This shows which indexers (receivers) the Universal Forwarder (UF) is sending logs to.
```bash
Look for:

server = <indexer>:9997
disabled = false
```
### 2️⃣ Check Inputs Configuration (Log Collection)
```powershell
& "C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" btool inputs list --debug
```
✅ This helps confirm if Splunk is listening for logs from event logs, files, or other sources.
### 3️⃣ Check Which Config File a Setting Comes From
If you're unsure where a setting is being defined, use:
```powershell
& "C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" btool outputs list --debug | findstr server
```
✅ This shows the exact file where an indexer is configured.
### 4️⃣ Check All Active Configurations
```powershell
& "C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" btool check --debug
```
✅ This scans for misconfigurations or missing settings.
### 5️⃣ Find Configurations Overwritten by Other Files
If a setting is being overridden, use:
```powershell
& "C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" btool <config_file> list --debug
```
Example:
```powershell
& "C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" btool props list --debug
```
✅ This shows how settings are inherited across different .conf files.

### 🛠 Example Troubleshooting Workflow
Check forwarder status
```powershell
& "C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" list forward-server
```
Check outputs (indexer connections)
```powershell
& "C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" btool outputs list --debug
```
Verify inputs (log sources)
```powershell
& "C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" btool inputs list --debug
```
Check logs for errors
```powershell
Get-Content "C:\Program Files\SplunkUniversalForwarder\var\log\splunk\splunkd.log" -Tail 50 -Wait
```
Restart Splunk with Debugging
```powershell
& "C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" restart --debug
```
## Inputs.conf
📄 Where to Find the Configuration?

1️⃣ Check in Local Inputs Configuration
```plaintext
C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf
```
or
```plaintext
C:\Program Files\SplunkUniversalForwarder\etc\apps\SplunkUniversalForwarder\local\inputs.conf
```
or
```plaintext
C:\Program Files\SplunkUniversalForwarder\etc\apps\search\local\inputs.conf
```
🔎 To verify, run:
```powershell
& "C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" btool inputs list --debug | findstr vizqlserver
```
✅ This will show the exact file where this monitoring input is stored.

## Understanding between local and apps for inputs.conf/output.conf
#### apps\search\local\inputs.conf
🔹 Scope:

This file is part of the Search app (search).
It affects only searches and dashboards within the Search app.
Used for app-specific configurations (e.g., setting up monitoring for logs specific to the Search app).

🔹 Usage:

If an input is defined here, it applies only when using the Search app.
If another app has its own inputs.conf, this app’s settings won’t override them unless it has higher precedence.

#### etc\system\local\inputs.conf
🔹 Scope:

This is a global configuration file for Splunk.
Highest precedence—settings here override all other inputs.conf files.
Used for system-wide configurations (e.g., monitoring log files that all apps should see).

🔹 Usage:

If an input is defined here, it overrides inputs from any app (apps\search, apps\SplunkUniversalForwarder, etc.).
Typically used for global log monitoring and universal configurations.

### Powershell command to add the following to inputs.conf in local
```bash
# Define the Tableau log paths to check
$tableauLogPaths = @(
    "C:\ProgramData\Tableau\Tableau Server\logs",
    "C:\ProgramData\Tableau\Tableau Server\data\tabsvc\logs",
    "C:\ProgramData\Tableau\Tableau Server\data\tabsvc\vizqlserver\Logs"
)

# Define the SFTP log path
$sftpLogPath = "C:\ProgramData\ssh\logs\"

# Define the Windows Event Logs to check
$windowsEventLogs = @("Application", "System", "Security")

# Define the Splunk Universal Forwarder service name
$splunkServiceName = "SplunkForwarder"

# Define the path to inputs.conf
$inputsConfPath = "C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf"

# Initialize the content for inputs.conf
$inputsConfContent = @()

# Get the hostname from the environment
$hostname = $env:COMPUTERNAME

# Function to check if a directory exists
function Test-DirectoryExists {
    param (
        [string]$path
    )
    if (Test-Path -Path $path) {
        Write-Host "Directory exists: $path"
        return $true
    } else {
        Write-Host "Directory does not exist: $path"
        return $false
    }
}

# Function to check if log files exist in a directory
function Test-LogFilesExist {
    param (
        [string]$path
    )
    if (Test-Path -Path "$path\*.log") {
        Write-Host "Log files exist in: $path"
        return $true
    } else {
        Write-Host "No log files found in: $path"
        return $false
    }
}

# Function to write to inputs.conf
function Write-InputsConf {
    param (
        [string]$path,
        [string]$content
    )
    try {
        Write-Host "Writing configuration to $path..."
        Set-Content -Path $path -Value $content
        Write-Host "Configuration written successfully."
    } catch {
        Write-Host "Failed to write configuration: $_"
    }
}

# Function to restart the Splunk Universal Forwarder service
function Restart-SplunkService {
    param (
        [string]$serviceName
    )
    try {
        Write-Host "Restarting Splunk Universal Forwarder service..."
        Restart-Service -Name $serviceName -Force
        Write-Host "Splunk Universal Forwarder service restarted successfully."
    } catch {
        Write-Host "Failed to restart Splunk Universal Forwarder service: $_"
    }
}

# Validate Tableau log directories
foreach ($tableauLogPath in $tableauLogPaths) {
    if (Test-DirectoryExists -path $tableauLogPath) {
        $inputsConfContent += @"
[monitor://$tableauLogPath]
sourcetype = tableau_server
index = tableau_index
host = $hostname
disabled = false

"@
    } else {
        Write-Host "Skipping Tableau logs: Directory does not exist - $tableauLogPath"
    }
}

# Validate SFTP log directory and files
if (Test-DirectoryExists -path $sftpLogPath) {
    if (Test-LogFilesExist -path $sftpLogPath) {
        $inputsConfContent += @"
[monitor://$sftpLogPath]
sourcetype = sftp_logs
index = sftp_index
host = $hostname
disabled = false

"@
    } else {
        Write-Host "Skipping SFTP logs: No log files found in $sftpLogPath"
    }
} else {
    Write-Host "Skipping SFTP logs: Directory does not exist - $sftpLogPath"
}

# Add Windows Event Logs configuration
foreach ($log in $windowsEventLogs) {
    if (Get-WinEvent -ListLog $log -ErrorAction SilentlyContinue) {
        $inputsConfContent += @"
[WinEventLog://$log]
index = windows_index
host = $hostname
disabled = false
evt_resolve_ad_obj = 0

"@
    } else {
        Write-Host "Skipping Windows Event Log: $log does not exist"
    }
}

# Check if there is any configuration to write
if ($inputsConfContent.Length -gt 0) {
    Write-Host "Writing configuration to inputs.conf..."
    Write-InputsConf -path $inputsConfPath -content $inputsConfContent

    # Restart Splunk Universal Forwarder to apply changes
    Restart-SplunkService -serviceName $splunkServiceName
} else {
    Write-Host "No valid configurations to write. Splunk will not be restarted."
}
```
