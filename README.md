# HTB-devhub

<img src="Images/01-Banner.png" width="600">

**Difficulty:** Medium

**OS:** Linux

**IP:** 10.129.245.216

**Date:** 30/05/2026

We began this machine with a standard Nmap scan which led to the discovery of an Nginx webserver on port 80 and the MCPJam platform on port 6274. Investigation revealed MCPJam version 1.4.2 was being used and that a Proof of Concept existed, giving us code execution. Leveraging this, we obtained a shell as mcp-dev.

Viewing running processes of the analyst user, we discovered disclosure of a Jupyter Notebook instance running on port 8888 and a sensitive login token. Port forwarding 8888 gives us access to Jupyter Notebook locally, where we log in and discover a Python3 Console function. Leveraging this, we obtained a shell as analyst.

Enumeration of the machine as analyst, leads us to discover the file /opt/opsmcp/server.py. This file gives us the information required to craft a malicious POST request, where we access hidden functionality and retrieve root's valuable private SSH key. We then format the key and log in as root.

**Key Findings**


| Finding         | Severity     | Impact                                                         |
| :-------------- | :----------- | :------------------------------------------------------------- |
| CVE-2026-23744 | Critical:&nbsp;9.8 | Remote Code Execution on a remote server running MCPJam 1.4.2. |
| Jupyter token disclosure | High:&nbsp;9.0 | Access gained to Jupyter Notebook leading to code execution and horizontal privilege escalation.
| Hidden software functionality, leading to privilege escalation | Critical:&nbsp;9.5 | Disclosure of root SSH key. |
