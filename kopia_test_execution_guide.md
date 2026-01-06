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
