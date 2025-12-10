# Enable BitLocker via TPM with Recovery Key Export

> A simple, automated PowerShell script to enable BitLocker encryption on a drive using TPM and safely export the recovery key for backup. Designed for use with deployment tools like **PDQ Deploy**. It is highly recommended to use a second script before this one to ensure the drive paths are writable.

---

## Overview

This script helps you:  
- Enable **BitLocker encryption** on a chosen drive (default `C:`).  
- Add a **TPM protector** if one doesn’t exist.  
- Add a **recovery password protector**.  
- Export the recovery key to a **network location** with metadata for easy identification.  
- Integrate smoothly with automated deployments using **exit codes** for error handling.  

---

## How It Works

Here’s the high-level workflow of the script:

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
