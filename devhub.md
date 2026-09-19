<img src="Images/01-Banner.png" width="600">

We will begin this machine by running an Nmap scan to identify open ports:

<img src="Images/02-Nmap.png" width="600">

Inspecting the web server, reveals the following landing page where we discover an MCP tool running on port 6274:

<img src="Images/03-Landing.png" width="600">

Navigating to port 6274 displays MCPjam version 1.4.2:

<img src="Images/04-MCPjam.png" width="600">

Searching for exploits related to this version of MCPjam uncovers the following proof of concept:

<img src="Images/05-Exploit-Location.png" width="600">

We will now set up a listener on port 4444, prepare and send the exploit:

<img src="Images/06-Listener.png" width="600">

<img src="Images/07-Exploit-Preparation.png" width="600">

And we catch a reverse shell as mcp-dev:

<img src="Images/08-Listener2.png" width="600">

We will now generate SSH keys for the target machine using ssh-keygen before importing our attacker host public key into the authorized keys file:

<img src="Images/09-Stabilisation1.png" width="600">

Now we can create an SSH session as mcp-dev:

<img src="Images/10-Login.png" width="600">

Enumerating the system, we display running processes for the analyst user, which discloses a token for Jupyter notebook on port 8888:

<img src="Images/11-Processes.png" width="600">

We will therefore forward this port and navigate to the address in our web browser:

<img src="Images/12-Port-Forwarding.png" width="600">

We can now use our newly discovered token to log into Jupyter Notebook:

<img src="Images/13-Jupyter1.png" width="600">

Once the page is loaded, we will click on Python notebook:

<img src="Images/14-Jupyter2.png" width="600">

And insert a Python reverse shell in the Jupyter code cell:

<img src="Images/15-Jupyter3.png" width="600">

Finally, we will set up a listener and click the play button to execute our reverse shell and catch a shell as the analyst user where we stabilise the shell:

<img src="Images/15-Stabilization.png" width="600">

Commands to stabilise the shell:

python3 -c ‘import  pty;pty.spawn(“/bin/bash”)’
export TERM=xterm
ctrl-z
stty raw -echo; fg

Next, we will secure an SSH connection for the analyst user and log in:

<img src="Images/16-Stabilization2.png" width="600">

<img src="Images/17-Stabilisation3.png" width="600">

Enumerating the machine with our new user, we discover the file /opt/opsmcp/server.py. Inspecting this file we discover a secret key:

<img src="Images/18-Secret-Key.png" width="600">

Moreover, displaying connections using netstat reveals the local port 5000:

<img src="Images/19-Netstat.png" width="600">

We will test this connection using curl, as shown below:

<img src="Images/20-cURL.png" width="600">

It tells us that auth is required. We will retest our connection with the opsmcp secret key we discovered in the server.py file:

<img src="Images/21-Testing-Key.png" width="600">

The key works!
Looking again at server.py reveals that there are hidden tools not available in the /tools/list function:

<img src="Images/22-Hidden-Tools.png" width="600">

A closer inspection shows that we can specify ssh_keys as the value to the target parameter:

<img src="Images/23-Hidden-Tools2.png" width="600">

I used AI for this, but eventually was able to identify the correct call with the following:

<img src="Images/24-SSH-Key-Extraction.png" width="600">

Success! We have recovered the root user’s SSH key. After formatting the key, we can log into the machine as root. This is shown below:

<img src="Images/25-Root-Access.png" width="600">

<img src="Images/26-Root-Access2.png" width="400">

