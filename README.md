```bash
# Define the paths to check
$tableauLogPaths = @(
    "C:\ProgramData\Tableau\Tableau Server\logs",
    "C:\ProgramData\Tableau\Tableau Server\data\tabsvc\logs\",
    "C:\ProgramData\Tableau\Tableau Server\data\tabsvc\vizqlserver\Logs\"
)
$sftpLogPath = "C:\ProgramData\ssh\logs\"

# Define the Splunk Universal Forwarder service name
$splunkServiceName = "SplunkForwarder"

# Define the path to inputs.conf
$inputsConfPath = "C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf"

# Get the hostname from the environment
$hostname = $env:COMPUTERNAME

# Initialize the content for inputs.conf
$inputsConfContent = @()

# Function to check if a directory exists
function Test-DirectoryExists {
    param ([string]$path)
    return Test-Path -Path $path
}

# Function to check if log files exist in a directory
function Test-LogFilesExist {
    param ([string]$path)
    return (Get-ChildItem -Path $path -Filter "*.log" -ErrorAction SilentlyContinue).Count -gt 0
}

# Function to write to inputs.conf
function Write-InputsConf {
    param ([string]$path, [string]$content)
    try {
        Write-Host "Writing configuration to $path..."
        Set-Content -Path $path -Value $content -Force
        Write-Host "Configuration written successfully."
    } catch {
        Write-Host "Failed to write configuration: $_"
    }
}

# Function to restart the Splunk Universal Forwarder service
function Restart-SplunkService {
    param ([string]$serviceName)
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
    if (Test-DirectoryExists -path $tableauLogPath -and Test-LogFilesExist -path $tableauLogPath) {
        $inputsConfContent += @"
[monitor://$tableauLogPath*.log]
sourcetype = tableau_server
index = tableau_index
host = $hostname
disabled = false
"@
    } else {
        Write-Host "Skipping Tableau logs: No log files found in $tableauLogPath"
    }
}

# Validate SFTP log directory and files
if (Test-DirectoryExists -path $sftpLogPath -and Test-LogFilesExist -path $sftpLogPath) {
    $inputsConfContent += @"
[monitor://$sftpLogPath*.log]
sourcetype = sftp_logs
index = sftp_index
host = $hostname
disabled = false
"@
} else {
    Write-Host "Skipping SFTP logs: No log files found in $sftpLogPath"
}

# Check if there is any configuration to write
if ($inputsConfContent.Length -gt 0) {
    Write-InputsConf -path $inputsConfPath -content $inputsConfContent
    Restart-SplunkService -serviceName $splunkServiceName
} else {
    Write-Host "No valid configurations to write. Splunk will not be restarted."
}
```
