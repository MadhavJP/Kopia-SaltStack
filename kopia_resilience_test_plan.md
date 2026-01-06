# Kopia Backup Resilience Test Plan

This document outlines test cases to validate the robustness, automation, and recovery capabilities of the SaltStack-deployed Kopia backup solution.

## Test Environment Prerequisites
- **Salt Master**: Configured with updated Pillar (`repo_username`) and States.
- **Windows Minion**: Online, with Salt Minion service running.
- **MinIO/S3 Target**: Accessible from the Minion.
- **Tools**: PowerShell (admin), Salt CLI.

---

## Section 1: Deployment & Configuration Verification

| ID | Test Case | Action / Command | Expected Result |
|----|-----------|------------------|-----------------|
| **1.1** | **Environment Variables** | `salt 'win-minion' cmd.run 'gci env:KOPIA*'` | Should list `KOPIA_S3_ENDPOINT`, `KOPIA_S3_ACCESS_KEY`, `KOPIA_REPO_USERNAME`, etc. Values must be correct. |
| **1.2** | **Binary & Scripts** | `salt 'win-minion' cmd.run 'ls C:\ProgramData\kopia\scripts'` | All scripts (`connect.ps1`, `reconnect.ps1`, `run-backup.ps1`, etc.) must exist. |
| **1.3** | **Scheduled Task Registration** | `salt 'win-minion' cmd.run 'Get-ScheduledTask -TaskName "Kopia*"'` | Should show tasks "Kopia Daily Backup" (or similar) and "Kopia Weekly Maintenance" with status `Ready`. |
| **1.4** | **Initial Connection** | Check `C:\ProgramData\kopia\logs` or run `kopia-status.ps1` | Output should show "Connected to repository" and list policy details. |

---

## Section 2: Backup Functionality

| ID | Test Case | Action / Command | Expected Result |
|----|-----------|------------------|-----------------|
| **2.1** | **On-Demand Backup** | `salt 'win-minion' cmd.run 'powershell -File C:\ProgramData\kopia\scripts\on-demand-backup.ps1 -Path C:\Users\Public'` | Backup should complete successfully. New snapshot ID returned in output. |
| **2.2** | **Scheduled Backup Execution** | Manually trigger task: `Start-ScheduledTask -TaskName "Kopia Daily Backup"` | Task starts. Check `C:\ProgramData\kopia\logs\backup-*.log`. Log should end with "Backup Complete". |
| **2.3** | **MinIO Verification** | Login to MinIO Console -> Browser Bucket `kopia-backups` | Verify folder structure: `kopia-backups/<hostname>/`. Data blobs and index files should be recently modified. |
| **2.4** | **Open File Backup (VSS)** | Open a Word doc in `C:\Data`. Run backup. | Backup should succeed without errors (Shadow Copy used). |

---

## Section 3: Resilience & Self-Healing (Critical)

These scenarios test the specific auto-recovery logic implemented in `run-backup.ps1` and `reconnect.ps1`.

| ID | Test Case | Scenario / Action | Expected Result |
|----|-----------|-------------------|-----------------|
| **3.1** | **Repository Disconnected** | 1. `salt 'win-minion' cmd.run '"C:\Program Files\Kopia\kopia.exe" repository disconnect'`<br>2. Run `run-backup.ps1` | **FAIL-OVER SUCCESS**:<br>1. Log shows "Repository not connected".<br>2. calling `reconnect.ps1`.<br>3. Connection established.<br>4. Backup proceeds and completes. |
| **3.2** | **Corrupt Config File** | 1. Delete `%LOCALAPPDATA%\kopia\repository.config`<br>2. Run `run-backup.ps1` | **AUTO-RECOVERY**:<br>Similar to 3.1. Script detects valid status failure, triggers reconnect, regenerates config, completes backup. |
| **3.3** | **Cache Corruption** | 1. Stop Kopia tasks.<br>2. Corrupt/Delete `C:\ProgramData\kopia\cache`<br>3. Run Backup | Kopia should rebuild cache locally. Backup might take longer but MUST succeed. |
| **3.4** | **Restart Persistence** | Reboot Windows Minion. Wait for startup. | 1. Env Vars persist (System level).<br>2. Scheduled Tasks persist.<br>3. First backup after reboot succeeds. |
| **3.5** | **Repository Re-initialization (Idempotency)** | Run `reconnect.ps1` when ALREADY connected. | Script should output "Already connected" and exit gracefully with Code 0. NOT attempt to reconnect or error out. |

---

## Section 4: Restore Operations

| ID | Test Case | Action / Command | Expected Result |
|----|-----------|------------------|-----------------|
| **4.1** | **List Snapshots** | `salt 'win-minion' cmd.run 'powershell -File ...\restore-snapshot.ps1 -ListSnapshots'` | Should list available snapshots with their IDs and timestamps. |
| **4.2** | **Full Restore** | 1. Create dummy file `C:\Data\restore_test.txt`. Backup it.<br>2. Delete file.<br>3. Run `restore-snapshot.ps1 -SnapshotId <latest_id> -TargetPath C:\RestoreTest` | File `restore_test.txt` appears in `C:\RestoreTest`. Content checksum matches original. |

---

## Section 5: Security & Maintenance

| ID | Test Case | Action / Command | Expected Result |
|----|-----------|------------------|-----------------|
| **5.1** | **Credential Exposure** | run `Get-Content C:\ProgramData\kopia\scripts\connect.ps1` | **PASS**: Output MUST NOT contain literal secrets/passwords. Should show `$env:KOPIA_...`. |
| **5.2** | **Maintenance Task** | Run `maintenance-task.ps1` | Logs showing "Running quick maintenance" / "full maintenance". Blob stats should be displayed. |
| **5.3** | **Log Rotation** | Check `C:\ProgramData\kopia\logs` | Logs older than 30 days (if simulating date change) should be deleted. |

---

# Kopia Test Execution Guide
**Target Minion: `svn-test` (Replace with `win-*` or specific ID as needed)**

This guide provides the exact commands to execute the test plan. Run these from your Salt Master.

---

## 1. Environment & Deployment Tests

### Test 1.1: Verify Environment Variables
Check if sensitive credentials and config are set as system environment variables.
```bash
salt 'svn-test' cmd.run 'Get-ChildItem env:KOPIA*' shell=powershell
```
**Success Criteria:** You should see `KOPIA_S3_ACCESS_KEY`, `KOPIA_REPO_USERNAME`, etc.

### Test 1.2: Check Script Existence
Verify all necessary scripts are deployed.
```bash
salt 'svn-test' cmd.run 'ls C:\ProgramData\kopia\scripts'
```
**Success Criteria:** List includes `connect.ps1`, `reconnect.ps1`, `run-backup.ps1`, `restore-snapshot.ps1`.

### Test 1.3: Verify Scheduled Tasks deployment
Check if the daily backup and weekly maintenance tasks are registered.
```bash
salt 'svn-test' cmd.run 'Get-ScheduledTask -TaskName "Kopia*"' shell=powershell
```
**Success Criteria:** Status should be `Ready`.

### Test 1.4: Verify Repository Status
Confirm Kopia is currently connected.
```bash
salt 'svn-test' cmd.run '"C:\Program Files\Kopia\kopia.exe" repository status'
```
**Success Criteria:** Output starts with "Connected to repository..."

---

## 2. Backup Functionality Tests

### Test 2.1: Run On-Demand Backup
Trigger a manual backup of a test folder to verify the `on-demand-backup.ps1` script.
```bash
# Create dummy data first
salt 'svn-test' cmd.run 'mkdir C:\TestBackup; echo "Critical Data" > C:\TestBackup\important.txt' shell=powershell

# Run backup
salt 'svn-test' cmd.run 'powershell -ExecutionPolicy Bypass -File "C:\ProgramData\kopia\scripts\on-demand-backup.ps1" -Path "C:\TestBackup" -Description "Manual Test 2.1"'
```
**Success Criteria:** Output shows snapshot created successfully with an ID.

### Test 2.2: Trigger Scheduled Task via Salt
Simulate the scheduled task running.
```bash
salt 'svn-test' cmd.run 'Start-ScheduledTask -TaskName "Kopia Daily Backup"' shell=powershell
```
```bash
salt '*' cmd.run 'Get-ScheduledTask -TaskName "Kopia Daily Backup" | Select-Object TaskName,State,LastRunTime,NextRunTime' shell=powershell
```
```bash
salt '*' cmd.run 'Get-ScheduledTaskInfo -TaskName "Kopia Daily Backup" | Select-Object LastRunTime,LastTaskResult,NumberOfMissedRuns' shell=powershell
```
```bash
salt '*' cmd.run 'Get-ChildItem "C:\ProgramData\kopia\logs\backup-*.log" | Sort-Object LastWriteTime | Select-Object -Last 1 | Get-Content | Select-Object -Last 10' shell=powershell
```
**Success Criteria:** Task starts. You can verify logs in the next step.

### Test 2.3: Inspect Backup Logs
Check the latest log created by the backup task.
```bash
salt 'svn-test' cmd.run 'Get-ChildItem "C:\ProgramData\kopia\logs\backup-*.log" | Sort-Object LastWriteTime | Select-Object -Last 1 | Get-Content | Select-Object -Last 10' shell=powershell
```
**Success Criteria:** Log ends with "=== Backup Complete ===".

---

## 3. Resilience & Self-Healing Tests (Auto-Reconnect)

### Test 3.1: Repository Disconnect & Auto-Heal
**Scenario:** Admin accidentally disconnects repo, or config is corrupted. The scheduled backup MUST fix this.

**Step A: Force Disconnect**
```bash
salt 'svn-test' cmd.run '"C:\Program Files\Kopia\kopia.exe" repository disconnect'
```

**Step B: Verify Disconnected State**
```bash
salt 'svn-test' cmd.run '"C:\Program Files\Kopia\kopia.exe" repository status'
```
*Expected: Error / Not connected.*

**Step C: Run Backup Script (Simulate Schedule)**
```bash
salt 'svn-test' cmd.run 'powershell -ExecutionPolicy Bypass -File "C:\ProgramData\kopia\scripts\run-backup.ps1"'
```

**Step D: Verify Self-Healing in Logs**
```bash
salt 'svn-test' cmd.run 'Get-ChildItem "C:\ProgramData\kopia\logs\backup-*.log" | Sort-Object LastWriteTime | Select-Object -Last 1 | Get-Content' shell=powershell
```
**Success Criteria:** Log should contain lines like:
> "Repository not connected. Attempting to reconnect..."
> "Successfully reconnected to repository!"
> "Creating snapshot..."

### Test 3.5: Idempotency (Already Connected)
Verify that `reconnect.ps1` doesn't break anything if already connected.
```bash
salt 'svn-test' cmd.run 'powershell -ExecutionPolicy Bypass -File "C:\ProgramData\kopia\scripts\reconnect.ps1" -NonInteractive'
```
**Success Criteria:** Output says "Already connected to repository".

---

## 4. Restore Tests

### Test 4.1: List Available Snapshots
Verify the restore script can connect and list snapshots (uses `reconnect` logic internally).
```bash
salt 'svn-test' cmd.run 'powershell -ExecutionPolicy Bypass -File "C:\ProgramData\kopia\scripts\restore-snapshot.ps1" -ListSnapshots'
```
**Success Criteria:** Lists snapshots with IDs, timestamps, and paths.

### Test 4.2: Perform Restore
Restore the data we backed up in Test 2.1.
```bash
# 1. Get latest snapshot ID for C:\TestBackup
# (Replace <SNAPSHOT_ID> below with an actual ID from previous step, or use 'latest')

# 2. Delete original file to prove restore works
salt 'svn-test' cmd.run 'del C:\TestBackup\important.txt' shell=powershell

# 3. Restore to original location (or new location)
salt 'svn-test' cmd.run 'powershell -ExecutionPolicy Bypass -File "C:\ProgramData\kopia\scripts\restore-snapshot.ps1" -SnapshotId <SNAPSHOT_ID> -TargetPath "C:\RestoreVerification"'

# 4. Verify restored file
salt 'svn-test' cmd.run 'Get-Content C:\RestoreVerification\important.txt' shell=powershell
```
**Success Criteria:** File content "Critical Data" is displayed.

---

## 5. Security & Maintenance Tests

### Test 5.1: Check for Exposed Secrets
Ensure scripts don't contain hardcoded passwords on the minion.
```bash
salt 'svn-test' cmd.run 'Select-String -Path "C:\ProgramData\kopia\scripts\connect.ps1" -Pattern "password=\""' shell=powershell
```
**Success Criteria:** Should NOT find lines setting password literal values (except variable assignments). Look for usage of `$env:`.

### Test 5.2: Run Maintenance
Manually run the maintenance script.
```bash
salt 'svn-test' cmd.run 'powershell -ExecutionPolicy Bypass -File "C:\ProgramData\kopia\scripts\maintenance-task.ps1"'
```
**Success Criteria:** Output shows blob statistics and successful maintenance run.
