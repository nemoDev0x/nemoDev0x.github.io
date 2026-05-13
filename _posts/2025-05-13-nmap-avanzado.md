---
layout: post
title: "Avanced NMAP"
date: 2026-01-02
categories: [ciberseguridad]
tags: [nmap, pentesting, recon, enumeración, NSE, evasión, firewall, puertos, hacking]
description: "Guía avanzada de Nmap: técnicas de evasión, scripts NSE, fingerprinting, automatización y flujos reales de pentesting."
---

## Introducción

Nmap (Network Mapper) no es solo un escaner de puertos. En manos de un pentester experimentado es una navaja suiza capaz de enumerar servicios, detectar vulnerabilidades, evadir firewalls, hacer fingerprinting de sistemas operativos y automatizar miles de comprobaciones mediante su motor de scripting NSE.

Esta guía cubre desde los escaneos de reconocimiento inicial hasta técnicas avanzadas de evasión y scripting. Cada sección incluye comandos reales usados en entornos de pentesting y CTFs.

> Todo el contenido es solo para uso en entornos autorizados. El escaneo no autorizado es ilegal.

---

## 1. Fundamentos que todo el mundo se salta

Antes de entrar en técnicas avanzadas, hay conceptos básicos que la mayoría ignora y que marcan la diferencia.

### Estados de puertos

Nmap distingue seis estados posibles para un puerto:

| Estado | Significado |
|--------|-------------|
| `open` | El puerto acepta conexiones activamente |
| `closed` | El puerto responde pero no hay servicio escuchando |
| `filtered` | Un firewall bloquea o descarta los paquetes |
| `unfiltered` | El puerto es accesible pero no se puede determinar si está abierto o cerrado |
| `open\|filtered` | No se puede determinar si está abierto o filtrado |
| `closed\|filtered` | Solo aparece en el escaneo IP ID idle |

Entender estos estados es clave para interpretar resultados y elegir la técnica de escaneo correcta.

### Privilegios y tipos de escaneo

Muchos escaneos requieren privilegios de root porque necesitan construir paquetes raw:

```bash
# Sin root: TCP Connect scan (-sT) — completa el three-way handshake
nmap -sT <IP>

# Con root: SYN scan (-sS) — más rápido y sigiloso, no completa la conexión
sudo nmap -sS <IP>
```

El SYN scan es el predeterminado cuando se ejecuta como root. El Connect scan cuando no hay privilegios.

---

## 2. Descubrimiento de hosts

Antes de escanear puertos hay que saber qué hosts están vivos. Nmap ofrece varios métodos.

### Ping sweep básico

```bash
# Descubrir hosts activos en una subred (no escanea puertos)
nmap -sn 192.168.1.0/24

# Sin resolución DNS (más rápido)
nmap -sn -n 192.168.1.0/24

# Guardar resultado
nmap -sn 192.168.1.0/24 -oG hosts_up.txt
```

### Técnicas de descubrimiento avanzadas

```bash
# ARP ping (solo redes locales, muy fiable)
sudo nmap -PR -sn 192.168.1.0/24

# ICMP echo + timestamp + netmask
sudo nmap -PE -PP -PM -sn 192.168.1.0/24

# TCP SYN al puerto 443 para descubrir hosts que bloquean ICMP
sudo nmap -PS443 -sn 192.168.1.0/24

# TCP ACK al puerto 80 (útil cuando SYN está bloqueado)
sudo nmap -PA80 -sn 192.168.1.0/24

# UDP al puerto 53 (DNS)
sudo nmap -PU53 -sn 192.168.1.0/24

# Combinado — máxima cobertura
sudo nmap -PE -PS22,80,443 -PA80,443 -PU53 -sn 192.168.1.0/24
```

### Descubrimiento sin ping

Cuando ICMP está completamente bloqueado:

```bash
# Tratar todos los hosts como activos (no hace ping previo)
nmap -Pn 192.168.1.0/24

# Útil en redes donde el firewall descarta todos los pings
nmap -Pn -p 22,80,443 192.168.1.100
```

---

## 3. Técnicas de escaneo de puertos

### TCP SYN scan (stealth scan)

El más utilizado en pentesting. No completa el handshake, es más rápido y genera menos logs.

```bash
sudo nmap -sS -p- --open 192.168.1.100
```

### TCP Connect scan

Completa el three-way handshake. Más lento pero no requiere root. Genera más logs en el objetivo.

```bash
nmap -sT -p 1-1000 192.168.1.100
```

### UDP scan

Lento pero fundamental. Muchos servicios críticos corren sobre UDP: DNS (53), SNMP (161), TFTP (69), NTP (123).

```bash
sudo nmap -sU -p 53,67,68,69,111,123,161,162,500,514,520 192.168.1.100

# Combinar TCP y UDP en un solo escaneo
sudo nmap -sS -sU -p T:1-1000,U:53,161,500 192.168.1.100
```

### TCP ACK scan

No detecta puertos abiertos. Sirve para mapear reglas de firewall — determina si los puertos son `filtered` o `unfiltered`.

```bash
sudo nmap -sA -p 80,443,8080 192.168.1.100
```

### FIN, NULL y Xmas scan

Técnicas antiguas para evadir firewalls stateless. Envían paquetes con flags inusuales. Los puertos abiertos no responden; los cerrados envían RST.

```bash
# FIN scan — solo el flag FIN activado
sudo nmap -sF -p 1-1000 192.168.1.100

# NULL scan — ningún flag activado
sudo nmap -sN -p 1-1000 192.168.1.100

# Xmas scan — FIN + PSH + URG activados
sudo nmap -sX -p 1-1000 192.168.1.100
```

> Nota: No funcionan contra Windows ni dispositivos que no siguen RFC 793 estrictamente.

### Idle / IP ID scan

Técnica muy sigilosa: utiliza un host zombie para hacer el escaneo. La IP del atacante nunca aparece en los logs del objetivo.

```bash
# Primero verificar que el zombie tiene IP ID incremental
sudo nmap -O --script ipidseq 192.168.1.50

# Escanear objetivo usando el zombie
sudo nmap -sI 192.168.1.50 192.168.1.100 -p 80,443,22
```

### SCTP scan

Para redes de telecomunicaciones (SS7, VoIP).

```bash
sudo nmap -sY 192.168.1.100     # SCTP INIT scan
sudo nmap -sZ 192.168.1.100     # SCTP COOKIE-ECHO scan
```

---

## 4. Detección de servicios y versiones

### Detección básica

```bash
nmap -sV 192.168.1.100
```

### Control de intensidad

El nivel de intensidad va de 0 (ligero) a 9 (máximo). El valor por defecto es 7.

```bash
# Ligero — menos peticiones, más rápido
nmap -sV --version-intensity 2 192.168.1.100

# Máximo — prueba todos los probes disponibles
nmap -sV --version-intensity 9 192.168.1.100

# Equivalente a intensity 9
nmap -sV --version-all 192.168.1.100

# Solo las sondas más ligeras (para no romper servicios frágiles)
nmap -sV --version-light 192.168.1.100
```

### Combinar con scripts

```bash
# Detección de versiones + scripts por defecto
nmap -sV -sC 192.168.1.100

# Equivalente: -A activa versiones, scripts, OS y traceroute
nmap -A 192.168.1.100
```

---

## 5. Detección de sistema operativo

Nmap usa 16 tests TCP/IP diferentes para determinar el OS del objetivo.

```bash
sudo nmap -O 192.168.1.100

# Adivinar OS aunque no haya suficiente confianza
sudo nmap -O --osscan-guess 192.168.1.100

# Limitar intentos (evitar escaneo muy lento)
sudo nmap -O --max-os-tries 1 192.168.1.100
```

### Interpretar el output de OS

```
OS details: Linux 4.15 - 5.6
```

Un rango amplio significa baja confianza. Para mejorar la detección necesitas al menos un puerto abierto y uno cerrado en el objetivo.

---

## 6. NSE — Nmap Scripting Engine

El NSE es el componente más potente de Nmap. Hay más de 600 scripts organizados por categorías.

### Categorías de scripts

| Categoría | Descripción |
|-----------|-------------|
| `auth` | Bypass y brute force de autenticación |
| `broadcast` | Descubrimiento en redes locales sin objetivo específico |
| `brute` | Fuerza bruta de credenciales |
| `default` | Scripts seguros ejecutados con `-sC` |
| `discovery` | Enumeración de información |
| `dos` | Pruebas de denegación de servicio |
| `exploit` | Explotación directa de vulnerabilidades |
| `external` | Consultas a servicios externos (Shodan, etc.) |
| `fuzzer` | Envío de datos inesperados para detectar fallos |
| `intrusive` | Scripts agresivos que pueden causar daños |
| `malware` | Detección de backdoors y malware |
| `safe` | Scripts que no dañan el objetivo |
| `version` | Detección extendida de versiones |
| `vuln` | Detección de vulnerabilidades conocidas |

### Uso de scripts

```bash
# Scripts por defecto (categoría default)
nmap -sC 192.168.1.100

# Script específico
nmap --script http-title 192.168.1.100

# Múltiples scripts
nmap --script http-title,http-headers 192.168.1.100

# Por categoría
nmap --script vuln 192.168.1.100
nmap --script safe,discovery 192.168.1.100

# Por patrón (wildcard)
nmap --script "http-*" 192.168.1.100
nmap --script "smb-enum-*" 192.168.1.100

# Excluir scripts específicos de una categoría
nmap --script "default and not http-brute" 192.168.1.100

# Pasar argumentos a los scripts
nmap --script http-brute --script-args userdb=/tmp/users.txt,passdb=/tmp/pass.txt 192.168.1.100
```

---

## 7. Enumeración por servicio con NSE

### HTTP / HTTPS

```bash
# Información general del servidor web
nmap -p 80,443,8080,8443 --script http-title,http-server-header,http-methods,http-auth-finder 192.168.1.100

# Enumerar directorios
nmap -p 80 --script http-enum 192.168.1.100

# Detectar WAF
nmap -p 80,443 --script http-waf-detect,http-waf-fingerprint 192.168.1.100

# Detectar CMS
nmap -p 80 --script http-wordpress-enum,http-drupal-enum 192.168.1.100

# Buscar vulnerabilidades HTTP conocidas
nmap -p 80,443 --script "http-vuln-*" 192.168.1.100

# Shellshock
nmap -p 80 --script http-shellshock --script-args uri=/cgi-bin/test.cgi 192.168.1.100

# SQL Injection básico
nmap -p 80 --script http-sql-injection 192.168.1.100

# XSS
nmap -p 80 --script http-stored-xss,http-dombased-xss 192.168.1.100
```

### SMB (Windows / Samba)

```bash
# Enumeración completa de SMB
nmap -p 445 --script smb-enum-shares,smb-enum-users,smb-enum-groups,smb-enum-sessions,smb-enum-domains,smb-os-discovery 192.168.1.100

# Detectar vulnerabilidades críticas de SMB
nmap -p 445 --script smb-vuln-ms17-010 192.168.1.100      # EternalBlue
nmap -p 445 --script smb-vuln-ms08-067 192.168.1.100      # MS08-067
nmap -p 445 --script smb-vuln-cve-2017-7494 192.168.1.100 # SambaCry
nmap -p 445 --script "smb-vuln-*" 192.168.1.100           # Todas las vulns SMB

# Información de seguridad SMB (firma, NTLMv2)
nmap -p 445 --script smb-security-mode 192.168.1.100

# Fuerza bruta SMB
nmap -p 445 --script smb-brute 192.168.1.100
```

### SSH

```bash
# Información del servidor SSH
nmap -p 22 --script ssh-hostkey,ssh2-enum-algos 192.168.1.100

# Algoritmos débiles
nmap -p 22 --script ssh2-enum-algos 192.168.1.100

# Fuerza bruta SSH
nmap -p 22 --script ssh-brute --script-args userdb=users.txt,passdb=pass.txt 192.168.1.100

# Autenticación por defecto
nmap -p 22 --script ssh-auth-methods --script-args="ssh.user=root" 192.168.1.100
```

### FTP

```bash
# Comprobar acceso anónimo
nmap -p 21 --script ftp-anon 192.168.1.100

# Bounce attack
nmap -p 21 --script ftp-bounce 192.168.1.100

# Fuerza bruta
nmap -p 21 --script ftp-brute 192.168.1.100

# Información del servidor
nmap -p 21 --script ftp-syst 192.168.1.100

# Vulnerabilidades vsFTPd backdoor (VSFTPD 2.3.4)
nmap -p 21 --script ftp-vsftpd-backdoor 192.168.1.100
```

### SNMP

```bash
# Enumerar con community strings por defecto
nmap -p 161 -sU --script snmp-info,snmp-interfaces,snmp-sysdescr 192.168.1.100

# Brute force de community strings
nmap -p 161 -sU --script snmp-brute 192.168.1.100

# Enumerar procesos corriendo
nmap -p 161 -sU --script snmp-processes 192.168.1.100

# Usuarios del sistema
nmap -p 161 -sU --script snmp-win32-users 192.168.1.100
```

### DNS

```bash
# Enumeración DNS completa
nmap -p 53 --script dns-zone-transfer,dns-recursion,dns-cache-snoop 192.168.1.100

# Zone transfer (AXFR)
nmap -p 53 --script dns-zone-transfer --script-args dns-zone-transfer.domain=target.com 192.168.1.100

# Subdominios por fuerza bruta
nmap -p 53 --script dns-brute --script-args dns-brute.domain=target.com 192.168.1.100
```

### MySQL / MSSQL / PostgreSQL

```bash
# MySQL
nmap -p 3306 --script mysql-info,mysql-databases,mysql-users,mysql-empty-password 192.168.1.100

# MSSQL
nmap -p 1433 --script ms-sql-info,ms-sql-config,ms-sql-dump-hashes 192.168.1.100
nmap -p 1433 --script ms-sql-xp-cmdshell --script-args "mssql.username=sa,mssql.password=,ms-sql-xp-cmdshell.cmd=whoami" 192.168.1.100

# PostgreSQL
nmap -p 5432 --script pgsql-brute 192.168.1.100
```

### RDP

```bash
# Información y vulnerabilidades RDP
nmap -p 3389 --script rdp-enum-encryption,rdp-vuln-ms12-020 192.168.1.100

# BlueKeep (CVE-2019-0708)
nmap -p 3389 --script rdp-vuln-ms12-020 192.168.1.100
```

### LDAP / Active Directory

```bash
# Enumeración LDAP
nmap -p 389,636 --script ldap-search,ldap-rootdse 192.168.1.100

# Brute force
nmap -p 389 --script ldap-brute --script-args ldap.base='"cn=users,dc=example,dc=com"' 192.168.1.100
```

---

## 8. Técnicas de evasión de firewalls e IDS

### Fragmentación de paquetes

Divide los paquetes IP en fragmentos pequeños para confundir sistemas de inspección.

```bash
# Fragmentos de 8 bytes
sudo nmap -f 192.168.1.100

# Fragmentos de 16 bytes
sudo nmap -ff 192.168.1.100

# Tamaño de fragmento personalizado (debe ser múltiplo de 8)
sudo nmap --mtu 16 192.168.1.100
```

### Decoys — Señuelos

Hace que el escaneo parezca provenir de múltiples IPs. La IP real está mezclada entre los señuelos.

```bash
# Señuelos con IPs inventadas (ME = tu IP real)
sudo nmap -D 10.0.0.1,10.0.0.2,ME,10.0.0.3 192.168.1.100

# Señuelos aleatorios (RND)
sudo nmap -D RND:10 192.168.1.100

# IPs reales de la red para más credibilidad
sudo nmap -D 192.168.1.5,192.168.1.6,ME 192.168.1.100
```

### IP spoofing

```bash
# Spoofear IP de origen
sudo nmap -S 192.168.1.200 -e eth0 192.168.1.100

# Spoofear MAC (solo en redes locales)
sudo nmap --spoof-mac 00:11:22:33:44:55 192.168.1.100
sudo nmap --spoof-mac Apple 192.168.1.100     # MAC aleatoria de Apple
sudo nmap --spoof-mac 0 192.168.1.100         # MAC completamente aleatoria
```

### Manipulación de puertos de origen

Algunos firewalls permiten tráfico desde puertos de origen confiables (DNS:53, HTTP:80).

```bash
sudo nmap --source-port 53 192.168.1.100
sudo nmap -g 80 192.168.1.100
```

### Timing — Control de velocidad

El timing afecta directamente la posibilidad de ser detectado por un IDS.

| Template | Nombre | Descripción |
|----------|--------|-------------|
| `-T0` | Paranoid | 5 minutos entre sondas — para IDS muy sensibles |
| `-T1` | Sneaky | 15 segundos entre sondas |
| `-T2` | Polite | 0.4 segundos entre sondas |
| `-T3` | Normal | Por defecto |
| `-T4` | Aggressive | Asume red rápida y fiable |
| `-T5` | Insane | Muy rápido, puede perder puertos |

```bash
# Escaneo sigiloso — muy lento pero difícil de detectar
sudo nmap -T1 -sS 192.168.1.100

# Escaneo rápido para entornos controlados
sudo nmap -T4 -sS 192.168.1.100
```

### Control de timing granular

```bash
# Máximo 1 sonda cada 2 segundos
sudo nmap --scan-delay 2s 192.168.1.100

# Entre 500ms y 1500ms aleatorio (evita detección por patrón)
sudo nmap --scan-delay 500ms --max-scan-delay 1500ms 192.168.1.100

# Retransmisiones máximas
sudo nmap --max-retries 1 192.168.1.100

# Timeout de host
sudo nmap --host-timeout 30m 192.168.1.100
```

### Payloads de datos aleatorios

```bash
# Añadir bytes aleatorios al final de los paquetes
sudo nmap --data-length 25 192.168.1.100
```

### Orden aleatorio de hosts

```bash
# No escanear hosts en orden secuencial
nmap --randomize-hosts -iL targets.txt
```

---

## 9. Rendimiento y velocidad

### Rate limiting

```bash
# Mínimo 1000 paquetes por segundo
sudo nmap --min-rate 1000 192.168.1.100

# Máximo 5000 paquetes por segundo
sudo nmap --max-rate 5000 192.168.1.100

# Sin límite (tan rápido como pueda)
sudo nmap --min-rate 10000 --max-parallelism 100 192.168.1.100
```

### Paralelismo

```bash
# Mínimo 100 sondas en paralelo
sudo nmap --min-parallelism 100 192.168.1.100

# Máximo 256 sondas en paralelo
sudo nmap --max-parallelism 256 192.168.1.100
```

### Escaneo optimizado para redes grandes

```bash
# Primero descubrir todos los puertos abiertos muy rápido
sudo nmap -p- --open -T4 --min-rate 5000 -n -Pn 192.168.1.0/24 -oG fastscan.txt

# Extraer IPs con puertos abiertos del resultado
grep "open" fastscan.txt | awk '{print $2}' > live_hosts.txt

# Segundo: escaneo detallado solo en hosts activos
sudo nmap -sCV -iL live_hosts.txt -oN detailed.txt
```

---

## 10. Formatos de output

Nmap soporta varios formatos de salida. Siempre guarda tus resultados.

```bash
# Normal — legible por humanos
nmap -oN scan.txt 192.168.1.100

# XML — para importar en Metasploit, Faraday, etc.
nmap -oX scan.xml 192.168.1.100

# Grepeable — para procesar con grep/awk
nmap -oG scan.gnmap 192.168.1.100

# Todos los formatos a la vez (recomendado)
nmap -oA scan_results 192.168.1.100
# Genera: scan_results.nmap, scan_results.xml, scan_results.gnmap

# Script kiddie output (s|<rIpt kIddi3 para retos)
nmap -oS scan_leet.txt 192.168.1.100
```

### Procesar resultados con grep/awk

```bash
# Extraer solo puertos abiertos del formato grepeable
grep "open" scan.gnmap

# Extraer IPs de hosts con puerto 22 abierto
grep "22/open" scan.gnmap | awk '{print $2}'

# Listar todos los puertos únicos encontrados
grep -oP '\d+/open' scan.gnmap | sort -u
```

---

## 11. Flujos reales de pentesting

### Flujo 1 — Red corporativa interna

```bash
# Fase 1: Descubrimiento de hosts (rápido, sin ruido)
sudo nmap -sn -PR 10.10.10.0/24 -oG phase1_hosts.txt

# Fase 2: Puertos más comunes en hosts activos
grep "Up" phase1_hosts.txt | awk '{print $2}' > alive.txt
sudo nmap -sS --top-ports 1000 -iL alive.txt -oG phase2_ports.txt

# Fase 3: Escaneo detallado en servicios descubiertos
sudo nmap -sCV -p 22,80,443,445,3389,8080 -iL alive.txt -oA phase3_detail

# Fase 4: Scripts de vulnerabilidades
sudo nmap --script "vuln" -p 80,443,445 -iL alive.txt -oA phase4_vulns
```

### Flujo 2 — Caja de CTF / HackTheBox

```bash
TARGET=10.10.11.100

# Escaneo rápido de todos los puertos
sudo nmap -p- --open -T4 --min-rate 5000 -n -Pn $TARGET -oG allports.txt

# Extraer puertos abiertos
PORTS=$(grep "open" allports.txt | grep -oP '\d+/open' | cut -d'/' -f1 | tr '\n' ',')
echo "Puertos: $PORTS"

# Escaneo detallado en los puertos encontrados
sudo nmap -sCV -p $PORTS $TARGET -oA targeted

# Búsqueda de vulnerabilidades
sudo nmap --script vuln -p $PORTS $TARGET -oA vulns
```

### Flujo 3 — Reconocimiento web

```bash
TARGET=192.168.1.100

# Enumerar servicios web en múltiples puertos
sudo nmap -p 80,443,8080,8443,8888,3000,4000,5000 -sV --script \
  http-title,http-server-header,http-methods,http-auth-finder,\
  http-enum,http-robots.txt,http-waf-detect \
  $TARGET -oA web_enum

# Buscar vulnerabilidades web
sudo nmap -p 80,443 --script "http-vuln-*" $TARGET -oA web_vulns
```

### Flujo 4 — Enumeración de Active Directory

```bash
DC=192.168.1.10

# Puertos típicos de AD
sudo nmap -sCV -p 53,88,135,139,389,445,464,593,636,3268,3269,5985,9389 $DC -oA ad_enum

# Scripts específicos de AD
sudo nmap -p 389,445 --script \
  ldap-rootdse,ldap-search,\
  smb-enum-users,smb-enum-shares,\
  smb-os-discovery,smb-security-mode \
  $DC -oA ad_scripts

# Kerberos
sudo nmap -p 88 --script krb5-enum-users \
  --script-args krb5-enum-users.realm='DOMAIN.LOCAL',\
  userdb=/usr/share/wordlists/seclists/Usernames/Names/names.txt \
  $DC
```

---

## 12. Scripts NSE personalizados

Puedes escribir tus propios scripts NSE en Lua.

```lua
-- /usr/share/nmap/scripts/mi-script.nse
description = [[
Detecta si el servidor devuelve una cabecera X-Powered-By
]]

categories = {"discovery", "safe"}

local http = require "http"
local shortport = require "shortport"

portrule = shortport.http

action = function(host, port)
  local response = http.get(host, port, "/")
  if response and response.header then
    local powered = response.header["x-powered-by"]
    if powered then
      return "X-Powered-By: " .. powered
    end
  end
  return "Cabecera no encontrada"
end
```

```bash
# Ejecutar script personalizado
nmap --script mi-script.nse -p 80 192.168.1.100

# Debug de script
nmap --script mi-script.nse -p 80 --script-trace 192.168.1.100
```

---

## 13. Integración con otras herramientas

### Importar resultados en Metasploit

```bash
# Generar XML con Nmap
nmap -sV -oX scan.xml 192.168.1.0/24

# Importar en Metasploit
msfconsole
> db_import scan.xml
> hosts
> services
> vulns
```

### Usar Metasploit para escanear (db_nmap)

```bash
# Desde msfconsole — los resultados quedan en la BD
msf6 > db_nmap -sV -sC -p- 192.168.1.100
```

### Convertir resultados XML con scripts

```bash
# Instalar xsltproc y convertir a HTML
xsltproc scan.xml -o scan.html

# Usar nmap-parse-output para extraer información
nmap-parse-output scan.xml hosts-with-port 80
nmap-parse-output scan.xml service http
```

### Combinar con EyeWitness

```bash
# Capturar screenshots de todos los servicios web encontrados
nmap -p 80,443,8080 --open -oX web_scan.xml 192.168.1.0/24
eyewitness --xml web_scan.xml -d screenshots/
```

---

## 14. Cheatsheet de referencia rápida

```bash
# ── DESCUBRIMIENTO ──────────────────────────────────────
nmap -sn 192.168.1.0/24                          # Ping sweep
nmap -Pn 192.168.1.100                           # Sin ping
sudo nmap -PR -sn 192.168.1.0/24                 # ARP ping

# ── ESCANEO DE PUERTOS ──────────────────────────────────
sudo nmap -sS -p- --open --min-rate 5000 <IP>    # Todos los puertos rápido
sudo nmap -sS --top-ports 1000 <IP>              # Top 1000 puertos
sudo nmap -sU --top-ports 200 <IP>               # UDP top 200
sudo nmap -sS -sU -p T:80,443,U:53,161 <IP>     # TCP + UDP juntos

# ── DETECCIÓN ───────────────────────────────────────────
nmap -sV <IP>                                    # Versiones de servicios
sudo nmap -O <IP>                                # Sistema operativo
nmap -A <IP>                                     # Todo (versión+OS+scripts)
nmap -sV --version-all <IP>                      # Máxima intensidad en versiones

# ── SCRIPTS NSE ─────────────────────────────────────────
nmap -sC <IP>                                    # Scripts por defecto
nmap --script vuln <IP>                          # Vulnerabilidades
nmap --script "smb-vuln-*" -p 445 <IP>          # Vulns SMB
nmap --script "http-*" -p 80,443 <IP>           # Scripts HTTP
nmap --script ftp-anon -p 21 <IP>               # FTP anónimo

# ── EVASIÓN ─────────────────────────────────────────────
sudo nmap -f <IP>                                # Fragmentar paquetes
sudo nmap -D RND:10 <IP>                         # 10 señuelos aleatorios
sudo nmap --source-port 53 <IP>                  # Puerto origen DNS
sudo nmap -T1 <IP>                               # Muy lento (sigiloso)
sudo nmap --data-length 25 <IP>                  # Payload aleatorio

# ── OUTPUT ──────────────────────────────────────────────
nmap -oA resultados <IP>                         # Todos los formatos
nmap -oN normal.txt <IP>                         # Formato normal
nmap -oX scan.xml <IP>                           # XML (para Metasploit)
nmap -oG grep.txt <IP>                           # Grepeable

# ── COMANDOS COMPLETOS PARA PENTESTING ──────────────────
# Reconocimiento inicial completo
sudo nmap -sCV -p- --open -T4 --min-rate 5000 -n -Pn <IP> -oA full_scan

# Detección de vulns en servicios comunes
sudo nmap -sV --script vuln -p 21,22,80,443,445,3389 <IP> -oA vuln_scan

# Enumeración web completa
sudo nmap -p 80,443,8080 --script http-enum,http-title,http-methods,http-waf-detect <IP>

# Enumeración SMB completa
sudo nmap -p 445 --script "smb-*" <IP>

# CTF one-liner
sudo nmap -p- --open -T4 --min-rate 5000 -n -Pn <IP> -oG - | grep open | grep -oP '\d+(?=/open)' | tr '\n' ',' | xargs -I{} sudo nmap -sCV -p{} <IP> -oA targeted
```
