# SidRid
A script to change the SID of a windows machine after it has been deployed already. *WARNING THIS SCRIPT IS DANGEROUS AND A LAST RESORT*

1.1. Purpose

This script is a dangerous but effective way to replace the Windows Security Identifier (SID) of a PC. Be aware this should not be used unless you are sure you want to do this. As this is still an experimental script that alters the core of the windows registry.

The primary function is to replace the machine's core SID and update all local user/group accounts accordingly. It includes a robust, fully automated failsafe mechanism to restore administrator access after the destructive operation is complete.

1.2. WARNING: DESTRUCTIVE TOOL

This script performs unsupported, low-level modifications to the Windows Registry (SAM/SECURITY hives).

Incorrect use or unforeseen system states can lead to an unbootable operating system.

2. CORE METHODOLOGY

The script operates in several distinct phases to ensure a precise and recoverable outcome.

Phase 1: Hive Extraction (reg.exe)

The script first needs a copy of the active security hives. It bypasses the problematic VSS (Volume Shadow Copy Service) and instead uses  (reg.exe).

Action: reg.exe save HKLM\SAM ... and reg.exe save HKLM\SECURITY ...

Reasoning: This method uses a low-level API to save a direct copy of the hives, even while they are locked by the operating system. It is the most reliable and universal method, independent of system services that may be misconfigured or corrupted.



Phase 1.5: Direct Binary Injection

This is the core of the destructive operation.

SID Identification: The script identifies the current machine SID by querying the SID of the "Administrator" account.

New SID Generation: A new, cryptographically random SID is generated.

Binary Search and Replace: The script reads the saved hive files into memory as byte arrays. It then performs a precise binary search for the old SID and replaces it with the new SID's binary representation.

Reasoning: SIDs are stored as binary data structures, not simple text strings. A binary replacement is the only way to perform the operation accurately without causing corruption or matching false positives.

Phase 2: Scheduling the Hive Swap

The script uses the PendingFileRenameOperations registry key.

Action: It writes entries that instruct Windows to delete the original SAM and SECURITY files and replace them with the newly modified versions.

Reasoning: This operation occurs during the earliest stages of the next boot cycle, before the hives are locked by the system, making it the standard mechanism for replacing in-use system files.

Phase 2.5: Automated Administrator Recovery (Scheduled Task)

This is the critical failsafe to ensure you are not locked out of the machine.

Script Creation: A simple command script (RecoverAdmin.cmd) is created in a temporary directory. Its only purpose is to enable the built-in Administrator account and add it to the Administrators group.

Task Creation: A one-time Scheduled Task is created using schtasks.exe.

Trigger: onstart (Runs at system boot).

User Account: NT AUTHORITY\SYSTEM (The highest possible privilege).

Action: Runs the RecoverAdmin.cmd script.

Self-Destruction: The final command in the recovery script is to delete the scheduled task itself, so it only ever runs once.

Reasoning: This method is the most robust automated approach. It is more resilient than other methods (RunOnce, SetupExecute) because it relies on the core Task Scheduler service, which is more likely to function correctly in a post-SID-change environment.

Phase 3: Filesystem ACL Scan

Action: The script recursively scans all files and folders on all local drives. For each item, it reads its Access Control List (ACL). If any permission entry references the old machine SID, it is updated to the new SID.

Reasoning: This preserves file and folder permissions. Without this step, users would lose access to their own profile data and other secured resources after the SID change.

Phase 4: Reboot

The final step is to reboot the machine, which triggers the hive swap and the admin recovery task.

3. USAGE INSTRUCTIONS

3.1. Prerequisites

Turn on Allow Local Powershell scripts to run unsigned

You must open PowerShell as an Administrator (Right-click -> Run as Administrator).

3.2. Execution

Place the SidChanger_1114.ps1 script file on the PC/

In the administrative PowerShell window, navigate to the script's location (e.g., cd C:\Users\YourUser\Desktop).

Execute the script by typing its name: .\SidChanger_1114.ps1

You will be prompted with a final warning. You must type I understand this will destroy the OS exactly and press Enter to proceed.

The script will execute its phases. The ACL scan (Phase 3) can take a significant amount of time depending on disk size and speed. Be patient.

At the end, you will be prompted to press Enter to reboot.

3.3. Post-Reboot Procedure

The system will reboot. The hive swap and admin recovery task will run automatically in the background.

When the Windows login screen appears, you should now see the "Administrator" user account.

Click on the Administrator account and sign in. There is no password by default.

Once logged in, you have regained administrative control. You can now use "Computer Management" (compmgmt.msc) to add your original user account back to the "Administrators" group.

4. TROUBLESHOOTING

reg.exe or other commands fail: Ensure you are running PowerShell as an Administrator.

System does not boot after reboot / Blue Screen (BSOD): The SID change has failed and caused OS corruption. There is no supported fix. 

Administrator account does not appear after reboot: The scheduled task failed to run due to severe system instability. This script represents the most reliable automated method; if it fails, manual recovery is the only option, which is outside the scope of this automated tool.
