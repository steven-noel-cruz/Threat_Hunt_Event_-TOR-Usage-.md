**Threat Hunt Report: Bryce Montgomery Investigation**

---

### **Platforms and Languages Leveraged**
- **Platforms**:
  - Microsoft Sentinel (Log Analytics Workspace)
  - Windows-based corporate workstations (corporate and shared environments)
  - Shared guest workstations on campus
  
- **Languages/Tools**:
  - **Kusto Query Language (KQL)** for querying device events, process logs, and file activities.
  - **Steganography tools**: *Steghide.exe* for embedding data into images.
  - **7z.exe** for compressing and packaging files into an archive for potential exfiltration.

---

### **Scenario**
The VP of Risk requested an investigation after suspecting that Bryce Montgomery, a company executive, may have been involved in the unauthorized access and exfiltration of sensitive corporate intellectual property. The investigation needed to focus on Mr. Montgomery’s computer and potential misuse of shared workstations.

Key points:
- **User**: Bryce Montgomery (username: *bmontgomery*).
- **Workstation**: Corporate workstation (*corp-ny-it-0334*), but additional guest workstations were suspected.
- **Concern**: Data exfiltration through steganography and compression of files for unauthorized transmission out of the corporate network.
  
Executives like Bryce have **full administrative privileges** on their machines, and some were exempted from the **Data Loss Prevention (DLP)** policy, which could make tracing Bryce's activities difficult.

---

### **Steps Taken**

1. **Initial Investigation on Bryce Montgomery's Workstation**:
   - A KQL query was executed on the workstation `corp-ny-it-0334` to identify files interacted with by Bryce.
   - Tracked **FileName** and **FilePath** for any critical corporate documents (such as “Q1-2025-ResearchAndDevelopment.pdf”) and identified the file *thumbprint* (hash) `b3302e58be7eb604fda65d1d04a5e18325c66792`.

**Query Used**:
```kql
DeviceFileEvents
| where DeviceName == "corp-ny-it-0334"
| where InitiatingProcessAccountName == "bmontgomery"
| order by TimeGenerated desc 
| project TimeGenerated ,DeviceName, InitiatingProcessAccountName, FileName, SHA1
```
![image](https://github.com/user-attachments/assets/8a2c9918-6c46-4ae7-bca4-a418a5a27d53)


2. **Cross-Reference on Shared Workstations**:
   - Investigated shared workstations that Mr. Montgomery might have accessed under generic or guest profiles.
   - The **DeviceFileEvents** table was queried to compare file interactions and identify other possible workstations used by Mr. Montgomery.
   - DeviceName **"lobby-fl2-ae5fc"** was flagged for matching files found on Bryce's workstation, establishing his possible use of a guest workstation.
   - Most importantly, the filenames of the data have been manipulated to mask their contents. A tactic common for data obfuscation and exfiltration.

 **Query Used**:
```kql
DeviceFileEvents
| where PreviousFileName contains "Q3-2025-AnimalTrials-SiberianTigers" or PreviousFileName contains "Q2-2025-HumanTrials" or PreviousFileName contains "Q1-2025-ResearchAndDevelopment"
| order by TimeGenerated desc 
| project TimeGenerated ,DeviceName, InitiatingProcessAccountName, PreviousFileName, FileName
```
![image](https://github.com/user-attachments/assets/ec998431-f79c-4ce7-b6e2-73fbcce41beb)

3. **Identifying Steganography Use**:
   - After querying **DeviceProcessEvents** for process interactions involving corporate files, it was discovered that **Steghide.exe** was used to embed corporate documents into personal image files on the shared workstation.
   - Images involved: `suzie-and-bob.bmp`, `bryce-and-kid.bmp`, and `bryce-fishing.bmp`.
   - The **ProcessCommandLine** indicated the use of *steghide.exe* with an output folder path to C:\ProgramData\.

 **Query Used**:
```kql
DeviceProcessEvents
| where DeviceName == "lobby-fl2-ae5fc"
| where ProcessCommandLine contains "bryce-homework-fall-2024.pdf" or ProcessCommandLine contains "Amazon-Order-123456789-Invoice.pdf" or ProcessCommandLine contains "temp___2bbf98cf.pdf"
| order by TimeGenerated desc 
| project TimeGenerated ,DeviceName, AccountName, ProcessCommandLine
```
![image](https://github.com/user-attachments/assets/85b67407-0150-4fb7-a012-562d4a7e5f2b)

4. **Investigating Compression and Transfer**:
   - Further KQL queries showed the **7z.exe** process interacting with these same stego images based on the proccess thumbprint, confirming they were being compressed into a zip file (`marketing_misc.zip`).
   - The zip file ended up on the **F:\** drive of the lobby workstation, further pointing to suspicious data packaging for exfiltration.
 
 **Query Used**:
```kql
DeviceFileEvents
| where DeviceName == "lobby-fl2-ae5fc"
| where InitiatingProcessCommandLine contains "suzie-and-bob.bmp" or InitiatingProcessCommandLine contains "bryce-fishing.bmp" or InitiatingProcessCommandLine contains "bryce-and-kid.bmp"
| order by TimeGenerated desc 
| project TimeGenerated ,DeviceName, FileName, InitiatingProcessCommandLine, FolderPath, SHA256
```
![image](https://github.com/user-attachments/assets/78c42ebe-e1c8-48cb-b5ff-80490afbc69d)

 **Query Used**:
```kql
DeviceFileEvents
| where SHA256 contains "07236346de27a608698b9e1ffef07b1987aa7fe8473aac171e66048ff322e2d6"
| order by TimeGenerated desc 
| project TimeGenerated ,DeviceName, FileName, PreviousFileName, InitiatingProcessCommandLine, FolderPath, SHA256
```
![image](https://github.com/user-attachments/assets/08e6a693-60c1-4d63-9f35-bd65dfc8ca88)


5. **Identifying the Culprit**:
   - The final piece of evidence came from a **damning KQL record** that directly tied Bryce Montgomery to these actions.
   - Timestamp **2025-02-05T08:57:32.2582822Z** revealed that Bryce, under their profile and device, collected the data for exfiltration in their personal folder.

   **Query Used**:
```kql
DeviceFileEvents
| where SHA256 contains "07236346de27a608698b9e1ffef07b1987aa7fe8473aac171e66048ff322e2d6"
| order by TimeGenerated desc 
| project TimeGenerated ,DeviceName, InitiatingProcessAccountName, FileName, PreviousFileName, InitiatingProcessCommandLine, FolderPath, SHA256
```

![image](https://github.com/user-attachments/assets/1ad95d2e-34a0-4cd3-ae9d-edd0c726b0e5)

---

### **Summary of Findings**
The investigation revealed that Bryce Montgomery, using both his own corporate workstation and a shared guest workstation, attempted to exfiltrate sensitive company data through the following steps:

1. **Thumbprint of one of the first corporate files that was accessed or interacted with by Bryce Montgomery during the investigation.:** b3302e58be7eb604fda65d1d04a5e18325c66792
2. **DeviceName of any other workstation Mister Montgomery may have used:** lobby-fl2-ae5fc
3. **FileName of the process that interacted with the files Q1-2025-ResearchAndDevelopment.pdf,Q2-2025-HumanTrials.pdf,Q3-2025-AnimalTrials-SiberianTigers.pdf:** steghide.exe
4. **The full path of one of the files that was created as a result of steghide.exe running:** C:\ProgramData\bryce-and-kid.bmp (suzie-and-bob.bmp)(bryce-fishing.bmp)
5. **The SHA256 thumbprint of the OTHER process that interacted with any of the stego images:** 707f415d7d581edd9bce99a0429ad4629d3be0316c329e8b9ebd576f7ab50b71
6. **The complete FolderPath where the zip file ultimately ended up on the lobby computer:** F:\marketing_misc.zip
7. **Damning piece of evidence that Bryce is the one attempting to steal corporate information and Timestamp (UTC) of the damning event:** F:\Bryce Personal\marketing_misc.zip & 2025-02-05T08:57:32.2582822Z
  
---

### **Response Taken**
1. **Immediate User Suspension**:
   - Bryce Montgomery's access to the corporate network and all systems was immediately suspended pending further legal and HR investigation.
  
2. **Incident Escalation**:
   - The Security Operations team escalated the incident to the VP of Risk and Corporate Legal departments, triggering a formal investigation into potential data theft and corporate espionage.
  
3. **Forensic Image and Collection**:
   - Forensic imaging of Bryce's workstation and the shared workstations involved was initiated. All activity logs, steganography tools, and data zipping processes were secured for further analysis.
  
4. **Review of DLP Policies**:
   - The Security team recommended reviewing the **DLP exemption** policy for executives, given that this exemption created a blind spot in monitoring potentially malicious activities.
  
5. **Data Integrity Review**:
   - An internal audit was initiated to determine whether any of the data embedded in the images had been compromised or transmitted outside the company.

6. **Awareness Training**:
   - A company-wide security awareness campaign was started, highlighting the dangers of insider threats, steganography, and the importance of following data protection policies.

This report will be kept as part of the formal record for this case. The Security Operations and Risk Departments will continue monitoring the situation and ensure all legal actions are taken.

--- 

**End of Report**
