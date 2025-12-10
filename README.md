# Enable BitLocker via TPM with Recovery Key Export

A simple, automated PowerShell script to enable BitLocker encryption on a drive using TPM and safely export the recovery key for backup. Designed for use with deployment tools like **Ansible** and **PDQ Deploy**. Note, the drive paths specified must be accessible and writable by whatever service account is being used to run this script.

## Overview

This script will:  
- Enable **BitLocker encryption** on a chosen drive (default `C:`).  
- Add a **TPM protector** if one doesn’t exist.  
- Add a **recovery password protector**.  
- Export the recovery key to a **network location** with metadata for easy identification.  
- Integrate smoothly with automated deployments using **exit codes** for error handling.  

## Configuration

$MountPoint = "C:"                      # Drive to encrypt <br />
$EncryptionMethod = "XtsAes256"         # Encryption strength<br />
$ExportPath = "\\network\share"         # Folder to save recovery key<br />
$Version = "Enable Bitlocker v2.1 PDQ"  # Script version<br />

## Exit Codes

0	BitLocker enabled and recovery key exported.<br />
99	BitLocker is already enabled on the drive.<br />

22	Failed to write the recovery key file (check permissions/network path).<br />
33	Could not parse recovery information from manage-bde.<br />
44	Failed to enable BitLocker.<br />
45	Failed to resume BitLocker encryption (if suspended).<br />
55	Failed to add recovery password protector.<br />
66	Failed to query manage-bde output.<br />
77	Export path does not exist or is unreachable.<br />
88	Unable to retrieve BitLocker volume information.<br />

## How It Works
```text
Start
 │
 │ Check if export path exists
 │       └─> Exit 77 if not reachable
 │
 │ Check BitLocker status on target drive
 │       └─> Exit 88 if drive info cannot be retrieved
 │       └─> Exit 99 if BitLocker already enabled
 │
 │ Check for TPM protector
 │       └─> If missing, enable BitLocker with TPM
 │            └─> Exit 44 on failure
 │       └─> If present and encryption suspended, resume encryption
 │            └─> Exit 45 on failure
 │
 │ Add recovery password protector
 │       └─> Exit 55 on failure
 │
 │ Query BitLocker info via manage-bde
 │       └─> Exit 66 on failure
 │
 │ Parse recovery info (password, GUID, identifier)
 │       └─> Exit 33 if parsing fails
 │
 │ Export recovery key to file
 │       └─> Exit 22 on failure
 │
 │ Check encryption progress and report
 │
End ──> Exit 0 (Success)
