---
layout: post
title: "Google Dorks — Búsquedas avanzadas para pentesters"
date: 2025-05-12
categories: [reconocimiento]
tags: [google-dorks, osint, recon, hacking, información, búsqueda]
description: "Guía completa de Google Dorks para reconocimiento ofensivo: operadores avanzados, dorks por categoría, automatización y uso profesional en pentesting."
---

## ¿Qué son los Google Dorks?

Los Google Dorks (también llamados Google Hacking) son búsquedas avanzadas que utilizan operadores especiales del buscador para encontrar información que normalmente no aparece en búsquedas convencionales. Fueron popularizados por Johnny Long con su base de datos **Google Hacking Database (GHDB)**, mantenida hoy en [Exploit-DB](https://www.exploit-db.com/google-hacking-database).

No explotan ninguna vulnerabilidad de Google — aprovechan información que está **públicamente indexada** pero que los propietarios no pretendían exponer. En un pentest profesional son una herramienta de reconocimiento pasivo de primer nivel.

> Todo el contenido es para uso en entornos autorizados o en tus propios sistemas.

---

## 1. Operadores fundamentales

### Operadores de restricción de dominio y URL

```
site:ejemplo.com
```
Limita los resultados al dominio especificado. Es el operador más usado en recon.

```bash
# Solo resultados de example.com
site:example.com

# Excluir el dominio principal — encontrar subdominios
site:*.example.com -site:www.example.com

# Subdirectorios específicos
site:example.com/admin
site:example.com/api
```

```
inurl:texto
```
Busca páginas donde la URL contiene el texto especificado.

```bash
inurl:admin
inurl:login
inurl:panel
inurl:dashboard
inurl:config
inurl:backup
```

```
allinurl:texto1 texto2
```
Todos los términos deben aparecer en la URL.

```bash
allinurl:admin login
allinurl:php?id=
```

### Operadores de contenido de página

```
intitle:texto
```
El texto debe aparecer en el título de la página.

```bash
intitle:"index of"
intitle:"admin panel"
intitle:"login" inurl:admin
```

```
allintitle:texto1 texto2
```
Todos los términos en el título.

```bash
allintitle:admin panel login
```

```
intext:texto
```
El texto debe aparecer en el cuerpo de la página.

```bash
intext:"password"
intext:"api_key"
intext:"BEGIN RSA PRIVATE KEY"
```

### Operadores de tipo de archivo

```
filetype:ext   (o ext:ext)
```

```bash
filetype:pdf
filetype:docx
filetype:xls
filetype:sql
filetype:env
filetype:log
filetype:bak
filetype:cfg
filetype:conf
filetype:ini
filetype:pem
filetype:key
```

### Operadores de caché y relacionados

```bash
# Ver versión en caché de una página
cache:example.com

# Páginas relacionadas con una URL
related:example.com

# Definición de una palabra
define:pentesting

# Rango numérico
precio 100..500 euros
```

### Operadores de exclusión y combinación

```bash
# Excluir término
site:example.com -inurl:www

# OR lógico
site:example.com filetype:pdf OR filetype:doc

# Frase exacta
"contraseña de administrador"

# Comodín
"admin * password"
```

---

## 2. Dorks por categoría

### Paneles de administración expuestos

```
intitle:"admin panel" site:example.com
intitle:"administration" inurl:admin
intitle:"login" inurl:/admin/
intitle:"Dashboard" inurl:dashboard
intitle:"Control Panel" inurl:cpanel
inurl:wp-admin site:example.com
inurl:administrator site:example.com
inurl:/phpmyadmin/ site:example.com
inurl:webmin site:example.com
intitle:"Plesk" inurl:8443
intitle:"cPanel" inurl:2083
intitle:"WHM" inurl:2086
intitle:"Webmin" inurl:10000
```

### Directorios con listado habilitado

```
intitle:"index of" site:example.com
intitle:"index of" "parent directory"
intitle:"index of" passwd
intitle:"index of" ".env"
intitle:"index of" "config.php"
intitle:"index of" "wp-config"
intitle:"index of" "backup"
intitle:"index of" "database"
intitle:"index of" site:example.com inurl:backup
```

### Archivos de configuración expuestos

```
site:example.com ext:env
site:example.com ext:cfg
site:example.com ext:conf
site:example.com ext:ini
site:example.com ext:xml "database"
site:example.com "db_password"
site:example.com "DB_PASSWORD"
site:example.com filetype:env "APP_KEY"
site:example.com filetype:cfg "password"
site:example.com filetype:ini "username"
```

### Credenciales y tokens en páginas web

```
site:example.com intext:"password" filetype:txt
site:example.com intext:"api_key"
site:example.com intext:"secret_key"
site:example.com intext:"access_token"
site:example.com intext:"BEGIN RSA PRIVATE KEY"
site:example.com intext:"BEGIN OPENSSH PRIVATE KEY"
site:example.com intext:"mysql_connect"
site:example.com intext:"mongodb://"
```

### Bases de datos y backups expuestos

```
site:example.com filetype:sql
site:example.com filetype:sql "INSERT INTO"
site:example.com filetype:sql "CREATE TABLE"
site:example.com filetype:bak
site:example.com filetype:backup
site:example.com ext:dump
site:example.com "database backup"
intitle:"index of" "*.sql"
intitle:"index of" "*.sql.gz"
intitle:"index of" "dump.sql"
```

### Logs y archivos de debug

```
site:example.com filetype:log
site:example.com ext:log "error"
site:example.com filetype:log "password"
site:example.com intext:"Traceback" filetype:txt
site:example.com intext:"Fatal error" filetype:php
site:example.com intext:"Warning: mysql"
site:example.com intext:"Stack trace:"
intitle:"index of" "error.log"
intitle:"index of" "access.log"
```

### Documentos con información sensible

```
site:example.com filetype:pdf "confidencial"
site:example.com filetype:pdf "internal use only"
site:example.com filetype:xls "username" "password"
site:example.com filetype:xls "employees"
site:example.com filetype:docx "salary"
site:example.com filetype:ppt "strategy" "confidential"
site:example.com filetype:pdf "board meeting"
site:example.com filetype:xlsx "social security"
```

### Cámaras y dispositivos IoT

```
inurl:"/view/view.shtml"
inurl:"ViewerFrame?Mode="
intitle:"Network Camera" inurl:main.cgi
intitle:"webcamXP 5"
inurl:"/mjpg/video.mjpg"
intitle:"Live View / - AXIS"
inurl:axis-cgi/jpg
intitle:"Trendnet" inurl:view
intitle:"IP CAMERA Viewer"
intitle:"IPCamera_Logo" inurl:main.cgi
```

### Servidores y servicios expuestos

```
intitle:"Apache2 Ubuntu Default Page"
intitle:"Welcome to nginx!"
intitle:"IIS Windows Server"
intitle:"Tomcat" "Apache Tomcat"
intitle:"Kibana" inurl:5601
intitle:"Grafana" inurl:3000
intitle:"Jenkins" inurl:8080
intitle:"GitLab" site:example.com
intitle:"Jupyter" inurl:8888
intitle:"phpMyAdmin" inurl:phpmyadmin
intitle:"Adminer" filetype:php
```

### Páginas de error que revelan información

```
site:example.com intext:"sql syntax near"
site:example.com intext:"syntax error has occurred"
site:example.com intext:"Unclosed quotation mark"
site:example.com intext:"Microsoft OLE DB Provider"
site:example.com intext:"ODBC Microsoft Access"
site:example.com intext:"Warning: pg_connect()"
site:example.com intext:"supplied argument is not a valid MySQL"
site:example.com intext:"ORA-" filetype:php
```

### GitHub y repositorios de código

```
site:github.com "example.com" password
site:github.com "example.com" secret
site:github.com "example.com" api_key
site:github.com "example.com" "private_key"
site:github.com "@example.com" password
site:github.com "example.com" token
site:github.com "example.com" "db_password"
site:github.com "example.com" "smtp_password"
site:github.com "example.com" ".env"
site:github.com "example.com" "config.yml"
site:github.com org:example-corp secret
site:gitlab.com "example.com" password
site:bitbucket.org "example.com" password
```

### Pastebin y paste sites

```
site:pastebin.com "example.com"
site:pastebin.com "@example.com" password
site:pastebin.com "example.com" "admin"
site:paste.ee "example.com"
site:ghostbin.com "example.com"
site:hastebin.com "example.com"
```

### VPNs, acceso remoto y portales corporativos

```
site:example.com inurl:vpn
site:example.com inurl:remote
site:example.com inurl:portal
site:example.com inurl:webmail
site:example.com inurl:owa  # Outlook Web Access
intitle:"Citrix" site:example.com
intitle:"GlobalProtect" site:example.com
intitle:"Pulse Secure" site:example.com
intitle:"Fortinet" inurl:remote
```

---

## 3. Google Hacking Database (GHDB)

La GHDB en Exploit-DB es la referencia definitiva. Tiene miles de dorks categorizados:

```
Categorías en GHDB:
├── Footholds               → Acceso inicial a sistemas
├── Files Containing Usernames   → Archivos con usuarios
├── Sensitive Directories   → Directorios sensibles
├── Web Server Detection    → Fingerprinting de servidores
├── Vulnerable Files        → Archivos con vulnerabilidades conocidas
├── Vulnerable Servers      → Servidores con vulns
├── Error Messages          → Mensajes de error informativos
├── Files Containing Passwords  → Archivos con contraseñas
├── Sensitive Online Shopping   → E-commerce
├── Network or Vulnerability Data  → Datos de red
├── Pages Containing Login Portals → Portales de login
├── Various Online Devices  → Dispositivos varios
└── Advisories and Vulnerabilities → Vulnerabilidades
```

```bash
# Acceder a GHDB
https://www.exploit-db.com/google-hacking-database

# Filtrar por categoría
# https://www.exploit-db.com/google-hacking-database?category=1
```

---

## 4. Automatización de Google Dorks

### googlesearch-python

```python
from googlesearch import search
import time

target = "example.com"

dorks = [
    f'site:{target} filetype:env',
    f'site:{target} filetype:sql',
    f'site:{target} intitle:"index of"',
    f'site:{target} inurl:admin',
    f'site:{target} filetype:log',
    f'site:github.com "{target}" password',
    f'site:pastebin.com "{target}"',
    f'site:{target} intext:"api_key"',
]

resultados = {}

for dork in dorks:
    print(f"\n[+] {dork}")
    resultados[dork] = []
    try:
        for url in search(dork, num_results=10, sleep_interval=3):
            print(f"    {url}")
            resultados[dork].append(url)
        time.sleep(5)  # Evitar rate limiting
    except Exception as e:
        print(f"    Error: {e}")

# Guardar resultados
with open("dorks_results.txt", "w") as f:
    for dork, urls in resultados.items():
        f.write(f"\n## {dork}\n")
        for url in urls:
            f.write(f"{url}\n")

print("\n[*] Resultados guardados en dorks_results.txt")
```

### GoogD0rker — herramienta dedicada

```bash
# Instalar
pip3 install googd0rker

# Uso básico
googd0rker -d example.com

# Con archivo de dorks personalizado
googd0rker -d example.com -w mi_dorks.txt
```

### DorkScout

```bash
git clone https://github.com/IvanGlinkin/DorkScout
cd DorkScout
pip3 install -r requirements.txt
python3 dorkscout.py -d example.com
```

### Pagodo — Google Dorks pasivo con GHDB

```bash
git clone https://github.com/opsdisk/pagodo
cd pagodo

# Descargar la GHDB completa
python3 ghdb_scraper.py -j

# Ejecutar dorks contra un dominio
python3 pagodo.py -d example.com -g dorks.txt -l 50 -s -e 35.0
```

---

## 5. Dorks específicos por tecnología

### WordPress

```
site:example.com inurl:wp-content
site:example.com inurl:wp-includes
site:example.com inurl:wp-login
site:example.com filetype:log inurl:wp-content
site:example.com inurl:wp-config.php
inurl:wp-content/uploads filetype:php
intitle:"WordPress" inurl:wp-login.php
```

### Joomla

```
site:example.com inurl:administrator
site:example.com inurl:com_content
site:example.com "Joomla! Debug Console"
inurl:index.php?option=com_users
```

### Drupal

```
site:example.com inurl:user/login
site:example.com inurl:/node/add
intitle:"Drupal" inurl:/user/login
site:example.com intext:"Powered by Drupal"
```

### Laravel / PHP

```
site:example.com filetype:env APP_KEY
site:example.com intext:"APP_DEBUG=true"
site:example.com "Whoops! There was an error"
site:example.com intext:"The Exception occurred"
site:example.com filetype:php "mysqli_connect"
```

### Spring Boot / Java

```
site:example.com inurl:actuator
site:example.com inurl:actuator/env
site:example.com inurl:actuator/heapdump
intitle:"Whitelabel Error Page" inurl:spring
```

### AWS y Cloud

```
site:example.com filetype:pem
site:example.com intext:"AKIA"  # AWS Access Key IDs
site:github.com "example.com" "AKIA"
site:github.com "s3.amazonaws.com" "example"
site:example.com inurl:s3.amazonaws.com
"s3.amazonaws.com/example"
intitle:"index of" "s3.amazonaws.com"
```

### Elasticsearch y bases de datos NoSQL

```
intitle:"Kibana" inurl:5601 site:example.com
site:example.com inurl:9200/_cat/indices
site:example.com inurl:27017
site:example.com inurl:6379
```

---

## 6. Dorking en otros buscadores

Google no es el único. Cada buscador indexa contenido diferente.

### Bing Dorks

```bash
# Bing usa operadores similares
site:example.com filetype:pdf
ip:93.184.216.34          # Buscar por IP — exclusivo de Bing
```

### DuckDuckGo

```bash
site:example.com
filetype:pdf site:example.com
```

### Yandex

```bash
# Indexa mucho contenido ruso y europeo
site:example.com
url:example.com/admin
```

### Baidu

```bash
# Útil para objetivos con presencia en China
site:example.com
inurl:admin site:example.com
```

### Shodan (como buscador de dorks)

```bash
# Shodan tiene su propio sistema de dorks
hostname:example.com
org:"Example Corp"
ssl.cert.subject.cn:example.com
```

---

## 7. Dorks de inteligencia competitiva

No todo el Google Hacking es ofensivo. En un contexto de business intelligence o red team:

```
# Tecnologías y proveedores
site:example.com "Powered by Salesforce"
site:example.com "Powered by SAP"
site:example.com intext:"cdn.cloudflare.com"
site:example.com intext:"amazonaws.com"

# Menciones en medios
"example corp" site:businesswire.com
"example corp" site:prnewswire.com

# Licitaciones públicas (España)
"example corp" site:contrataciondelestado.es
"example corp" site:boe.es

# Patentes
site:patents.google.com assignee:"Example Corp"
site:espacenet.com "Example Corp"

# Ofertas de trabajo → revelan tecnologías internas
site:linkedin.com/jobs "example corp" "kubernetes"
site:indeed.com "example corp" developer
site:infojobs.net "example corp"
```

---

## 8. Defensas contra Google Dorks

Si eres el defensor, aquí tienes cómo protegerte:

### robots.txt

```
# Directorios que no deben indexarse
User-agent: *
Disallow: /admin/
Disallow: /backup/
Disallow: /config/
Disallow: /api/internal/
Disallow: /staging/
```

> Ojo: `robots.txt` **no protege** — le dice a los bots bien portados que no indexen, pero los atacantes lo ignorarán. Es útil para el SEO, no para la seguridad.

### Auditoría de lo que Google sabe de ti

```bash
# Comprobar qué tiene indexado Google de tu dominio
site:tudominio.com

# Buscar archivos sensibles de tu propio dominio
site:tudominio.com filetype:env OR filetype:sql OR filetype:log OR filetype:bak

# Buscar tu empresa en paste sites
site:pastebin.com "tudominio.com"
site:github.com "tudominio.com" password

# Alertas de Google — monitoreo continuo
# https://alerts.google.com
# Crear alerta para: "tudominio.com" confidential
```

### Google Search Console

```bash
# Solicitar eliminación de URLs indexadas
# https://search.google.com/search-console
# Herramientas → Eliminación de URLs

# Para emergencias (credenciales expuestas)
# https://support.google.com/webmasters/troubleshooter/6366738
```

---

## 9. Cheatsheet de referencia rápida

```bash
# ── OPERADORES BÁSICOS ──────────────────────────────────
site:ejemplo.com                   # Limitar al dominio
intitle:"texto"                    # Texto en el título
inurl:admin                        # Texto en la URL
intext:"password"                  # Texto en el cuerpo
filetype:pdf                       # Tipo de archivo
-término                           # Excluir resultado
"frase exacta"                     # Búsqueda exacta

# ── TOP DORKS DE RECONOCIMIENTO ─────────────────────────
site:ejemplo.com filetype:env                    # Variables de entorno
site:ejemplo.com filetype:sql                    # Bases de datos
site:ejemplo.com intitle:"index of"              # Directorios expuestos
site:ejemplo.com inurl:admin                     # Paneles admin
site:ejemplo.com filetype:log                    # Logs expuestos
site:ejemplo.com filetype:bak                    # Backups
site:ejemplo.com intext:"api_key"                # API keys
site:github.com "ejemplo.com" password           # Credenciales en GitHub
site:pastebin.com "ejemplo.com"                  # Pastes
site:*.ejemplo.com -site:www.ejemplo.com         # Subdominios

# ── GHDB ────────────────────────────────────────────────
# https://www.exploit-db.com/google-hacking-database

# ── AUTOMATIZACIÓN ──────────────────────────────────────
pip3 install googlesearch-python
pip3 install googd0rker
```

---

## 10. Recursos adicionales

- **Google Hacking Database** — [exploit-db.com/google-hacking-database](https://www.exploit-db.com/google-hacking-database)
- **Google Advanced Search** — [google.com/advanced_search](https://www.google.com/advanced_search)
- **Libro**: *Google Hacking for Penetration Testers* — Johnny Long
- **DorkSearch** — [dorksearch.com](https://dorksearch.com) — buscador de dorks con plantillas
- **Pentest-Tools Google Hacker** — [pentest-tools.com/information-gathering/google-hacking](https://pentest-tools.com/information-gathering/google-hacking)
