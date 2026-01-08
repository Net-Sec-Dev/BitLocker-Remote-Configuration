# Enable BitLocker via TPM and export recovery key using manage-bde
# Made for PDQ Deploy and Ansible. Requires admin or SYSTEM privleges.
# -----------------------------------------------------------

# ------------------- Configuration -------------------
$MountPoint = "C:"                           # Drive to encrypt - default = C:
$EncryptionMethod = "XtsAes256"              # Encryption strength - default = XtsAes256
$ExportPath = "//network/path"               # Network folder to save recovery key
$Version = "Enable Bitlocker Remote v1.0"    # Script name and version number (placed in log file along with recovery key) - default = Enable Bitlocker Remote v1.0                  

# ------------------- Pre-checks ----------------------
Write-Host "Starting BitLocker enablement on $MountPoint..."

# Ensure export path exists and is reachable
if (-not (Test-Path $ExportPath)) {
    Write-Error "The export path '$ExportPath' is not accessible. Exiting."
    exit 77
}

# Check BitLocker status
try {
    $BLV = Get-BitLockerVolume -MountPoint $MountPoint -ErrorAction Stop
} catch {
    Write-Error "Unable to retrieve BitLocker volume information: $($_.Exception.Message)"
    exit 88
}

if ($BLV.ProtectionStatus -eq "On") {
    Write-Error "BitLocker is already enabled on $MountPoint."
    exit 99
}

# ------------------- Enable BitLocker -----------------
Write-Host "Checking for existing TPM protector..."
$ExistingTpm = $BLV.KeyProtector | Where-Object { $_.KeyProtectorType -eq 'Tpm' }

if (-not $ExistingTpm) {
    Write-Host "No TPM protector found. Enabling BitLocker encryption with TPM protector..."
    try {
        Enable-BitLocker -MountPoint $MountPoint `
                         -EncryptionMethod $EncryptionMethod `
                         -UsedSpaceOnly `
                         -TpmProtector `
                         -SkipHardwareTest
    } catch [System.ArgumentException] {
        Write-Warning "BitLocker returned a non-critical ArgumentException, encryption likely started successfully. Continuing..."
    } catch {
        Write-Error "Failed to enable BitLocker: $($_.Exception.Message)"
        exit 44
    }
} else {
    Write-Host "TPM protector already exists. Skipping Enable-BitLocker step."
    if ($BLV.ProtectionStatus -eq "Off") {
        Write-Host "Resuming BitLocker encryption..."
        try {
            Resume-BitLocker -MountPoint $MountPoint
        } catch {
            Write-Error "Failed to resume BitLocker encryption: $($_.Exception.Message)"
            exit 45
        }
    }
}

# Wait a few seconds for TPM protector to initialize
Start-Sleep -Seconds 5

# ------------------- Add Recovery Protector -----------------
Write-Host "Adding recovery password protector..."
try {
    $RecoveryProtector = Add-BitLockerKeyProtector -MountPoint $MountPoint -RecoveryPasswordProtector -ErrorAction Stop
} catch {
    Write-Error "Failed to add recovery password protector: $($_.Exception.Message)"
    exit 55
}

# Wait for protector to register
Start-Sleep -Seconds 5

# ------------------- Ensure BitLocker is active -----------------
$BLV = Get-BitLockerVolume -MountPoint $MountPoint
if ($BLV.ProtectionStatus -eq "Off") {
    Write-Host "BitLocker is suspended. Resuming protection..."
    try {
        Resume-BitLocker -MountPoint $MountPoint
    } catch {
        Write-Error "Failed to resume BitLocker: $($_.Exception.Message)"
        exit 45
    }
} else {
    Write-Host "BitLocker protection is active."
}

# ------------------- Export Recovery Key -----------------
Write-Host "Exporting recovery key via manage-bde..."
try {
    $output = & manage-bde -protectors -get $MountPoint | Out-String
} catch {
    Write-Error "Failed to query manage-bde output: $($_.Exception.Message)"
    exit 66
}

# ------------------- Parse Key Data -----------------
# Extract recovery password
$RecoveryPassword = ($output | Select-String -Pattern "([0-9]{6}-){7}[0-9]{6}").Matches.Value

# Extract protector GUID
$ProtectorGUIDMatch = ($output | Select-String -Pattern "ID:\s*({[0-9A-Fa-f-]+})")
if ($ProtectorGUIDMatch) {
    $ProtectorGUID = $ProtectorGUIDMatch.Matches.Groups[1].Value
} else {
    $ProtectorGUID = "N/A"
}

# Extract GUI-style Numerical Password Identifier
$Lines = $output -split "`r?`n"
$Identifier = $null
for ($i = 0; $i -lt $Lines.Count; $i++) {
    if ($Lines[$i] -match "Numerical Password") {
        for ($j = $i + 1; $j -lt $Lines.Count; $j++) {
            if ($Lines[$j] -match "ID:\s*({[0-9A-Fa-f-]+})") {
                $Identifier = $Matches[1]
                $IdentifierSource = "GUI Identifier"
                break
            }
        }
        if ($Identifier) { break }
    }
}

# Fallback if GUI-style identifier not found
if (-not $Identifier) {
    $Identifier = $ProtectorGUID
    $IdentifierSource = "Fallback (Protector GUID)"
}

if (-not $RecoveryPassword -or -not $ProtectorGUID) {
    Write-Error "Could not parse recovery information from manage-bde output."
    exit 33
}

# ------------------- Save to File -----------------
# Filename: <HOSTNAME>, <GUI-Identifier>, <DATE>.txt
$ComputerName = $env:COMPUTERNAME
$Date = (Get-Date).ToString("yyyy-MM-dd")
$FileName = "$ComputerName, $Identifier, $Date.txt"
$TargetFile = Join-Path $ExportPath $FileName

try {
    $lines = @(
        "Computer Name: $ComputerName"
        "Drive: $MountPoint"
        "Protector ID: $ProtectorGUID"
        "Identifier: $Identifier ($IdentifierSource)"
        "Recovery Password: $RecoveryPassword"
        "Export Date: $Date"
        "Script that made this file: $Version"
    )
    $lines | Out-File -FilePath $TargetFile -Encoding ASCII
    Write-Host "Recovery key exported successfully to $TargetFile"
} catch {
    Write-Error "Failed to write recovery key file: $($_.Exception.Message)"
    exit 22
}

# ------------------- Status Output -----------------
Start-Sleep -Seconds 10
$EncStatus = (Get-BitLockerVolume -MountPoint $MountPoint).EncryptionPercentage
Write-Host "Encryption started. Progress: $EncStatus%"

Write-Host "BitLocker enablement and recovery key export completed successfully for $ComputerName."
exit 0
