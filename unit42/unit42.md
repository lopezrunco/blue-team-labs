# Unit42

## Scenario:

Palo Alto's Unit42 recently conducted research on an UltraVNC campaign, wherein attackers utilized a backdoored version of UltraVNC to maintain access to systems. This lab is inspired by that campaign and guides participants through the initial access stage of the campaign.

## Tools used:

- Event Viewer
- Sysmon
- VirusTotal

## Skills Learnt:

- Event Log Analysis
- Sysmon Log analysis
- UltraVNC infection campaign
- Timeline creation
- Contextual Analysis

## Path:

After downloading the artefacts I inzup them and found a `.evtx` file named `Microsoft-Windows-SysmonOperational.evtx` which I opened in the **EventViewer**. The file has 169 Sysmon events.

To know how many Event logs are there with Event ID 11 (which indicates a file has been created on a host within Sysmon logs), I went to **Filter Current log** in the **Actions** panel, filtered by **ID 11** and the results were **56 logs**.

![56 events with ID 11](./assets/56-events.png)

Whenever a process is created in memory, an event with Event ID 1 is recorded with details such as command line, hashes, process path, parent process path, etc. This information is very useful for an analyst because it allows us to see all programs executed on a system, which means we can spot any malicious processes being executed. What is the malicious process that infected the victim's system?

To determine which malicious process infected the victim's system I filtered by created proccess, with **Event ID 1**, that returned 6 events. Looking through them, a suspicious event catched my eye, as it had a double `.exe` extension and it was executed from the **Downloads** folder.

![C:\Users\CyberJunkie\Downloads\Preventivo24.02.14.exe.exe](./assets/preventivo.png)

Checking the hash in VirusTotal I confirmed that this is a malicious executable:

![Checking in virustotal](./assets/malicious.jpg)

To know which Cloud drive was used to distribute the malware, I filtered by Event ID 22, which corresponds to DNS queries made by the system. That returned only 3 event logs, and the first one inmediatly catched my eye, showing a query to `dropbox`:

![Dropbox](./assets/dropbox.png)

For many of the files it wrote to disk, the initial malicious file used a defense evasion technique called Time Stomping, where the file creation date is changed to make it appear older and blend in with other files. The next question is what was the timestamp changed to for the `PDF` file.
I filtered by **Event ID 2**, which records any file creation time change on any files on the system. That resulted in **16** events. I looked for a `PDF` file and once I found it, I checked the **CreationUtcTime** which was `2024-01-14 08:10:06`.

![Creation UTC time](./assets/creation-utc-time.png)

Next steps is to determine where was the "once.cmd" file created on disk, si I filtered by **Event ID 11**, which logs when a file is created. This showed me 56 events. Then I search by the filename **"once.cmd"**m this showed me 2 results: The first one created by `msiexec` and the second created by `preventivo24.02.14.exe`:

![once.cmd](./assets/once-cmd.png)

As I determined in the beginning that `preventivo24.02.14.exe` was the executable which malicious process infected the victim's system, I concluded that the route is `C:\Users\CyberJunkie\AppData\Roaming\Photo and Fax Vn\Photo and vn 1.1.2\install\F97891C\WindowsVolume\Games\once.cmd`.

To determine the dummy domain the malicious file attempted to reach, I filter by **Event ID 22**, which is the one that captures DNS queries made by processes. After filtering I got 3 results and by checking them I found that `www.example.com` is the dummy domain.

![Example.com](./assets/examplecom.png)

Next, to know which IP address did the malicious process try to reach out to, I filtered by **Event ID 3**, which records outgoing network connections initiated by processes. I only got 1 event, looking throught in the field **DestinationIP** which was `93.184.216.34`:

![Destination IP](./assets/destip.png)

Final question is when the malicious process terminated itself. I filtered by **Event ID 5** to show the logs of the termination of processes, showing 1 result, which showed the date `2024-02-14 03:41:58`:

![Terminated process](./assets/terminated.png)

## Conclusion:

The arracker deployed a backdoored **UltraVNC** executable named `preventivo24.02.14.exe.exe` via **Dropbox**. By analizing the logs, I identified malicious activity, file creation, DNS queries and process termination.