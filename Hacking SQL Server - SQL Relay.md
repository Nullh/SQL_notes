# Hacking SQL Server (Security)
## Andre Melancia
### SQL Relay 2016

#### Man in the Middle attacks
Connection between client and server is interrupted by a 3rd party, altering information or collecting it before passing it on.

TDS (Tabular Data Stream) is 'SQL Server protocol', and works on port 1433.

Changing the default port is a weak option, because NMAP will find the port anyway. It's a judgement call.

Back in SQL 2000, the login/pwd packet was not encrypted (password was weakly hashed, so could be brute forced). 2005 solves this but other packets are unencrypted by default.

##### The hard way:
* Physical network tampering
* Crack the router/switch
* ARP spoofing
* DNS poisoning
* and more!!

##### The easy way:
* Change the hosts file
* Change SQL Server aliases - (can I harden my laptop for this?)

Using Wireshark you can see the query text (including comments) which are sent by default. Scarily, the results are in plain text too.

You can't sniff packets internally, as the Windows kernel prevents it. Worth looking at for our Azure servers?

Check in SQL Config Manager if force encryption is set to yes
