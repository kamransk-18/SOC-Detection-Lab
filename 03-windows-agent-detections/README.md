# 03 — Windows Agent Detections (Windows 11)

Sysmon-based telemetry, a PowerShell execution detection, and a fully automated host-isolation response to brute force.

## What's being tested

Whether Windows endpoint telemetry (Sysmon) reaches the Manager, whether a specific process-execution pattern (PowerShell) triggers a custom rule, and whether the Manager can take a Windows host off the network automatically in response to an attack.

## Agent enrolment

```powershell
# From Dashboard's "Deploy new agent" wizard: select Windows, copy the PowerShell one-liner, run as Administrator on Kamran-PC, then:
NET START WazuhSvc
```

---

## A. Installing Sysmon (foundation for B and C below)

### Commands
```powershell
# Download Sysmon from https://learn.microsoft.com/sysinternals/downloads/sysmon, extract to C:\Sysmon
# Download a config (Wazuh's sample or SwiftOnSecurity's) as C:\Sysmon\sysmonconfig.xml

cd C:\Sysmon
.\Sysmon64.exe -accepteula -i .\sysmonconfig.xml
```
Verify: **Event Viewer → Applications and Services Logs → Microsoft → Windows → Sysmon → Operational**.

Point the agent at the Sysmon channel — edit `C:\Program Files (x86)\ossec-agent\ossec.conf` (`<localfile>`, `eventchannel` format, `Microsoft-Windows-Sysmon/Operational`), then:
```powershell
Restart-Service wazuh-agent
Select-String -Path "C:\Program Files (x86)\ossec-agent\ossec.log" -Pattern "sysmon" -SimpleMatch
```
On the Manager, confirm raw events are arriving:
```bash
sudo tail -f /var/ossec/logs/archives/archives.json | grep Sysmon
```

---

## B. PowerShell / Malicious Command Detection

Add rule 100900 to `local_rules.xml` (Dashboard → Management → Rules), restart the Manager, then trigger on Kamran-PC:
```powershell
powershell.exe
```
Confirm the alert under **Agents → Security events**. Wazuh's built-in `0595-win-sysmon_rules.xml` ruleset already covers many related techniques and is worth reviewing alongside the custom rule.

---

## C. Complete Host Isolation

**Architecture:** Attack detected → Manager triggers active response → Kamran-PC runs the isolation script → network adapter disabled → endpoint isolated.

### Commands
Create `C:\Program Files (x86)\ossec-agent\active-response\bin\host-isolation.cmd`:
```bat
@echo off
for /f "tokens=2 delims=:" %%i in ('netsh interface show interface ^| findstr "Connected"') do set IFACE=%%i
netsh interface set interface "%IFACE%" admin=disable
echo %date% %time% : Host isolated by Wazuh Active Response >> "C:\Program Files (x86)\ossec-agent\active-response\active-responses.log"
```
Register the command in `ossec.conf` (`<command><name>host-isolation</name>...`), then:
```powershell
Restart-Service wazuh-agent
```
On the **Manager**, tie it to the brute-force rules (`<active-response><command>host-isolation</command><rules_id>60122,60204</rules_id>`):
```bash
sudo systemctl restart wazuh-manager
```
**Attack from Kali Machine:**
```bash
hydra -l Administrator -P rockyou.txt ssh://<Kamran-PC IP>
```
**Verify isolation on Kamran-PC:**
```powershell
Get-NetAdapter
```
**Restore:**
```powershell
Enable-NetAdapter -Name "Ethernet" -Confirm:$false
```
**Alternative (softer) response** — block all traffic except the Manager instead of disabling the adapter entirely:
```powershell
$ManagerIP = "<Wazuh Host IP>"
netsh advfirewall firewall add rule name="Block-All-Out" dir=out action=block
netsh advfirewall firewall add rule name="Block-All-In" dir=in action=block
netsh advfirewall firewall add rule name="Allow-Manager-Out" dir=out action=allow remoteip=$ManagerIP
netsh advfirewall firewall add rule name="Allow-Manager-In" dir=in action=allow remoteip=$ManagerIP
```

---

## Screenshot Evidence

| # | Screenshot | What it captures |
|---|---|---|
  | 1 | ![Sysmon Event Viewer](screenshots/03-sysmon-eventviewer.png) | Windows Event Viewer showing Sysmon Operational events being generated on Windows 11. |
| 2 | ![Manager Sysmon Arriving](screenshots/04-manager-sysmon-arriving.png) | Wazuh Manager terminal confirming Sysmon events are arriving in the archived event stream. |
| 3 | ![Local Rule 100900](screenshots/05-local-rule-100900.png) | Wazuh `local_rules.xml` showing the custom PowerShell detection rule `100900`. |
| 4 | ![PowerShell Launch](screenshots/06-powershell-launch.png) | PowerShell being launched on Windows 11 to trigger the custom detection rule. |
| 5 | ![PowerShell Alert Dashboard](screenshots/07-powershell-alert-dashboard.png) | Wazuh Dashboard → Agents → Security events showing the PowerShell detection alert. |
| 6 | ![Isolation Script](screenshots/08-isolation-script.png) | Contents of `host-isolation.cmd`, showing the Windows host-isolation response script. |
| 7 | ![Active Response Manager Config](screenshots/09-active-response-manager-config.png) | Wazuh Manager `ossec.conf` showing `host-isolation` configured as an active response for rules `60122/60204`. |
| 8 | ![Hydra Attack Windows](screenshots/10-hydra-attack-windows.png) | Kali terminal running the SSH brute-force test against Windows 11. |
| 9 | ![NetAdapter Disabled](screenshots/11-netadapter-disabled.png) | PowerShell `Get-NetAdapter` output showing the network adapter in the **Disabled** state after host isolation. |
| 10 | ![Isolation Alert Dashboard](screenshots/12-isolation-alert-dashboard.png) | Wazuh Dashboard alert confirming the detection and execution of the host-isolation active response. |
| 11 | ![NetAdapter Restored](screenshots/13-netadapter-restored.png) | Windows network adapter restored to the **Up** state after manually re-enabling it. |
## Analyst notes

- **Sysmon/PowerShell:** matching purely on `powershell.exe` in the image path is broad — real PowerShell attacks (encoded commands, `-nop -w hidden`, unusual parent process like `winword.exe`) need a more specific rule; this version is a v1 baseline you'd iterate on.
- **Host isolation:** this is the highest-blast-radius response in the whole lab — killing the network adapter also cuts the agent's own connection back to the Manager, so there's no way to *remotely* undo it once triggered. In production you'd almost always prefer the "block all except Manager" alternative shown above, specifically so the isolated host stays reachable for remediation.
- Next step as an analyst: confirm false-positive rate on rule 100900 over a normal workday (admins run PowerShell legitimately, constantly) before ever wiring isolation to anything less specific than a confirmed brute-force rule ID.
