---
layout: post
title: "Reconocimiento Pasivo con OSINT Framework"
date: 2025-05-13
categories: [reconocimiento]
tags: [osint, reconocimiento, recon, maltego, shodan, theHarvester, google-dorks, metadatos, osint-framework]
description: "Guía profesional de reconocimiento pasivo con OSINT: metodología, herramientas, fuentes de inteligencia y flujos reales usados en Red Team y Bug Bounty."
---

## ¿Qué es el reconocimiento pasivo?

El reconocimiento pasivo es la primera fase de cualquier operación de seguridad ofensiva. A diferencia del reconocimiento activo, **no se envía ningún paquete directamente al objetivo** — toda la información se obtiene de fuentes públicas sin que el objetivo pueda detectar que está siendo investigado.

En un entorno profesional, esta fase puede durar días o incluso semanas. La calidad del reconocimiento determina directamente el éxito del resto del engagement. Un recon pobre = vectores de ataque perdidos. Un recon exhaustivo = superficie de ataque completa antes de tocar el primer puerto.

### Diferencia entre reconocimiento pasivo y activo

| Tipo | Interacción con el objetivo | Detectable |
|------|----------------------------|------------|
| Pasivo | Ninguna — fuentes externas | No |
| Semi-pasivo | Mínima — tráfico normal | Casi no |
| Activo | Directa — escaneos, peticiones | Sí |

Este post cubre el reconocimiento **pasivo y semi-pasivo**. El activo (Nmap, Nikto, etc.) tiene su propia guía.

---

## 1. OSINT Framework — El mapa del territorio

[OSINT Framework](https://osintframework.com) es un árbol interactivo que categoriza cientos de herramientas y fuentes de inteligencia de fuentes abiertas. No es una herramienta en sí — es una guía de recursos organizada por tipo de información que buscas.

### Categorías principales

```
OSINT Framework
├── Username                → Buscar usuarios en múltiples plataformas
├── Email Address           → Verificar, rastrear y enumerar emails
├── Domain Name             → Whois, DNS, subdominios, tecnologías
├── IP Address              → Geolocalización, ASN, historial, abuso
├── Social Networks         → Twitter, LinkedIn, Facebook, Instagram
├── Instant Messaging       → Telegram, Discord, Slack
├── People Search Engines   → Spokeo, Pipl, WhitePages
├── Dating Sites            → Perfiles en apps de citas
├── Telephone               → OSINT de números de teléfono
├── Public Records          → Registros públicos, judiciales, empresariales
├── Business Records        → Empresas, directivos, filiales
├── Transportation          → Matrículas, barcos, aviones
├── Geospatial              → Google Maps, Satellites, Streetview
├── Images / Videos         → Búsqueda inversa, metadata
├── Documents               → Metadatos de PDFs, DOCx, imágenes
├── Forums / Blogs          → Rastros digitales en comunidades
├── Dark Web                → .onion sites, mercados, forums
├── Digital Currency        → Rastreo de wallets crypto
└── Threat Intelligence     → IOCs, malware, C2 infrastructure
```

### Cómo usarlo en un engagement real

En un pentest profesional el flujo típico es:

```
1. Recibir el scope (dominios, IPs, rangos autorizados)
2. Abrir OSINT Framework
3. Explorar cada categoría relevante para el objetivo
4. Documentar todos los hallazgos en un repositorio central (Notion, Obsidian, CherryTree)
5. Cruzar datos entre herramientas para obtener relaciones
```

---

## 2. Metodología profesional de OSINT

Antes de abrir cualquier herramienta, define la metodología. En Red Team profesional se usa el modelo **OASIS** o derivados del ciclo de inteligencia clásico.

### El ciclo de inteligencia aplicado a OSINT

```
┌─────────────────────────────────────────────────────┐
│                CICLO DE INTELIGENCIA                 │
│                                                      │
│  1. PLANIFICACIÓN     → ¿Qué necesito saber?         │
│  2. RECOLECCIÓN       → ¿Dónde está esa info?        │
│  3. PROCESAMIENTO     → Normalizar y limpiar datos   │
│  4. ANÁLISIS          → Extraer conclusiones         │
│  5. DISEMINACIÓN      → Documentar y reportar        │
│  6. RETROALIMENTACIÓN → ¿Qué falta? Iterar           │
└─────────────────────────────────────────────────────┘
```

### Definir el scope y objetivos de inteligencia

Antes de empezar, responde estas preguntas:

- ¿Cuál es el objetivo primario? (empresa, persona, infraestructura)
- ¿Qué información necesito? (empleados, emails, tecnologías, proveedores)
- ¿Cuál es el objetivo final? (phishing, acceso inicial, escalada)
- ¿Qué está fuera de scope?

### Estructura de documentación recomendada

```
recon/
├── target_overview.md          # Resumen del objetivo
├── domains/
│   ├── main_domain.md          # Dominio principal
│   ├── subdomains.txt          # Lista de subdominios
│   └── whois.txt               # Registros WHOIS
├── emails/
│   ├── emails_found.txt        # Emails recolectados
│   └── email_patterns.md      # Patrones identificados
├── employees/
│   ├── linkedin.md             # Empleados de LinkedIn
│   └── org_chart.md            # Organigrama inferido
├── infrastructure/
│   ├── ips.txt                 # IPs y rangos
│   ├── asn.txt                 # Números ASN
│   └── technologies.md        # Stack tecnológico
├── credentials/
│   └── leaked.txt              # Credenciales en brechas
└── screenshots/                # Capturas de evidencias
```

---

## 3. Reconocimiento de dominios

### WHOIS — Información de registro

```bash
# WHOIS básico
whois example.com

# WHOIS de IP
whois 93.184.216.34

# Con herramienta específica
whois -h whois.iana.org example.com
```

**Qué buscar en WHOIS:**
- Registrant name y organización (puede estar oculto por privacidad)
- Registrar y fechas de creación/expiración
- Name servers — revelan el proveedor DNS
- Email de contacto — patrón de emails corporativos
- Dirección física — información de la empresa

**Herramientas online alternativas:**
- [who.is](https://who.is)
- [domaintools.com](https://domaintools.com)
- [viewdns.info](https://viewdns.info)

### Búsqueda de subdominios pasiva

Los subdominios revelan infraestructura interna, entornos de desarrollo, servicios expuestos y más.

```bash
# Subfinder — herramienta más completa
subfinder -d example.com -o subdominios.txt
subfinder -d example.com -all -recursive  # Modo agresivo

# Amass — el más exhaustivo (tarda más)
amass enum -passive -d example.com -o amass_output.txt
amass enum -passive -d example.com -src  # Muestra la fuente de cada subdominio

# assetfinder
assetfinder --subs-only example.com

# Combinar todas las fuentes y deduplicar
cat amass_output.txt subfinder_output.txt | sort -u > all_subs.txt
```

**Fuentes pasivas que usan estas herramientas:**
- Certificate Transparency logs (crt.sh)
- Wayback Machine / Common Crawl
- VirusTotal, SecurityTrails, Shodan
- DNSdumpster, HackerTarget
- RapidDNS, BufferOver

### Certificate Transparency — crt.sh

Los certificados SSL/TLS son públicos. Los CT logs registran todos los certificados emitidos y revelan subdominios que nunca aparecen en DNS público.

```bash
# Consultar crt.sh directamente
curl -s "https://crt.sh/?q=%.example.com&output=json" | jq '.[].name_value' | sed 's/"//g' | sort -u

# Con Python
python3 -c "
import requests, json
r = requests.get('https://crt.sh/?q=%.example.com&output=json')
subs = set()
for cert in r.json():
    for name in cert['name_value'].split('\n'):
        subs.add(name.strip())
for s in sorted(subs):
    print(s)
"
```

### DNS histórico y pasivo

```bash
# SecurityTrails (requiere API key gratis)
curl -H "apikey: TU_API_KEY" "https://api.securitytrails.com/v1/domain/example.com/subdomains"

# DNSdumpster — sin API
# https://dnsdumpster.com

# Passive DNS con CIRCL
curl -s "https://www.circl.lu/pdns/query/example.com" | jq .

# VirusTotal subdomains
curl -s "https://www.virustotal.com/vtapi/v2/domain/report?apikey=TU_KEY&domain=example.com" | jq '.subdomains'
```

### Transferencia de zona DNS (semi-activo)

Si el servidor DNS está mal configurado, puede revelar toda la zona:

```bash
# Intentar AXFR
dig axfr example.com @ns1.example.com
dig axfr @ns1.example.com example.com

# Con host
host -l example.com ns1.example.com

# Con fierce (automatizado)
fierce --domain example.com
```

---

## 4. Reconocimiento de emails

Los emails son el vector principal de phishing y permiten inferir el patrón de nomenclatura corporativa.

### theHarvester — Recolección masiva

```bash
# Instalar
pip3 install theHarvester

# Búsqueda básica con múltiples motores
theHarvester -d example.com -b google,bing,duckduckgo,yahoo -l 500

# Con Shodan integrado
theHarvester -d example.com -b shodan -l 100

# Todos los motores disponibles
theHarvester -d example.com -b all -l 500 -f resultados

# Motores útiles
# google, bing, yahoo, duckduckgo — búsquedas web
# linkedin — empleados
# shodan — dispositivos
# hunter — emails profesionales
# anubis, dnsdumpster — subdominios
```

**Output que genera:**
```
[*] Emails found: 12
------------------
jsmith@example.com
admin@example.com
info@example.com
[...]

[*] Hosts found: 34
-------------------
mail.example.com:93.184.216.34
dev.example.com:93.184.216.100
[...]
```

### Hunter.io — Emails corporativos

[Hunter.io](https://hunter.io) especializado en emails profesionales. Tiene API gratuita limitada.

```bash
# Con cURL
curl "https://api.hunter.io/v2/domain-search?domain=example.com&api_key=TU_KEY"

# Con Python
import requests
r = requests.get("https://api.hunter.io/v2/domain-search",
    params={"domain": "example.com", "api_key": "TU_KEY"})
data = r.json()
for email in data['data']['emails']:
    print(email['value'], email['type'])
```

### Identificar el patrón de emails

Una vez tienes varios emails, infiere el patrón corporativo:

```
john.smith@company.com        → firstname.lastname
jsmith@company.com            → firstinitial+lastname
john_smith@company.com        → firstname_lastname
johns@company.com             → firstname+lastinitial
```

Con el patrón y la lista de empleados de LinkedIn puedes generar emails válidos para toda la organización.

```python
# Script para generar emails desde lista de nombres
names = [
    "John Smith",
    "María García",
    "Carlos López"
]

domain = "example.com"
pattern = "firstname.lastname"  # Cambiar según el patrón

for name in names:
    parts = name.lower().split()
    first = parts[0]
    last = parts[-1] if len(parts) > 1 else ""

    if pattern == "firstname.lastname":
        print(f"{first}.{last}@{domain}")
    elif pattern == "firstinitial+lastname":
        print(f"{first[0]}{last}@{domain}")
    elif pattern == "firstname_lastname":
        print(f"{first}_{last}@{domain}")
```

### Verificar emails sin enviar nada

```bash
# Verificar existencia por SMTP (sin enviar email)
# smtp-user-enum
smtp-user-enum -M VRFY -U emails.txt -t mail.example.com

# Con Python directamente
python3 -c "
import smtplib
s = smtplib.SMTP('mail.example.com')
s.helo('test.com')
result = s.verify('jsmith@example.com')
print(result)
"
```

---

## 5. LinkedIn OSINT — Inteligencia corporativa

LinkedIn es la fuente más rica de información corporativa: empleados, tecnologías usadas, proyectos, organigramas, proveedores y mucho más.

### Enumeración manual sin cuenta

```
https://www.linkedin.com/company/example/people/
https://www.linkedin.com/company/example/jobs/
```

### google dorks para LinkedIn

```
site:linkedin.com "example company" "engineer"
site:linkedin.com/in "example company" "security"
site:linkedin.com "example company" "works at" "developer"
```

### linkedin2username — Generar listas de usuarios

```bash
# Instalar
git clone https://github.com/initstring/linkedin2username
cd linkedin2username
pip3 install -r requirements.txt

# Ejecutar (requiere cuenta LinkedIn)
python3 linkedin2username.py -u tu_usuario -c "Nombre de la Empresa"

# Output genera variantes de nombres:
# jsmith, john.smith, j.smith, smithj, etc.
```

### Información de tecnologías por ofertas de trabajo

Las ofertas de trabajo revelan el stack tecnológico interno sin buscarlo activamente:

**Qué buscar en las ofertas:**
- Lenguajes y frameworks en uso (Java Spring, .NET, Python Django)
- Infraestructura cloud (AWS, Azure, GCP)
- Herramientas DevOps (Jenkins, GitLab CI, Terraform)
- Productos de seguridad (Splunk, CrowdStrike, Palo Alto)
- Bases de datos (Oracle, MySQL, PostgreSQL, MongoDB)
- Versiones específicas mencionadas

```bash
# Automatizar con Google
site:linkedin.com/jobs "example company" "AWS" "Kubernetes"
site:linkedin.com/jobs "example company" "security engineer"

# Indeed y Glassdoor también son fuentes válidas
site:indeed.com "example company" developer
```

---

## 6. Shodan — El buscador de dispositivos expuestos

Shodan indexa dispositivos conectados a internet: servidores, cámaras, routers, SCADA, IoT, y cualquier cosa con IP pública. A diferencia de un escaner de puertos, **Shodan no escanea el objetivo** — consulta su base de datos de escaneos previos.

### Búsquedas básicas

```
# Por organización
org:"Example Corp"

# Por nombre de host
hostname:example.com

# Por rango de IPs o ASN
net:93.184.216.0/24
asn:AS15169

# Por producto
product:"Apache httpd"
product:"nginx" version:"1.18"

# Por puerto
port:8080 org:"Example"
port:22 country:"ES"

# Por banner/tecnología
"X-Powered-By: PHP/5" org:"Example"
title:"Dashboard" org:"Example"

# SSL certificado
ssl.cert.subject.CN:example.com
ssl:"Example Corp"

# Combinadas
org:"Example Corp" port:22 country:"US"
hostname:example.com port:443 product:"nginx"
```

### Shodan desde CLI

```bash
# Instalar
pip3 install shodan

# Inicializar con API key (gratis en shodan.io)
shodan init TU_API_KEY

# Buscar
shodan search "org:Example Corp"
shodan search --fields ip_str,port,org,hostname "nginx"

# Info de una IP específica
shodan host 93.184.216.34

# Enumerar todos los servicios de una organización
shodan search --limit 1000 "org:Example Corp" --fields ip_str,port,transport,product > shodan_results.txt

# Monitorear cambios en una IP
shodan alert create --name "Example Alert" --ip 93.184.216.34
```

### Shodan con Python

```python
import shodan

api = shodan.Shodan("TU_API_KEY")

# Buscar servicios de una organización
results = api.search('org:"Example Corp"')
print(f"Resultados: {results['total']}")

for result in results['matches']:
    print(f"""
    IP:       {result['ip_str']}
    Puerto:   {result['port']}
    OS:       {result.get('os', 'Desconocido')}
    Org:      {result.get('org', 'N/A')}
    Banner:   {result['data'][:100]}
    """)

# Info completa de una IP
host = api.host("93.184.216.34")
print(f"OS: {host.get('os', 'Desconocido')}")
for service in host['data']:
    print(f"Port: {service['port']}/tcp — {service.get('product', '?')}")
```

### Filtros avanzados de Shodan

```
# Versiones específicas vulnerables
product:"OpenSSH" version:"7.2"
product:"Apache" version:"2.2"

# Paneles de administración expuestos
title:"admin" port:80
title:"phpMyAdmin" port:80
http.title:"Kibana" port:5601
http.title:"Grafana"

# Bases de datos expuestas
product:"MongoDB" port:27017
product:"Elasticsearch" port:9200
product:"Redis" port:6379 -auth

# Cámaras IP
product:"webcamXP"
title:"Network Camera"

# SCADA / ICS
product:"Modbus"
"Schneider Electric"

# Servidores con certificados expirados
ssl.cert.expired:true org:"Example"

# Tecnologías con vulnerabilidades conocidas
vuln:CVE-2021-44228   # Log4Shell
vuln:CVE-2017-0144    # EternalBlue
```

---

## 7. Metadatos de documentos — FOCA y Exiftool

Los documentos publicados en internet (PDFs, DOCx, XLS, imágenes) contienen metadatos que revelan información interna valiosa.

### Qué revelan los metadatos

- **Autor y empresa** — confirma empleados y organización
- **Software y versión** — "Microsoft Word 2016" → inferir parches
- **Rutas internas** — `C:\Users\jsmith\Documents\informe_financiero.docx`
- **Nombres de equipos** — `CORP-PC-JSMITH`
- **Servidor de impresión** — `\\print-server-01\HP_LaserJet`
- **Usernames del sistema** — coinciden con credenciales AD
- **GPS en imágenes** — localización exacta donde se tomó la foto
- **Timestamps** — horarios de trabajo, zonas horarias

### Recolección de documentos con Google Dorks

```
# PDFs del dominio
site:example.com filetype:pdf
site:example.com filetype:doc OR filetype:docx
site:example.com filetype:xls OR filetype:xlsx
site:example.com filetype:ppt OR filetype:pptx

# Documentos con información sensible
site:example.com filetype:pdf "confidencial"
site:example.com filetype:xls "password"
site:example.com filetype:pdf "internal use only"
```

### Exiftool — Extracción de metadatos

```bash
# Instalar
apt install exiftool

# Analizar un archivo
exiftool documento.pdf
exiftool imagen.jpg

# Campos más relevantes
exiftool -Author -Creator -Producer -CreationDate documento.pdf

# Procesar directorio completo
exiftool /ruta/documentos/ -csv > metadatos.csv

# Solo campos con información de usuario
exiftool -Author -Creator -LastModifiedBy -Company *.docx

# Extraer coordenadas GPS de imágenes
exiftool -GPSLatitude -GPSLongitude foto.jpg
exiftool -n -GPSLatitude -GPSLongitude foto.jpg  # Formato decimal

# Eliminar todos los metadatos (cuando eres el defensor)
exiftool -all= documento.pdf
```

### FOCA — Análisis masivo de metadatos

FOCA (Fingerprinting Organizations with Collected Archives) automatiza la recolección y análisis de metadatos de documentos públicos.

```
Características:
- Descarga automática de documentos del dominio objetivo
- Análisis batch de metadatos
- Generación de mapa de red interno basado en rutas
- Identificación de usuarios, software y servidores
- Correlación de información entre documentos
```

**Flujo de uso:**

```
1. Crear proyecto con el dominio objetivo
2. FOCA busca en Google, Bing y Exalead documentos del dominio
3. Descarga automática de PDFs, DOCx, XLS, etc.
4. Extracción y análisis de metadatos de todos los archivos
5. Generación de árbol de usuarios, equipos y rutas internas
```

### Análisis manual de metadatos con Python

```python
import os
import subprocess
import json

def extract_metadata(filepath):
    """Extrae metadatos con exiftool y devuelve JSON"""
    result = subprocess.run(
        ['exiftool', '-json', filepath],
        capture_output=True, text=True
    )
    return json.loads(result.stdout)[0] if result.stdout else {}

def analyze_documents(directory):
    """Analiza todos los documentos de un directorio"""
    interesting_fields = [
        'Author', 'Creator', 'LastModifiedBy', 'Company',
        'Producer', 'CreatorTool', 'GPSLatitude', 'GPSLongitude',
        'OwnerName', 'Template', 'LastPrinter'
    ]

    findings = {
        'authors': set(),
        'companies': set(),
        'software': set(),
        'internal_paths': set(),
        'printers': set(),
    }

    for filename in os.listdir(directory):
        filepath = os.path.join(directory, filename)
        meta = extract_metadata(filepath)

        if meta.get('Author'):
            findings['authors'].add(meta['Author'])
        if meta.get('Company'):
            findings['companies'].add(meta['Company'])
        if meta.get('Creator') or meta.get('CreatorTool'):
            findings['software'].add(meta.get('Creator') or meta.get('CreatorTool'))
        if meta.get('LastPrinter'):
            findings['printers'].add(meta['LastPrinter'])

        print(f"\n[{filename}]")
        for field in interesting_fields:
            if meta.get(field):
                print(f"  {field}: {meta[field]}")

    print("\n[RESUMEN]")
    print(f"Autores únicos: {findings['authors']}")
    print(f"Empresas: {findings['companies']}")
    print(f"Software: {findings['software']}")
    print(f"Impresoras: {findings['printers']}")

analyze_documents("./documentos/")
```

---

## 8. Reconocimiento de infraestructura

### Identificar rangos de IP y ASN

```bash
# Buscar el ASN de una empresa
whois -h whois.radb.net "!gAS15169"

# Con bgp.he.net (Hurricane Electric)
# https://bgp.he.net/search?search[search]=example+corp

# Herramienta bgpview
curl -s "https://api.bgpview.io/search?query_term=Example+Corp" | jq '.data.asns[] | {asn, name, description}'

# Obtener rangos de IPs de un ASN
curl -s "https://api.bgpview.io/asn/15169/prefixes" | jq '.data.ipv4_prefixes[].prefix'

# Con whois
whois -h whois.arin.net "org:Example Corp"
```

### IP History — Historial de cambios

Las IPs cambian. El historial revela infraestructura antigua, proveedores anteriores y posibles configuraciones olvidadas.

```bash
# SecurityTrails IP History
curl -H "apikey: TU_KEY" "https://api.securitytrails.com/v1/history/example.com/dns/a"

# ViewDNS IP History
# https://viewdns.info/iphistory/?domain=example.com

# RiskIQ (ahora Microsoft Defender TI)
# Registrar en community.riskiq.com — plan gratuito
```

### Identificar el CDN y la IP real detrás

Muchos sitios usan Cloudflare u otros CDN. Encontrar la IP real detrás del CDN permite escanear el servidor directamente.

```bash
# Buscar registros históricos antes del CDN
# https://securitytrails.com (historial DNS)

# Subdominios que no pasan por CDN
# mail.example.com, ftp.example.com, vpn.example.com
# suelen apuntar directamente al servidor

# Certificados SSL — buscar la IP real
# https://censys.io → buscar por certificado del dominio

# shodan
ssl.cert.subject.CN:example.com

# Headers de respuesta que revelan el origen
curl -I https://example.com | grep -i "server\|x-powered\|x-backend\|via"
```

### Wayback Machine — Infraestructura del pasado

```bash
# Todas las URLs capturadas de un dominio
curl -s "http://web.archive.org/cdx/search/cdx?url=example.com/*&output=text&fl=original&collapse=urlkey" | sort -u > wayback_urls.txt

# Filtrar por tipo de archivo
curl -s "http://web.archive.org/cdx/search/cdx?url=example.com/*.js&output=text&fl=original&collapse=urlkey" > wayback_js.txt

# Buscar endpoints API históricos
grep -E "api|v1|v2|admin|login|config" wayback_urls.txt

# Con waybackurls (Go)
go install github.com/tomnomnom/waybackurls@latest
echo "example.com" | waybackurls > wayback_all.txt

# Con gau (Get All URLs)
gau example.com > all_urls.txt
```

---

## 9. Inteligencia sobre brechas de datos

Una de las fuentes más valiosas en reconocimiento: credenciales filtradas en brechas anteriores.

### Have I Been Pwned

```bash
# API (requiere key de pago para dominio completo)
curl -H "hibp-api-key: TU_KEY" "https://haveibeenpwned.com/api/v3/breachesforaccount/jsmith@example.com"

# Verificar por hash de contraseña (k-anonymity)
# 1. Calcular SHA1 del password
echo -n "password123" | sha1sum
# cbfdac6008f9cab4083784cbd1874f76618d2a97

# 2. Enviar solo los 5 primeros caracteres
curl "https://api.pwnedpasswords.com/range/CBFDA"
# Devuelve todos los hashes que empiezan por CBFDA
```

### Dehashed — Búsqueda de credenciales filtradas

[Dehashed.com](https://dehashed.com) — base de datos de brechas con búsqueda por dominio, email, usuario, IP, nombre.

```bash
# Con API (requiere cuenta)
curl -H "Accept: application/json" \
     -u "email:api_key" \
     "https://api.dehashed.com/search?query=domain:example.com"
```

**Qué buscar en Dehashed:**
- Emails corporativos + contraseñas → credenciales válidas para VPN o webmail
- Hashes MD5/SHA1 → crackear offline con Hashcat
- Nombres de usuario → patrones para ataques de fuerza bruta
- IPs internas filtradas → esquema de red

### Otros repositorios de brechas

```bash
# IntelX (Intelligence X)
# https://intelx.io — busca en paste sites, dark web, etc.

# BreachDirectory
curl "https://breachdirectory.org/api?func=auto&term=example.com"

# LeakLookup
# https://leak-lookup.com

# Pastebin dorks (datos filtrados en pastes)
site:pastebin.com "example.com" "password"
site:pastebin.com "@example.com"
site:github.com "example.com" "password" OR "api_key" OR "secret"
```

---

## 10. Google Dorks aplicados a OSINT

Los Google Dorks son el complemento perfecto para encontrar información que el objetivo no pretende publicar.

### Dorks de reconocimiento corporativo

```
# Documentos internos filtrados
site:example.com filetype:pdf "internal" OR "confidential" OR "restricted"
site:example.com filetype:xls "employees" OR "staff" OR "payroll"

# Paneles de administración
site:example.com intitle:"admin" OR intitle:"login" OR intitle:"dashboard"
site:example.com inurl:admin OR inurl:wp-admin OR inurl:cpanel

# Subdominios y servicios
site:*.example.com -site:www.example.com

# Archivos de configuración expuestos
site:example.com ext:env OR ext:cfg OR ext:conf OR ext:ini
site:example.com ext:sql OR ext:bak OR ext:backup

# Directorios con listado habilitado
site:example.com intitle:"index of"

# Credenciales en GitHub (CRÍTICO)
site:github.com "example.com" password
site:github.com "example.com" api_key
site:github.com "example.com" "private_key"
site:github.com "@example.com" password

# Logs y dumps
site:example.com ext:log
site:example.com "error" filetype:txt

# Cámaras y dispositivos
inurl:"ViewerFrame?Mode="
inurl:"/view/view.shtml"
```

### Automatizar con dorks

```python
# googlesearch-python para automatizar
pip3 install googlesearch-python

from googlesearch import search

dorks = [
    'site:example.com filetype:pdf "confidential"',
    'site:example.com inurl:admin',
    'site:github.com "example.com" password',
    'site:example.com ext:env',
]

for dork in dorks:
    print(f"\n[+] Dork: {dork}")
    for result in search(dork, num_results=10, sleep_interval=2):
        print(f"    {result}")
```

---

## 11. Herramientas todo-en-uno

### Recon-ng

Framework modular de reconocimiento con decenas de módulos especializados.

```bash
# Instalar
pip3 install recon-ng

# Iniciar
recon-ng

# Comandos básicos
[recon-ng] > workspaces create example_corp
[recon-ng] > db insert domains
[domain] > example.com

# Cargar módulo
[recon-ng] > modules load recon/domains-hosts/hackertarget
[recon-ng][hackertarget] > options set SOURCE example.com
[recon-ng][hackertarget] > run

# Módulos útiles
recon/domains-hosts/hackertarget       # Subdominios
recon/domains-contacts/whois_pocs      # Contactos WHOIS
recon/domains-hosts/bing_domain_web    # Subdominios via Bing
recon/hosts-hosts/shodan_hostname      # Info Shodan
recon/contacts-credentials/hibp        # Have I Been Pwned
recon/companies-contacts/linkedin_auth # LinkedIn

# Generar reporte HTML
[recon-ng] > reporting load reporting/html
[recon-ng][html] > options set FILENAME /tmp/report.html
[recon-ng][html] > run
```

### Maltego

Herramienta de visualización de relaciones entre entidades. La versión Community es gratuita.

```
Entidades que maneja Maltego:
- Persona → Email, Teléfono, Alias, Organización
- Dominio → DNS, WHOIS, Subdominios, Certificados
- IP → Geolocalización, ASN, Hostnames
- Organización → Empleados, Dominios, Filiales
- Alias → Plataformas, Redes sociales
```

**Transforms útiles en Maltego:**
```
Domain → DNS Name (subdominios)
Domain → Whois Person (contactos WHOIS)
Email → Person (persona detrás del email)
Email → Have I Been Pwned (brechas)
IP → Shodan (servicios expuestos)
Person → LinkedIn (perfil profesional)
Person → Social Networks (presencia en redes)
```

### SpiderFoot

Automatiza OSINT combinando más de 200 fuentes.

```bash
# Instalar
pip3 install spiderfoot
# O Docker
docker pull smicallef/spiderfoot

# Iniciar el servidor web
python3 sf.py -l 127.0.0.1:5001

# Abrir en navegador
http://127.0.0.1:5001

# Módulos disponibles:
# - sfp_shodan, sfp_censys, sfp_virustotal
# - sfp_haveibeenpwned, sfp_dehashed
# - sfp_linkedin, sfp_twitter
# - sfp_whois, sfp_dns, sfp_ssl
# - sfp_pastebin, sfp_darksearch (dark web)
```

---

## 12. OSINT en redes sociales

### Twitter / X OSINT

```bash
# Twint — scraping sin API (puede estar deprecado, usar alternativas)
pip3 install twint
twint -u @username --email --phone

# API oficial de Twitter (Tier gratuito limitado)
# Bearer token desde developer.twitter.com

# Búsquedas avanzadas en Twitter
from:@CEO_username since:2024-01-01 until:2024-12-31
"example corp" lang:es -RT
to:@company (filtrar respuestas)
```

### Instagram OSINT

```bash
# Osintgram
git clone https://github.com/Datalux/Osintgram
cd Osintgram
pip3 install -r requirements.txt
python3 main.py <username>

# Comandos disponibles:
# addrs     → Direcciones geolocalizadas
# captions  → Texto de todas las fotos
# hashtags  → Hashtags utilizados
# fwingsnot → Seguidores que no siguen de vuelta
# location  → Ubicaciones visitadas
```

### Búsqueda de usernames entre plataformas

```bash
# Sherlock — busca un username en 300+ sitios
pip3 install sherlock-project
sherlock username_objetivo

# Maigret — más completo que Sherlock
pip3 install maigret
maigret username_objetivo --all-sites

# WhatsMyName
git clone https://github.com/WebBreacher/WhatsMyName
python3 whats_my_name.py -u username_objetivo
```

---

## 13. Automatización y scripts de reconocimiento

### Script de reconocimiento completo

```bash
#!/bin/bash
# recon.sh — Reconocimiento pasivo automatizado

TARGET=$1
OUTPUT_DIR="recon_${TARGET}"

if [ -z "$TARGET" ]; then
    echo "Uso: ./recon.sh ejemplo.com"
    exit 1
fi

mkdir -p "$OUTPUT_DIR"/{subdomains,emails,whois,shodan,metadata,wayback}

echo "[*] Iniciando reconocimiento de: $TARGET"
echo "[*] Output: $OUTPUT_DIR"

# 1. WHOIS
echo "[+] WHOIS..."
whois $TARGET > "$OUTPUT_DIR/whois/whois.txt" 2>/dev/null

# 2. Subdominios con subfinder
echo "[+] Subdominios con subfinder..."
subfinder -d $TARGET -silent -o "$OUTPUT_DIR/subdomains/subfinder.txt" 2>/dev/null

# 3. Subdominios con amass (pasivo)
echo "[+] Subdominios con amass..."
amass enum -passive -d $TARGET -o "$OUTPUT_DIR/subdomains/amass.txt" 2>/dev/null

# 4. Certificate Transparency
echo "[+] CT logs (crt.sh)..."
curl -s "https://crt.sh/?q=%.${TARGET}&output=json" 2>/dev/null | \
    jq -r '.[].name_value' | sort -u > "$OUTPUT_DIR/subdomains/crtsh.txt"

# 5. Combinar y deduplicar subdominios
cat "$OUTPUT_DIR/subdomains/"*.txt | sort -u > "$OUTPUT_DIR/subdomains/all_subdomains.txt"
TOTAL=$(wc -l < "$OUTPUT_DIR/subdomains/all_subdomains.txt")
echo "    Total subdominios: $TOTAL"

# 6. theHarvester
echo "[+] Recolectando emails con theHarvester..."
theHarvester -d $TARGET -b google,bing,duckduckgo -l 300 \
    -f "$OUTPUT_DIR/emails/harvester" 2>/dev/null

# 7. Wayback Machine URLs
echo "[+] URLs históricas de Wayback Machine..."
curl -s "http://web.archive.org/cdx/search/cdx?url=${TARGET}/*&output=text&fl=original&collapse=urlkey" \
    2>/dev/null | sort -u > "$OUTPUT_DIR/wayback/wayback_urls.txt"
WAYBACK=$(wc -l < "$OUTPUT_DIR/wayback/wayback_urls.txt")
echo "    URLs encontradas: $WAYBACK"

# 8. Resumen
echo ""
echo "══════════════════════════════════"
echo "  RECONOCIMIENTO COMPLETADO"
echo "══════════════════════════════════"
echo "  Directorio: $OUTPUT_DIR"
echo "  Subdominios: $(wc -l < $OUTPUT_DIR/subdomains/all_subdomains.txt)"
echo "  URLs históricas: $WAYBACK"
echo "══════════════════════════════════"
```

```bash
# Uso
chmod +x recon.sh
./recon.sh ejemplo.com
```

---

## 14. Documentación y reporte

Un reconocimiento sin documentación no vale nada. En un pentest profesional, el reporte de reconocimiento debe incluir:

### Estructura del reporte de reconocimiento

```markdown
# Reporte de Reconocimiento — [EMPRESA]
Fecha: DD/MM/AAAA
Analista: [Nombre]
Scope autorizado: [dominios/IPs]

## Resumen Ejecutivo
- X subdominios descubiertos
- Y emails identificados
- Z empleados enumerados
- W servicios expuestos con riesgo alto

## Infraestructura
### Dominios y subdominios
[tabla con subdominio, IP, servicio, estado]

### Servicios expuestos
[tabla con servicio, versión, riesgo, evidencia]

## Capital humano
### Empleados identificados
[tabla con nombre, cargo, email, fuente]

### Patrón de emails
Formato identificado: firstname.lastname@company.com

## Credenciales en brechas
[tabla con email, breach, año, datos expuestos]

## Tecnologías identificadas
[tabla con tecnología, versión, inferencia]

## Vectores de ataque potenciales
1. Phishing dirigido a [lista de emails]
2. Explotación de servicio X en versión Y (CVE-XXXX)
3. Credenciales filtradas en breachX

## Evidencias
[capturas, archivos, referencias]
```

### Herramientas de documentación recomendadas

```
CherryTree    → Notas jerárquicas, ideal para CTFs y pentests
Obsidian      → Markdown con links entre notas, mejor para engagements largos
Notion        → Colaborativo, bueno para equipos
Maltego       → Visualización de relaciones
draw.io       → Diagramas de red y relaciones
```

---

## 15. Cheatsheet de referencia rápida

```bash
# ── DOMINIOS ────────────────────────────────────────────
whois ejemplo.com                                     # WHOIS
subfinder -d ejemplo.com -o subs.txt                  # Subdominios
amass enum -passive -d ejemplo.com                    # Subdominios pasivo
curl -s "https://crt.sh/?q=%.ejemplo.com&output=json" | jq '.[].name_value' | sort -u

# ── EMAILS ──────────────────────────────────────────────
theHarvester -d ejemplo.com -b all -l 500             # Recolección masiva
# https://hunter.io                                   # Emails corporativos
# https://haveibeenpwned.com                          # Check brechas

# ── SHODAN ──────────────────────────────────────────────
shodan search 'org:"Ejemplo Corp"'                    # Búsqueda por org
shodan host 93.184.216.34                             # Info de IP
shodan search 'ssl.cert.subject.CN:ejemplo.com'       # Por certificado

# ── METADATOS ───────────────────────────────────────────
exiftool documento.pdf                                # Metadatos
exiftool -csv *.pdf > metadatos.csv                  # Batch CSV
exiftool -GPSLatitude -GPSLongitude foto.jpg         # GPS

# ── WAYBACK ─────────────────────────────────────────────
echo "ejemplo.com" | waybackurls > wayback.txt        # URLs históricas
gau ejemplo.com > all_urls.txt                        # Get All URLs

# ── USERNAMES ───────────────────────────────────────────
sherlock username                                     # Multi-plataforma
maigret username --all-sites                          # Más completo

# ── FRAMEWORK ───────────────────────────────────────────
# https://osintframework.com  → mapa de herramientas
# https://crt.sh              → CT logs
# https://shodan.io           → dispositivos expuestos
# https://censys.io           → alternativa a Shodan
# https://dehashed.com        → brechas de datos
# https://intelx.io           → paste sites + dark web
# https://hunter.io           → emails corporativos
# https://bgp.he.net          → ASN y rangos IP
```
