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

# Validate Tableau log directories and files
foreach ($tableauLogPath in $tableauLogPaths) {
    if (Test-DirectoryExists -path $tableauLogPath) {
        if (Test-LogFilesExist -path $tableauLogPath) {
            $inputsConfContent += @"
[monitor://$tableauLogPath]
sourcetype = tableau_server
index = tableau_index
host = $hostname
disabled = false

"@
        } else {
            Write-Host "Skipping Tableau logs: No log files found in $tableauLogPath"
        }
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
