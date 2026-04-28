# **UpdateBlockerProTray — A Windows Update blocker that actually works (without breaking MedicSvc)**

Windows Update cannot be disabled by services, registry hacks, or “blocker” tools because MedicSvc and SIHClient automatically repair them.  
**UpdateBlockerProTray blocks only the execution layer (wuauserv / UsoSvc) without touching MedicSvc**, so it works even during forced updates like Windows 11 25H2.

---

## 🚀 **Features**

- Blocks Windows Update *execution layer only*  
  - `wuauserv`  
  - `UsoSvc`
- **Does NOT modify or break**  
  - MedicSvc  
  - SIHClient  
  - ACLs  
  - Registry  
  - Scheduled tasks
- Works during **forced update phases** (Windows 11 25H2, Update Stack Package)
- 0% CPU usage / lightweight tray application
- No corruption, no self‑repair loops, no Windows panic

---

## 📸 **Screenshots**

### **1. Tray Menu**

Lightweight tray application with simple controls.  
Auto‑startup, monitoring, and logging can be toggled instantly.

<img width="514" height="256" alt="UpdateBlockerProTray2" src="https://github.com/user-attachments/assets/ed550250-3702-49fb-b144-9a776f2cf7f3" />


---

### **2. Live Log Viewer**

Shows every attempt by Windows Update to start wuauserv/UsoSvc,  
and how the tool stops them instantly — even during forced update phases.

<img width="645" height="370" alt="UpdateBlockerProTray1" src="https://github.com/user-attachments/assets/2b79eb38-f875-413d-9427-32703dd77048" />


---

## 🧠 **Why other methods fail**

### **1. Disabling services (wuauserv, UsoSvc, MedicSvc)**  
This is what 99% of “Windows Update Blocker” tools do.

- Disable services  
- Remove ACLs  
- Kill MedicSvc  
- Modify registry keys  
- Delete scheduled tasks  

**Why it fails:**  
Windows treats this as *corruption*.

- MedicSvc repairs the services  
- SIHClient repairs them daily  
- Update Orchestrator forces them to start  
- Update Stack Package becomes unstable  
- CPU and disk spike  
- Windows Update becomes *more aggressive*  

---

### **2. Network blocking (DNS / HOSTS / Metered)**

This only delays updates.

- Forced updates ignore metered connections  
- Update services still run  
- MedicSvc still repairs  

---

### **3. Execution‑layer blocking (the method this tool uses)**

- Allows Windows Update to start normally  
- Immediately stops only:
  - `wuauserv`
  - `UsoSvc`
- Does **not** touch MedicSvc  
- Windows believes everything is healthy  
- No self‑repair loops  
- No CPU spikes  
- Works during forced updates  

**This is the only method that does not fight Windows.**

---

## **Prototype 1: PowerShell WMI event watcher**  
Missed fast restarts, high overhead, too slow for forced updates.

```PowerShell
# Update Blocker
$currentPid = $PID
Get-Process powershell -ErrorAction SilentlyContinue | ForEach-Object {
    $proc = $_
    if ($proc.MainWindowTitle -eq "Update Blocker" -and $proc.Id -ne $currentPid) {
        Stop-Process -Id $proc.Id -Force
    }
}

$host.UI.RawUI.WindowTitle = "Update Blocker"

$services = @("wuauserv", "UsoSvc")
foreach ($s in $services) {
    Set-Service -Name $s -StartupType Disabled
    Stop-Service -Name $s -Force -ErrorAction SilentlyContinue
}
```

---

## **Prototype 2: PowerShell polling loop**  
Surprisingly effective, but too heavy and unstable for long‑term use.

```powershell
# Update Blocker (Stable Edition)
$host.UI.RawUI.WindowTitle = "Update Blocker - Monitoring..."
$services = @("wuauserv", "UsoSvc")
Stop-Service -Name $services -Force -ErrorAction SilentlyContinue
```

---

## 🔍 **How it works**

Windows Update consists of 5 conceptual layers:

1. Execution (wuauserv / UsoSvc)  
2. Repair (MedicSvc / SIHClient)  
3. Scheduler (Update Orchestrator)  
4. Network (Delivery Optimization)  
5. Policy (GPO / Registry)

**Only the execution layer should be blocked.  
The repair layer must never be touched.**

---

## 🧩 **Core Logic (VB.NET)**

```vbnet
' Core Logic: Monitors and stops target services
Private Sub MonitorServices()
    Static lastDate As String = DateTime.Now.ToString("yyyy-MM-dd")
    Dim currentDate = DateTime.Now.ToString("yyyy-MM-dd")

    If currentDate <> lastDate Then
        logs.Add($"=== {currentDate} ===")
        lastDate = currentDate
    End If

    For Each serviceName As String In targets
        Try
            Dim sc As New ServiceController(serviceName)
            If sc.Status <> ServiceControllerStatus.Stopped Then
                logs.Add($"[Block] [{DateTime.Now:HH:mm:ss}] (Running → Stopped) {serviceName}")
                sc.Stop()

                Dim timeout As DateTime = DateTime.Now.AddSeconds(30)
                Dim retryCount As Integer = 0

                While sc.Status <> ServiceControllerStatus.Stopped AndAlso DateTime.Now < timeout
                    Threading.Thread.Sleep(1000)
                    sc.Refresh()
                    retryCount += 1
                    sc.Stop()
                End While

                If retryCount > 0 Then
                    logs(logs.Count - 1) &= $" (retry: {retryCount})"
                End If
            End If
        Catch
        End Try
    Next
End Sub
```

---

## ⚡ **Quick Start**


1. Extract all files into a single folder.
2. Run UpdateBlockerProTray.exe as Administrator.
3. A New icon will appear in the system tray.
4. The tool silently blocks Windows Update (wuauserv/UsoSvc) with 0% CPU.

Requirements:
- Administrator privileges are required.
- Do not delete Microsoft.Win32.TaskScheduler.dll.

Notes:
- This tool does NOT disable MedicSvc or modify the system.
- It only stops the execution layer, so Windows remains stable.

---

## 📄 **License**

MIT License

---

## 👤 **Author**

 Edgelord+Rhyme

---
