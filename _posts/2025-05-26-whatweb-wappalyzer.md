---
layout: post
title: "WhatWeb y Wappalyzer — Fingerprinting de Tecnologías Web"
date: 2025-05-26
categories: [enumeración]
tags: [whatweb, wappalyzer, fingerprinting, tecnologías, web, enumeración, recon, cms, frameworks]
description: "Guía profesional de WhatWeb y Wappalyzer: identificación del stack tecnológico de aplicaciones web, versiones de software, CMS, frameworks, servidores y técnicas de fingerprinting para pentesting."
---

## Por qué el fingerprinting tecnológico es el primer paso del pentesting web

Antes de buscar vulnerabilidades en una aplicación web, necesitas saber exactamente con qué estás tratando. Un servidor Apache 2.4.49 tiene el CVE-2021-41773. Un WordPress 5.8.1 tiene docenas de vulnerabilidades conocidas. Un jQuery 1.11.1 es vulnerable a XSS. Un servidor con PHP 5.6 lleva años sin recibir parches de seguridad. Sin identificar las tecnologías y sus versiones, estás probando a ciegas.

El fingerprinting tecnológico es el proceso de identificar qué software usa una aplicación web — el servidor, el CMS, los frameworks, las librerías JavaScript, el sistema de caché, el CDN, las herramientas de analítica, el proveedor de email — y cruzar esos datos con bases de datos de vulnerabilidades conocidas. Lo que para el desarrollador es simplemente el stack tecnológico de su aplicación, para el pentester es un mapa de posibles vectores de ataque.

WhatWeb y Wappalyzer son las dos herramientas más usadas para esta tarea, y se complementan perfectamente. WhatWeb es una herramienta de línea de comandos que hace fingerprinting profundo y agresivo, ideal para automatización. Wappalyzer es una extensión de navegador que hace fingerprinting pasivo mientras navegas, ideal para reconocimiento discreto y para identificar tecnologías que solo se cargan en rutas específicas.

---

## 1. Qué revela el fingerprinting tecnológico

Antes de entrar en herramientas, conviene entender exactamente qué información se puede extraer de un sitio web y de dónde viene:

### Fuentes de información tecnológica

```
Cabeceras HTTP de respuesta:
├── Server: Apache/2.4.49 (Ubuntu)          → Servidor web y versión
├── X-Powered-By: PHP/7.4.3                 → Lenguaje y versión
├── X-Generator: Drupal 9                   → CMS
├── X-Frame-Options, X-XSS-Protection       → Configuración de seguridad
├── Set-Cookie: PHPSESSID=...               → PHP; ASP.NET_SessionId → IIS
├── Set-Cookie: wp-settings-...             → WordPress
└── Via: 1.1 cloudflare                     → CDN

Contenido HTML:
├── <meta name="generator" content="WordPress 5.8.1">
├── <link rel="stylesheet" href="/wp-content/themes/...">
├── <script src="/wp-includes/js/jquery/jquery.min.js?ver=5.8.1">
├── <!--[if IE]> → soporte Internet Explorer
└── Comentarios HTML del desarrollador con rutas internas

Ficheros específicos:
├── /robots.txt       → rutas del CMS y estructura del sitio
├── /sitemap.xml      → todas las URLs indexadas
├── /wp-login.php     → confirma WordPress
├── /administrator/   → confirma Joomla
├── /CHANGELOG.txt    → versión exacta de Drupal
└── /.well-known/     → servicios y proveedores configurados

JavaScript:
├── Nombres de variables y funciones específicas de frameworks
├── Comentarios con versiones
├── Source maps que revelan el stack de build
└── Endpoints de API hardcodeados

Cookies:
├── PHPSESSID    → PHP
├── ASP.NET_SessionId → ASP.NET / IIS
├── JSESSIONID   → Java / Tomcat
├── laravel_session → Laravel
└── _rails_*     → Ruby on Rails

Estructura de URLs:
├── /index.php?option=com_content  → Joomla
├── /wp-content/uploads/           → WordPress
├── /Umbraco/                      → Umbraco CMS
└── /typo3/                        → TYPO3 CMS
```

---

## 2. WhatWeb — fingerprinting desde la línea de comandos

WhatWeb es una herramienta Ruby que identifica tecnologías web usando más de 1.800 plugins. Cada plugin busca patrones específicos en las respuestas HTTP: cabeceras, cookies, HTML, JavaScript, URLs. Lo que la hace especialmente potente para pentesting es su modo de agresividad configurable: desde consultas completamente pasivas hasta escaneos activos que prueban rutas adicionales.

### Instalación

```bash
# En Kali Linux — viene preinstalado
whatweb --version

# Instalar desde el repositorio oficial
git clone https://github.com/urbanadventurer/WhatWeb
cd WhatWeb
sudo gem install bundler
bundle install

# Con apt en sistemas Debian/Ubuntu
sudo apt update && sudo apt install whatweb

# Verificar instalación y ver número de plugins
whatweb --list-plugins | wc -l
```

### Uso básico

```bash
# Fingerprinting básico de un dominio
whatweb http://example.com

# El output tiene este formato:
# http://example.com [200 OK]
# Apache[2.4.49], Country[UNITED STATES][US],
# HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.49 (Ubuntu)],
# IP[93.184.216.34], JQuery[3.6.0], PHP[7.4.3],
# Script, Title[Example Domain], X-Powered-By[PHP/7.4.3]

# HTTPS
whatweb https://example.com

# Ignorar errores de certificado SSL
whatweb --no-check-certificate https://example.com

# Múltiples objetivos
whatweb http://example.com http://example.net

# Desde un fichero con URLs (una por línea)
whatweb --input-file urls.txt

# Guardar resultados
whatweb http://example.com -o resultados.txt        # Formato texto
whatweb http://example.com --log-json resultados.json  # JSON
whatweb http://example.com --log-xml resultados.xml    # XML
whatweb http://example.com --log-brief resultados.txt  # Formato compacto
```

### Niveles de agresividad — el parámetro más importante

WhatWeb tiene cinco niveles de agresividad que determinan cuántas peticiones hace y qué tan intrusivo es el fingerprinting:

```bash
# Nivel 1 (por defecto) — una sola petición HTTP
# Solo analiza la respuesta de la URL dada
# Completamente sigiloso — no genera tráfico adicional
whatweb -a 1 http://example.com

# Nivel 2 — peticiones pasivas adicionales
# Sigue redirecciones, analiza recursos enlazados
# Muy poco intrusivo
whatweb -a 2 http://example.com

# Nivel 3 — peticiones moderadamente agresivas
# Prueba rutas comunes del CMS detectado
# Puede dejar rastros en los logs del servidor
whatweb -a 3 http://example.com

# Nivel 4 — agresivo
# Prueba muchas más rutas y variaciones
# Genera tráfico notable
whatweb -a 4 http://example.com

# Nivel 5 — máximo
# Prueba todas las rutas posibles de todos los CMS conocidos
# Muy ruidoso — definitivamente visible en los logs
whatweb -a 5 http://example.com

# En reconocimiento pasivo: usar nivel 1 o 2
# En pentesting activo autorizado: nivel 3 o 4 para más información
```

### Escaneo de redes completas

Una de las funcionalidades más potentes de WhatWeb es la capacidad de escanear rangos de red completos, identificando qué tecnologías usa cada host web:

```bash
# Escanear una subred completa buscando servidores web
whatweb 10.10.10.0/24

# Escanear más rápido con múltiples threads
whatweb --threads 20 10.10.10.0/24

# Guardar resultados en JSON para procesamiento
whatweb --threads 20 10.10.10.0/24 --log-json red_completa.json

# Solo mostrar hosts con tecnologías específicas detectadas
whatweb 10.10.10.0/24 | grep -i "wordpress\|joomla\|drupal\|apache\|nginx"

# Combinar con nmap para escanear solo los hosts con puerto 80/443 abierto
nmap -p 80,443 --open 10.10.10.0/24 -oG - \
    | grep "80/open\|443/open" \
    | awk '{print $2}' \
    | whatweb --input-file - --threads 10 --log-json resultados.json
```

### Plugins de WhatWeb — entender qué detecta

WhatWeb usa plugins para cada tecnología. Puedes ver qué plugins están disponibles y qué detectan:

```bash
# Listar todos los plugins disponibles
whatweb --list-plugins

# Ver los detalles de un plugin específico
whatweb --info-plugin WordPress

# Buscar plugins relacionados con una tecnología
whatweb --list-plugins | grep -i "apache\|nginx\|cms"

# Crear un plugin personalizado para detectar tecnología propietaria
# Los plugins son ficheros Ruby en el directorio plugins/
cat > ~/.whatweb/my-plugin.rb << 'EOF'
Plugin.define "CustomApp" do
  author "Tu Nombre"
  version "0.1"
  description "Detecta la aplicación custom de Example Corp"

  matches [
    { :name => "x-powered-by header",
      :headers => { "X-Powered-By" => /CustomFramework/ } },
    { :name => "meta generator",
      :text => /<meta name="generator" content="CustomApp/ },
    { :name => "script path",
      :text => /\/customapp\/static\/js\// }
  ]
end
EOF
```

---

## 3. Wappalyzer — fingerprinting visual en el navegador

Wappalyzer es una extensión de navegador (Chrome, Firefox, Edge) que analiza en tiempo real las páginas que visitas e identifica las tecnologías que usan. A diferencia de WhatWeb que hace peticiones desde la terminal, Wappalyzer analiza la página exactamente como la ve el navegador, incluyendo JavaScript que se ejecuta dinámicamente, recursos cargados de forma asíncrona, y contenido generado en cliente.

### Instalación

```bash
# Extensión de Chrome
# Buscar "Wappalyzer" en Chrome Web Store
# https://chrome.google.com/webstore/detail/wappalyzer/gppongmhjkpfnbhagpmjfkannfbllamg

# Extensión de Firefox
# https://addons.mozilla.org/en-US/firefox/addon/wappalyzer/

# CLI para automatización
npm install -g wappalyzer
# o
pip3 install python-Wappalyzer
```

### Uso de Wappalyzer CLI

```bash
# Análisis básico de una URL
wappalyzer https://example.com

# Output en JSON
wappalyzer https://example.com --pretty

# Guardar resultados
wappalyzer https://example.com > tecnologias.json

# Con Python
python3 << 'EOF'
from Wappalyzer import Wappalyzer, WebPage

wappalyzer = Wappalyzer.latest()
webpage = WebPage.new_from_url("https://example.com")
tecnologias = wappalyzer.analyze_with_versions_and_categories(webpage)

for tech, info in tecnologias.items():
    version = info.get("version", "")
    cats = [c for c in info.get("categories", [])]
    print(f"  {tech:<30} {version:<15} {', '.join(cats)}")
EOF
```

### La API de Wappalyzer

Wappalyzer tiene una API REST que permite analizar URLs programáticamente:

```python
import requests

def wappalyzer_api(url, api_key):
    """
    Analiza una URL con la API de Wappalyzer.
    Requiere API key de wappalyzer.com (plan gratuito disponible).
    """
    response = requests.get(
        "https://api.wappalyzer.com/v2/lookup/",
        params={"urls": url},
        headers={"x-api-key": api_key}
    )

    if response.status_code == 200:
        data = response.json()
        technologies = data[0].get("technologies", [])

        print(f"\n[+] Tecnologías en {url}:")
        for tech in technologies:
            name       = tech.get("name")
            version    = tech.get("version", "")
            categories = [c.get("name") for c in tech.get("categories", [])]
            print(f"  {name:<30} {version:<15} [{', '.join(categories)}]")

        return technologies
    return []
```

---

## 4. Técnicas manuales de fingerprinting

Las herramientas automáticas no siempre capturan todo. Estas técnicas manuales complementan el fingerprinting automatizado:

### Analizar cabeceras HTTP directamente

```bash
# Ver todas las cabeceras de respuesta
curl -sI https://example.com

# Las más reveladoras:
# Server: nginx/1.18.0 → servidor y versión
# X-Powered-By: PHP/7.4.3 → lenguaje
# X-Generator: Drupal 9 (https://www.drupal.org) → CMS
# Set-Cookie: PHPSESSID → PHP
# Set-Cookie: ASP.NET_SessionId → IIS + ASP.NET

# Ver cabeceras con verbose para ver también las de petición
curl -v https://example.com 2>&1 | grep -E "^[<>] "

# Detectar WAF por las cabeceras
curl -sI https://example.com | grep -iE "cloudflare|akamai|imperva|sucuri|wordfence|x-firewall"

# Cabeceras de seguridad — su ausencia revela configuración descuidada
curl -sI https://example.com | grep -iE "strict-transport|content-security|x-frame|x-content-type|referrer-policy"
```

### Ficheros de fingerprinting estándar

Ciertos ficheros son específicos de cada tecnología y confirman su presencia:

```bash
# WordPress
curl -s https://example.com/wp-login.php | grep -i "wordpress"
curl -s https://example.com/wp-json/wp/v2/ | python3 -m json.tool   # API REST
curl -s https://example.com/readme.html | grep -i "version"         # Versión exacta

# Joomla
curl -s https://example.com/administrator/ | grep -i "joomla"
curl -s https://example.com/language/en-GB/en-GB.xml | grep -i "version"

# Drupal
curl -s https://example.com/CHANGELOG.txt | head -5    # Versión exacta
curl -s https://example.com/core/CHANGELOG.txt | head  # Drupal 8+

# Django (Python)
curl -s https://example.com/admin/ | grep -i "django"
# El panel de admin de Django tiene un diseño muy reconocible

# Laravel (PHP)
curl -s https://example.com | grep -i "laravel\|_token"
# Las cookies de Laravel tienen formato específico: laravel_session

# Spring Boot (Java)
curl -s https://example.com/actuator/ | python3 -m json.tool
curl -s https://example.com/actuator/health
curl -s https://example.com/actuator/env    # Puede revelar variables de entorno

# Ruby on Rails
curl -s https://example.com | grep -i "rails\|turbolinks"
# Las cookies de Rails tienen formato específico: _session_id

# Verificar robots.txt — siempre relevante
curl -s https://example.com/robots.txt

# Verificar sitemap.xml — revela estructura completa
curl -s https://example.com/sitemap.xml | grep -oP 'https?://[^<]+'
```

### Identificar versiones exactas de JavaScript

Las versiones de librerías JavaScript pueden revelar vulnerabilidades conocidas:

```bash
# Descargar la página y buscar versiones en el HTML
curl -s https://example.com | grep -oiE "jquery[./\-]([0-9]+\.[0-9]+\.?[0-9]*)"
curl -s https://example.com | grep -oiE "angular[./\-]([0-9]+\.[0-9]+\.?[0-9]*)"
curl -s https://example.com | grep -oiE "react[./\-]([0-9]+\.[0-9]+\.?[0-9]*)"
curl -s https://example.com | grep -oiE "bootstrap[./\-]([0-9]+\.[0-9]+\.?[0-9]*)"

# Buscar en los source maps (revelan stack de build completo)
curl -s https://example.com/static/js/main.chunk.js.map 2>/dev/null | python3 -c "
import json, sys
data = json.load(sys.stdin)
sources = data.get('sources', [])
for s in sources:
    if 'node_modules' in s:
        print(s)
" | sort -u

# Verificar versiones de jQuery en fichero JS
curl -s https://example.com/wp-includes/js/jquery/jquery.min.js | head -3
# Primera línea suele contener la versión
```

---

## 5. Cruzar tecnologías con CVEs

El objetivo final del fingerprinting es identificar vulnerabilidades explotables. Este proceso de cruce es fundamental:

```python
#!/usr/bin/env python3
"""
tech_vuln_mapper.py — Cruza tecnologías detectadas con CVEs conocidos
Usa la NVD API para buscar vulnerabilidades por producto y versión
"""

import requests
import json
import sys

def search_nvd_cves(product, version=None):
    """
    Busca CVEs en la base de datos NVD para un producto y versión.
    La API NVD es gratuita pero tiene rate limiting.
    """
    params = {
        "keywordSearch": product,
        "resultsPerPage": 10
    }
    if version:
        params["keywordSearch"] = f"{product} {version}"

    try:
        response = requests.get(
            "https://services.nvd.nist.gov/rest/json/cves/2.0",
            params=params,
            timeout=10
        )
        if response.status_code == 200:
            data = response.json()
            vulns = []
            for item in data.get("vulnerabilities", []):
                cve = item.get("cve", {})
                cve_id = cve.get("id", "")
                desc = cve.get("descriptions", [{}])[0].get("value", "")
                metrics = cve.get("metrics", {})

                # Extraer CVSS score
                score = "N/A"
                if "cvssMetricV31" in metrics:
                    score = metrics["cvssMetricV31"][0]["cvssData"]["baseScore"]
                elif "cvssMetricV2" in metrics:
                    score = metrics["cvssMetricV2"][0]["cvssData"]["baseScore"]

                vulns.append({
                    "id": cve_id,
                    "score": score,
                    "description": desc[:150] + "..." if len(desc) > 150 else desc
                })

            return vulns
    except Exception as e:
        print(f"  Error consultando NVD: {e}")
    return []

def analyze_technologies(tech_list):
    """
    Para cada tecnología detectada, busca CVEs relevantes.
    tech_list: lista de dicts con 'name' y 'version'
    """
    print("\n" + "="*60)
    print("  ANÁLISIS DE VULNERABILIDADES POR TECNOLOGÍA")
    print("="*60)

    for tech in tech_list:
        name    = tech.get("name", "")
        version = tech.get("version", "")

        print(f"\n[*] {name} {version}")

        cves = search_nvd_cves(name, version)

        if cves:
            # Ordenar por CVSS score descendente
            cves_sorted = sorted(
                cves,
                key=lambda x: float(x["score"]) if x["score"] != "N/A" else 0,
                reverse=True
            )

            for cve in cves_sorted[:5]:  # Mostrar top 5
                score = cve["score"]
                # Colorear por severidad (simplificado)
                severity = "CRÍTICO" if float(score) >= 9.0 else \
                           "ALTO"    if float(score) >= 7.0 else \
                           "MEDIO"   if float(score) >= 4.0 else "BAJO" \
                           if score != "N/A" else "N/A"

                print(f"  [{severity}] {cve['id']} (CVSS: {score})")
                print(f"    {cve['description']}")
        else:
            print(f"  Sin CVEs encontrados en NVD")

# Ejemplo de uso con tecnologías detectadas por WhatWeb o Wappalyzer
technologies_detected = [
    {"name": "Apache", "version": "2.4.49"},
    {"name": "PHP", "version": "7.4.3"},
    {"name": "WordPress", "version": "5.8.1"},
    {"name": "jQuery", "version": "1.11.1"},
    {"name": "OpenSSL", "version": "1.0.2"},
]

analyze_technologies(technologies_detected)
```

---

## 6. Flujo completo de fingerprinting web

```bash
#!/bin/bash
# tech_fingerprint.sh — Fingerprinting tecnológico completo
# Uso: ./tech_fingerprint.sh https://example.com

TARGET=$1
OUTDIR="./fingerprint_$(echo $TARGET | sed 's|https\?://||;s|/|_|g')"
mkdir -p "$OUTDIR"

if [ -z "$TARGET" ]; then
    echo "Uso: $0 <URL>"
    exit 1
fi

echo "╔══════════════════════════════════════════╗"
echo "  Fingerprinting: $TARGET"
echo "╚══════════════════════════════════════════╝"

# ── FASE 1: CABECERAS HTTP ────────────────────────────────────────────
echo ""
echo "[1/5] Analizando cabeceras HTTP..."
curl -skI "$TARGET" > "$OUTDIR/01_headers.txt" 2>/dev/null
echo "  Cabeceras relevantes:"
grep -iE "server:|x-powered-by:|x-generator:|via:|set-cookie:|x-aspnet" \
    "$OUTDIR/01_headers.txt" | sed 's/^/  /'

# ── FASE 2: WHATWEB ───────────────────────────────────────────────────
echo ""
echo "[2/5] WhatWeb fingerprinting..."
whatweb -a 3 "$TARGET" --log-json "$OUTDIR/02_whatweb.json" 2>/dev/null
whatweb -a 3 "$TARGET" 2>/dev/null | tee "$OUTDIR/02_whatweb.txt"

# ── FASE 3: FICHEROS DE FINGERPRINTING ───────────────────────────────
echo ""
echo "[3/5] Verificando ficheros de fingerprinting..."

# Lista de rutas a verificar con su significado
declare -A FINGERPRINT_PATHS=(
    ["/robots.txt"]="Estructura del sitio"
    ["/sitemap.xml"]="URLs indexadas"
    ["/readme.html"]="WordPress versión"
    ["/wp-login.php"]="WordPress login"
    ["/wp-json/wp/v2/"]="WordPress API REST"
    ["/administrator/"]="Joomla admin"
    ["/CHANGELOG.txt"]="Drupal versión"
    ["/actuator/health"]="Spring Boot Actuator"
    ["/actuator/"]="Spring Boot Actuator (todos)"
    ["/.env"]="Variables de entorno expuestas"
    ["/config.php"]="Configuración PHP"
    ["/phpinfo.php"]="PHP Info expuesto"
    ["/server-status"]="Apache server-status"
    ["/server-info"]="Apache server-info"
)

echo "" > "$OUTDIR/03_fingerprint_paths.txt"
for path in "${!FINGERPRINT_PATHS[@]}"; do
    status=$(curl -sk -o /dev/null -w "%{http_code}" "${TARGET}${path}" 2>/dev/null)
    if [[ "$status" == "200" || "$status" == "301" || "$status" == "302" ]]; then
        desc="${FINGERPRINT_PATHS[$path]}"
        echo "  [HTTP $status] ${path} — $desc"
        echo "HTTP $status | ${TARGET}${path} | $desc" >> "$OUTDIR/03_fingerprint_paths.txt"
    fi
done

# ── FASE 4: BUSCAR VERSIONES JS ───────────────────────────────────────
echo ""
echo "[4/5] Detectando librerías JavaScript..."
curl -sk "$TARGET" 2>/dev/null | grep -oiE \
    "(jquery|angular|react|vue|bootstrap|lodash)[./\-]([0-9]+\.?){2,3}" \
    | sort -u | tee "$OUTDIR/04_js_versions.txt"

# ── FASE 5: RESUMEN Y RECOMENDACIONES ────────────────────────────────
echo ""
echo "[5/5] Generando resumen..."

echo ""
echo "╔══════════════════════════════════════════╗"
echo "  FINGERPRINTING COMPLETADO"
echo "╠══════════════════════════════════════════╣"
echo "  Resultados en: $OUTDIR/"
echo "  Revisar especialmente:"
echo "   → $OUTDIR/02_whatweb.json"
echo "   → $OUTDIR/03_fingerprint_paths.txt"
echo "╚══════════════════════════════════════════╝"

echo ""
echo "[!] Próximos pasos recomendados:"
echo "    1. Cruzar versiones detectadas con CVE database"
echo "    2. Buscar exploits en SearchSploit: searchsploit <tecnología> <versión>"
echo "    3. Si hay CMS → usar escáner específico (WPScan, CMSmap, Droopescan)"
echo "    4. Si hay Spring Actuator → verificar /actuator/env y /actuator/dump"
echo "    5. Si hay .env accesible → CRÍTICO — credenciales expuestas"
```

---

## 7. Cheatsheet de referencia rápida

```bash
# ── WHATWEB ───────────────────────────────────────────────────────────────
whatweb http://target                              # Básico (nivel 1)
whatweb -a 3 http://target                        # Moderadamente agresivo
whatweb -a 3 --threads 20 10.10.10.0/24           # Red completa
whatweb http://target --log-json out.json         # Output JSON
whatweb http://target --no-check-certificate      # Ignorar SSL
whatweb --input-file urls.txt --threads 10        # Múltiples URLs
whatweb --list-plugins | grep -i wordpress        # Buscar plugins

# ── CABECERAS HTTP MANUALES ───────────────────────────────────────────────
curl -sI https://target                           # Solo cabeceras
curl -sI https://target | grep -iE "server:|x-powered-by:|x-generator:"
curl -sv https://target 2>&1 | grep "^[<>]"       # Verbose con petición

# ── FICHEROS DE FINGERPRINTING ────────────────────────────────────────────
/robots.txt          → estructura del sitio
/wp-login.php        → confirma WordPress
/readme.html         → versión de WordPress
/wp-json/wp/v2/      → API REST de WordPress
/administrator/      → confirma Joomla
/CHANGELOG.txt       → versión de Drupal (versiones antiguas)
/.env                → variables de entorno (CRÍTICO si accesible)
/phpinfo.php         → PHP info completo (CRÍTICO)
/actuator/           → Spring Boot Actuator (puede exponer env, heap dump)
/server-status       → Apache server-status (IPs clientes, requests activos)

# ── DETECTAR JS VERSIONS ──────────────────────────────────────────────────
curl -s https://target | grep -oiE "jquery[./\-][0-9.]+"
curl -s https://target | grep -oiE "angular[./\-][0-9.]+"
curl -s https://target | grep -oiE "bootstrap[./\-][0-9.]+"

# ── WAPPALYZER CLI ────────────────────────────────────────────────────────
npm install -g wappalyzer
wappalyzer https://target --pretty

# ── PYTHON WAPPALYZER ─────────────────────────────────────────────────────
pip3 install python-Wappalyzer
python3 -c "
from Wappalyzer import Wappalyzer, WebPage
w = Wappalyzer.latest()
page = WebPage.new_from_url('https://target')
print(w.analyze_with_versions_and_categories(page))
"

# ── CRUZAR CON EXPLOITS ───────────────────────────────────────────────────
searchsploit apache 2.4.49           # Buscar exploits por versión
searchsploit wordpress 5.8           # WordPress
searchsploit jquery 1.11             # jQuery
searchsploit --json apache 2.4 | python3 -m json.tool

# ── ESCÁNERES ESPECÍFICOS DE CMS ──────────────────────────────────────────
wpscan --url http://target           # WordPress (post 19)
droopescan scan drupal -u http://target  # Drupal
joomscan -u http://target            # Joomla
cmseek -u http://target              # Detección automática de CMS

# ── RECURSOS ──────────────────────────────────────────────────────────────
https://github.com/urbanadventurer/WhatWeb    → WhatWeb
https://www.wappalyzer.com                   → Wappalyzer
https://nvd.nist.gov                         → NVD CVE database
https://www.exploit-db.com                   → Exploit Database
https://cve.mitre.org                        → MITRE CVE
```
