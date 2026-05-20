---
layout: post
title: "Wayback Machine y Caché — Encontrar Recursos Ocultos"
date: 2025-05-19
categories: [reconocimiento]
tags: [wayback-machine, cache, osint, reconocimiento, recon, urls, endpoints, google-cache, web-archive]
description: "Guía profesional de reconocimiento con Wayback Machine y caché web: encontrar endpoints eliminados, credenciales expuestas, infraestructura antigua y recursos ocultos en el historial de internet."
---

## El pasado de internet como fuente de inteligencia

Internet tiene memoria. Cuando una empresa elimina una página de su web, borra un endpoint de su API, o retira un panel de administración porque alguien se dio cuenta de que estaba expuesto, asume que ese recurso ha desaparecido para siempre. Esa suposición es errónea. Servicios como Wayback Machine, Google Cache, y docenas de herramientas especializadas archivan continuamente el contenido de internet, y ese historial puede contener información que la empresa creía eliminada hace años.

En reconocimiento profesional, el historial web es una de las fuentes más valiosas y menos explotadas. Hemos visto engagements donde el acceso inicial vino de credenciales encontradas en un archivo JavaScript que fue eliminado hace dos años pero que seguía vivo en Wayback Machine. O endpoints de API que fueron deprecados pero que el servidor seguía sirviendo porque nadie los había deshabilitado realmente — solo los habían quitado de la documentación. O paneles de administración que existieron durante unas semanas mientras migraban la aplicación y que quedaron archivados con sus parámetros de autenticación visibles.

La pregunta clave no es "¿qué tiene el objetivo hoy?" sino "¿qué tuvo el objetivo antes, y qué rastros dejó?". El historial responde esa pregunta.

---

## 1. Wayback Machine — El archivo de internet

Wayback Machine es el proyecto de archive.org que lleva archivando páginas web desde 1996. Actualmente almacena más de 800.000 millones de páginas y crece continuamente. Para el reconocimiento pasivo es una fuente de datos única: muestra cómo era un sitio en cualquier momento del pasado, qué URLs existían, qué código JavaScript se cargaba, qué formularios había, y qué estructuras de directorios estaban expuestas.

Lo que hace Wayback Machine especialmente valioso en pentesting no es ver cómo se veía la web visualmente, sino acceder a los **recursos técnicos** que existieron: archivos de configuración que estuvieron expuestos brevemente, endpoints de API que ya no están en producción pero que el servidor aún acepta, versiones antiguas de JavaScript con comentarios que revelan infraestructura, y páginas de error que mostraban rutas del sistema de archivos.

### Acceso manual a Wayback Machine

La interfaz web de archive.org permite navegar visualmente por el historial de cualquier URL:

```
# Formato de URL para acceder directamente a una captura
https://web.archive.org/web/[TIMESTAMP]/[URL]

# Ejemplos:
# Ver example.com tal como era el 1 de enero de 2020
https://web.archive.org/web/20200101000000*/example.com

# Ver el calendario de capturas de un dominio
https://web.archive.org/web/*/example.com

# Ver una URL específica en la fecha más cercana a una fecha dada
https://web.archive.org/web/20230615/https://example.com/admin/

# Buscar todas las capturas de un directorio
https://web.archive.org/web/*/https://example.com/api/*
```

### La API CDX de Wayback Machine

La verdadera potencia de Wayback Machine para reconocimiento está en su API CDX (Content Delivery Index), que permite hacer consultas programáticas sobre el índice de todas las URLs archivadas. Esta API es gratuita, no requiere autenticación, y devuelve resultados en texto plano o JSON.

La API CDX permite buscar no solo por dominio sino también por patrones de URL, filtrar por tipo de contenido, por código de respuesta HTTP, y ordenar los resultados. Para reconocimiento esto es extraordinariamente útil:

```bash
# Obtener TODAS las URLs archivadas de un dominio
# output=text devuelve una URL por línea, fl=original especifica el campo
# collapse=urlkey elimina duplicados de la misma URL
curl -s "http://web.archive.org/cdx/search/cdx?url=example.com/*\
&output=text\
&fl=original\
&collapse=urlkey" > todas_las_urls.txt

# Ver cuántas URLs únicas hay
wc -l todas_las_urls.txt

# Solo URLs que devolvieron HTTP 200 (páginas que existieron y respondieron)
curl -s "http://web.archive.org/cdx/search/cdx?url=example.com/*\
&output=text\
&fl=original\
&filter=statuscode:200\
&collapse=urlkey" > urls_200.txt

# Solo archivos JavaScript — pueden contener endpoints de API, credenciales hardcodeadas
curl -s "http://web.archive.org/cdx/search/cdx?url=example.com/*.js\
&output=text\
&fl=original\
&collapse=urlkey" > archivos_js.txt

# Solo archivos de configuración potencialmente expuestos
curl -s "http://web.archive.org/cdx/search/cdx?url=example.com/*\
&output=text\
&fl=original\
&filter=original:.*\.(env|conf|config|ini|cfg|xml|json|yaml|yml|bak|backup|old|sql)\
&collapse=urlkey" > archivos_config.txt

# Solo endpoints de API (URLs que contienen /api/)
curl -s "http://web.archive.org/cdx/search/cdx?url=example.com/api/*\
&output=text\
&fl=original\
&collapse=urlkey" > endpoints_api.txt

# Obtener también el timestamp y el código de respuesta para cada URL
curl -s "http://web.archive.org/cdx/search/cdx?url=example.com/*\
&output=text\
&fl=timestamp,statuscode,original\
&collapse=urlkey" > urls_con_metadata.txt
```

### Parámetros avanzados de la API CDX

La API CDX tiene muchos más parámetros que permiten afinar las búsquedas:

```bash
# Filtrar por tipo MIME — solo páginas HTML
curl -s "http://web.archive.org/cdx/search/cdx?url=example.com/*\
&output=text\
&fl=original\
&filter=mimetype:text/html\
&collapse=urlkey"

# Filtrar por tipo MIME — solo JSON (endpoints de API)
curl -s "http://web.archive.org/cdx/search/cdx?url=example.com/*\
&output=text\
&fl=original\
&filter=mimetype:application/json\
&collapse=urlkey"

# Obtener la primera y última captura de cada URL (sin duplicados intermedios)
# Útil para ver cuándo apareció y desapareció un recurso
curl -s "http://web.archive.org/cdx/search/cdx?url=example.com/*\
&output=json\
&fl=timestamp,original,statuscode\
&from=20200101\
&to=20241231\
&collapse=urlkey"

# Buscar en subdominios también
curl -s "http://web.archive.org/cdx/search/cdx?url=*.example.com/*\
&output=text\
&fl=original\
&collapse=urlkey" > urls_todos_subdominios.txt

# Limitar el número de resultados (útil para dominios muy grandes)
curl -s "http://web.archive.org/cdx/search/cdx?url=example.com/*\
&output=text\
&fl=original\
&collapse=urlkey\
&limit=10000" > primeras_10000_urls.txt
```

---

## 2. Herramientas especializadas para reconocimiento histórico

Aunque la API CDX es muy potente, hay herramientas que la envuelven con funcionalidades adicionales específicas para reconocimiento de seguridad.

### waybackurls — El clásico de Jason Haddix

waybackurls es una herramienta Go creada por el investigador de bug bounty tomnomnom que simplifica la consulta de Wayback Machine para obtener todas las URLs históricas de un dominio. Es más rápida que curl para dominios grandes porque hace múltiples requests en paralelo.

```bash
# Instalar waybackurls
go install github.com/tomnomnom/waybackurls@latest

# Uso básico: obtener todas las URLs históricas de un dominio
echo "example.com" | waybackurls > wayback_urls.txt

# Múltiples dominios desde un archivo
cat dominios.txt | waybackurls > wayback_todos.txt

# Ver el progreso mientras se ejecuta
echo "example.com" | waybackurls | tee wayback_urls.txt | wc -l

# Filtrar resultados interesantes inmediatamente
echo "example.com" | waybackurls | grep -E "\.(js|json|xml|env|sql|bak)$"
```

### gau — Get All URLs

gau (Get All URLs) es otra herramienta de tomnomnom que combina múltiples fuentes: Wayback Machine, Common Crawl, y URLScan.io. Al combinar más de una fuente, encuentra URLs que Wayback Machine no tiene indexadas.

```bash
# Instalar gau
go install github.com/lc/gau/v2/cmd/gau@latest

# Uso básico
gau example.com > gau_urls.txt

# Solo fuentes específicas
gau --subs example.com > gau_con_subdominios.txt

# Con threads para ir más rápido
gau --threads 5 example.com > gau_urls.txt

# Combinar gau y waybackurls para máxima cobertura
cat <(echo "example.com" | waybackurls) \
    <(gau example.com) \
    | sort -u > todas_las_urls_historicas.txt
```

### hakrawler — Combinando archivo y crawling activo

hakrawler combina el reconocimiento pasivo (Wayback Machine) con un crawler web activo ligero. Útil cuando quieres enriquecer el historial con URLs nuevas que el objetivo ha añadido recientemente.

```bash
# Instalar hakrawler
go install github.com/hakluke/hakrawler@latest

# Crawl básico con profundidad 2
echo "https://example.com" | hakrawler -depth 2 > hakrawler_urls.txt

# Solo URLs del subdominio actual (sin seguir enlaces externos)
echo "https://example.com" | hakrawler -subs -depth 3 > urls_subs.txt

# Combinar con datos históricos de waybackurls
{
  echo "example.com" | waybackurls
  echo "https://example.com" | hakrawler -depth 2
} | sort -u > urls_completas.txt
```

---

## 3. Filtrado inteligente de URLs para encontrar recursos valiosos

Tener 50.000 URLs históricas no sirve de nada si no sabes dónde buscar lo importante. El proceso de filtrado es donde se extrae el valor real del reconocimiento histórico.

### Filtrar por extensión de archivo

```bash
# Archivos JavaScript — fuente de endpoints, API keys, comentarios de desarrollo
grep -E "\.js(\?|$)" todas_las_urls.txt | sort -u > urls_js.txt

# Archivos de configuración que no deberían ser públicos
grep -iE "\.(env|conf|config|ini|cfg|htaccess|htpasswd)(\?|$)" \
    todas_las_urls.txt | sort -u > urls_config.txt

# Archivos de base de datos y backups
grep -iE "\.(sql|db|sqlite|bak|backup|dump|tar|zip|gz|rar|7z)(\?|$)" \
    todas_las_urls.txt | sort -u > urls_backups.txt

# Archivos de log
grep -iE "\.(log|txt)(\?|$)" todas_las_urls.txt | sort -u > urls_logs.txt

# Archivos XML y JSON — APIs, configuraciones, sitemaps
grep -iE "\.(xml|json|yaml|yml)(\?|$)" todas_las_urls.txt | sort -u > urls_xml_json.txt

# Archivos PHP que pueden tener vulnerabilidades
grep -iE "\.php(\?|$)" todas_las_urls.txt | sort -u > urls_php.txt
```

### Filtrar por palabras clave en la URL

```bash
# Paneles de administración que existieron en el pasado
grep -iE "/(admin|administrator|panel|backend|dashboard|console|manage)" \
    todas_las_urls.txt | sort -u > urls_admin.txt

# Endpoints de API
grep -iE "/api/|/v1/|/v2/|/v3/|/graphql|/rest/" \
    todas_las_urls.txt | sort -u > urls_api.txt

# Rutas de autenticación y usuarios
grep -iE "/(login|logout|signin|signup|register|auth|oauth|token|password|reset)" \
    todas_las_urls.txt | sort -u > urls_auth.txt

# Rutas de carga de archivos — potenciales vulnerabilidades de upload
grep -iE "/(upload|file|import|attach|media)" \
    todas_las_urls.txt | sort -u > urls_upload.txt

# Rutas de desarrollo y testing
grep -iE "/(dev|test|staging|debug|trace|phpinfo|info\.php|server-info)" \
    todas_las_urls.txt | sort -u > urls_dev.txt

# Rutas de backup y versiones antiguas
grep -iE "/(backup|old|bak|temp|tmp|archive|legacy)" \
    todas_las_urls.txt | sort -u > urls_legacy.txt

# Parámetros GET que sugieren inyecciones o SSRF
grep -iE "[?&](id|user|file|path|url|redirect|next|return|callback|host)=" \
    todas_las_urls.txt | sort -u > urls_params_interesantes.txt
```

### Filtrar por parámetros de URL

Los parámetros GET en las URLs históricas son especialmente interesantes porque pueden revelar vectores de inyección, rutas de SSRF, o comportamientos de la aplicación que ya no son evidentes en la versión actual:

```bash
# Parámetros que sugieren SQL Injection
grep -iE "[?&](id|item|product|cat|category|page|num|order|sort)=\d" \
    todas_las_urls.txt | sort -u > params_sqli_candidatos.txt

# Parámetros que sugieren LFI/Path Traversal
grep -iE "[?&](file|page|include|path|dir|folder|document|load)=" \
    todas_las_urls.txt | sort -u > params_lfi_candidatos.txt

# Parámetros que sugieren SSRF
grep -iE "[?&](url|uri|link|src|source|dest|destination|host|proxy|callback)=" \
    todas_las_urls.txt | sort -u > params_ssrf_candidatos.txt

# Parámetros con valores de redirección
grep -iE "[?&](redirect|return|next|goto|target|redir|forward)=" \
    todas_las_urls.txt | sort -u > params_redirect.txt
```

---

## 4. Analizar el contenido histórico

Encontrar URLs es solo el primer paso. Lo más valioso es acceder al contenido de esas páginas tal como estaba en el pasado.

### Recuperar el contenido de una URL archivada

```bash
# Obtener el contenido más reciente de una URL archivada
curl -s "https://web.archive.org/web/2024*/https://example.com/api/config.json"

# Obtener la versión más antigua disponible
curl -s "https://web.archive.org/web/19960101000000*/https://example.com/"

# Script para descargar el contenido de múltiples URLs archivadas
#!/bin/bash
while IFS= read -r url; do
    # Construir la URL de Wayback Machine
    wayback_url="https://web.archive.org/web/2023*/${url}"
    # Descargar y guardar con nombre basado en la URL
    filename=$(echo "$url" | md5sum | cut -d' ' -f1)
    curl -s --max-time 30 "$wayback_url" -o "./archived_content/${filename}.html"
    sleep 0.5  # Ser respetuoso con el servidor de archive.org
done < urls_interesantes.txt
```

### Buscar secretos en JavaScript histórico

Los archivos JavaScript son especialmente valiosos porque los desarrolladores frecuentemente hardcodean credenciales, API keys, y endpoints durante el desarrollo y olvidan eliminarlos antes de subir a producción. La versión archivada de un JS puede tener secretos que la versión actual ya no tiene:

```python
#!/usr/bin/env python3
"""
js_secret_hunter.py — Busca secretos en archivos JavaScript históricos
Descarga cada JS de Wayback Machine y busca patrones de credenciales
"""

import re
import requests
import sys
from urllib.parse import urljoin

# Patrones que indican posibles secretos o información sensible
SECRET_PATTERNS = [
    # API Keys y tokens
    (r'api[_\-]?key["\s]*[:=]["\s]*(["\']?)([A-Za-z0-9_\-]{20,})\1',
     "API Key"),
    (r'api[_\-]?secret["\s]*[:=]["\s]*(["\']?)([A-Za-z0-9_\-]{20,})\1',
     "API Secret"),
    (r'access[_\-]?token["\s]*[:=]["\s]*(["\']?)([A-Za-z0-9_\-]{20,})\1',
     "Access Token"),
    (r'auth[_\-]?token["\s]*[:=]["\s]*(["\']?)([A-Za-z0-9_\-]{20,})\1',
     "Auth Token"),

    # Credenciales de base de datos
    (r'password["\s]*[:=]["\s]*(["\'])([^"\']{6,})\1',
     "Password"),
    (r'passwd["\s]*[:=]["\s]*(["\'])([^"\']{6,})\1',
     "Password"),
    (r'db[_\-]?pass["\s]*[:=]["\s]*(["\'])([^"\']{6,})\1',
     "DB Password"),

    # AWS
    (r'AKIA[0-9A-Z]{16}',
     "AWS Access Key ID"),
    (r'aws[_\-]?secret["\s]*[:=]["\s]*(["\'])([A-Za-z0-9/+]{40})\1',
     "AWS Secret Key"),

    # Endpoints internos
    (r'https?://(?:localhost|127\.0\.0\.1|10\.\d+\.\d+\.\d+|192\.168\.\d+\.\d+)[^\s"\']+',
     "Internal Endpoint"),
    (r'https?://[a-zA-Z0-9\-]+\.internal[^\s"\']*',
     "Internal Domain"),

    # Comentarios de desarrollo con información sensible
    (r'//\s*TODO.*(?:password|credential|secret|key|token)[^\n]*',
     "TODO Comment with credentials"),
    (r'/\*.*(?:password|credential|secret|key|token).*\*/',
     "Comment with credentials"),
]

def get_archived_js_content(js_url):
    """
    Descarga el contenido de un archivo JS desde Wayback Machine.
    Primero obtiene la lista de capturas disponibles, luego descarga la más reciente.
    """
    # Consultar la API CDX para ver qué capturas hay disponibles
    cdx_url = f"http://web.archive.org/cdx/search/cdx?url={js_url}&output=json&fl=timestamp&limit=5&filter=statuscode:200"

    try:
        response = requests.get(cdx_url, timeout=15)
        captures = response.json()

        if len(captures) < 2:  # La primera fila es el header
            return None

        # Tomar la captura más reciente
        latest_timestamp = captures[-1][0]
        archived_url = f"https://web.archive.org/web/{latest_timestamp}id_/{js_url}"

        # Descargar el contenido del JS archivado
        # id_ en la URL de Wayback Machine devuelve el contenido original sin modificar
        js_response = requests.get(archived_url, timeout=30)
        if js_response.status_code == 200:
            return js_response.text

    except Exception as e:
        print(f"  Error descargando {js_url}: {e}")

    return None

def search_secrets(content, source_url):
    """
    Busca patrones de secretos en el contenido del JS.
    Devuelve lista de hallazgos con el tipo y valor encontrado.
    """
    findings = []

    for pattern, secret_type in SECRET_PATTERNS:
        matches = re.finditer(pattern, content, re.IGNORECASE)
        for match in matches:
            # El valor del secreto puede estar en diferentes grupos según el patrón
            value = match.group(0)
            if len(match.groups()) > 1:
                value = match.group(2) if match.group(2) else match.group(0)

            findings.append({
                "type": secret_type,
                "value": value[:100],  # Limitar longitud para no mostrar secretos completos en logs
                "context": content[max(0, match.start()-50):match.end()+50].strip(),
                "url": source_url
            })

    return findings

def analyze_js_files(js_urls_file):
    """
    Procesa una lista de URLs de JavaScript y busca secretos en cada una.
    """
    all_findings = []

    with open(js_urls_file) as f:
        urls = [line.strip() for line in f if line.strip()]

    print(f"[*] Analizando {len(urls)} archivos JavaScript históricos...")

    for i, url in enumerate(urls, 1):
        print(f"  [{i}/{len(urls)}] {url}")

        content = get_archived_js_content(url)
        if not content:
            print(f"    [-] No disponible en Wayback Machine")
            continue

        findings = search_secrets(content, url)

        if findings:
            print(f"    [!] {len(findings)} hallazgos potenciales:")
            for f in findings:
                print(f"      [{f['type']}] {f['value'][:50]}...")
            all_findings.extend(findings)
        else:
            print(f"    [+] Sin hallazgos evidentes")

    return all_findings

# Uso
if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Uso: python3 js_secret_hunter.py <archivo_con_urls_js>")
        sys.exit(1)

    findings = analyze_js_files(sys.argv[1])

    print(f"\n[+] RESUMEN: {len(findings)} hallazgos potenciales totales")
    for f in findings:
        print(f"\n  Tipo: {f['type']}")
        print(f"  URL:  {f['url']}")
        print(f"  Valor: {f['value']}")
```

---

## 5. Google Cache y otras cachés web

Wayback Machine no es la única fuente de contenido histórico. Los motores de búsqueda también mantienen cachés de las páginas que indexan, y estas cachés a veces contienen versiones más recientes que las de archive.org.

### Google Cache

Google Cache almacena la última versión indexada de cada página. Es especialmente útil para páginas que desaparecieron recientemente porque Google las cachea con más frecuencia que Wayback Machine:

```bash
# Acceder a la caché de Google de una URL específica
# En el navegador:
cache:https://example.com/admin/

# Con curl (menos fiable, Google suele bloquear scraping directo)
curl -s "https://webcache.googleusercontent.com/search?q=cache:https://example.com/admin/"

# Alternativa: buscar en Google con el operador cache:
# cache:example.com/api/internal/
# cache:example.com filetype:env
```

La ventaja de Google Cache es que contiene la versión más reciente indexada por Google, que puede ser de días o semanas atrás en lugar de meses. La desventaja es que Google elimina la caché cuando el sitio pide que no se indexe (robots.txt con Disallow o meta noindex), y también cuando el propietario solicita la eliminación.

### Bing Cache

Bing mantiene su propia caché independiente de Google. En algunos casos tiene versiones diferentes de las mismas páginas:

```bash
# En el navegador: buscar en Bing y hacer clic en "Cached" en los resultados
# O usar la URL directa:
# https://cc.bingj.com/cache.aspx?q=&url=https://example.com/endpoint/

# Buscar en Bing con operadores específicos para encontrar páginas en caché
# En Bing: url:example.com/admin cache
```

### Common Crawl — El archivo académico

Common Crawl es un archivo público del contenido de internet mantenido con fines de investigación. Es menos completo que Wayback Machine en términos de cobertura histórica, pero sus datos están disponibles en bulk para análisis masivo:

```bash
# La API de Common Crawl (similar a la CDX de Wayback Machine)
curl -s "https://index.commoncrawl.org/CC-MAIN-2024-10-index?url=example.com/*\
&output=json\
&limit=100"

# Listar todos los índices disponibles (actualizados mensualmente)
curl -s "https://index.commoncrawl.org/collinfo.json" | python3 -m json.tool

# Herramienta cdx-toolkit para consultas avanzadas a Common Crawl
pip3 install cdx-toolkit
cdx-toolkit --cc --limit 1000 iter 'example.com/*' --fields url,status,mime
```

### URLScan.io — Escaneos recientes de URLs

URLScan.io es un servicio que escanea URLs bajo demanda y archiva los resultados. Mantiene un historial de todos los escaneos realizados, lo que puede revelar información sobre cómo se veía un sitio recientemente:

```bash
# Buscar escaneos históricos de un dominio
curl -s "https://urlscan.io/api/v1/search/?q=domain:example.com&size=100" \
    | python3 -m json.tool

# La respuesta incluye screenshots, DOM, JS cargado, y mucho más

# Buscar por IP
curl -s "https://urlscan.io/api/v1/search/?q=ip:93.184.216.34&size=100" \
    | python3 -m json.tool

# Buscar subdominios no conocidos
curl -s "https://urlscan.io/api/v1/search/?q=domain:example.com&size=1000" \
    | python3 -c "
import sys, json
data = json.load(sys.stdin)
domains = set()
for result in data.get('results', []):
    page = result.get('page', {})
    domains.add(page.get('domain', ''))
for d in sorted(domains):
    print(d)
"
```

---

## 6. Reconocimiento de cambios en la infraestructura

Una técnica avanzada es comparar el estado histórico del objetivo con el estado actual para identificar qué ha cambiado: subdominios que desaparecieron (pueden seguir funcionando), tecnologías que fueron reemplazadas (la versión antigua puede seguir activa), y recursos que fueron "eliminados" pero que el servidor aún sirve.

### Script de comparación histórico vs actual

```python
#!/usr/bin/env python3
"""
wayback_diff.py — Compara URLs históricas con el estado actual del servidor
Encuentra endpoints que fueron eliminados del sitio pero que el servidor aún puede servir
"""

import requests
import sys
import time
from concurrent.futures import ThreadPoolExecutor, as_completed

def check_url_current_status(url, timeout=10):
    """
    Comprueba si una URL que existió en el pasado sigue respondiendo hoy.
    Devuelve el código de estado HTTP o None si no responde.
    """
    try:
        response = requests.get(
            url,
            timeout=timeout,
            allow_redirects=False,  # No seguir redirecciones para ver el estado real
            headers={
                "User-Agent": "Mozilla/5.0 (compatible; reconbot/1.0)"
            }
        )
        return response.status_code
    except requests.RequestException:
        return None

def load_historical_urls(filename):
    """Carga URLs históricas del archivo generado por waybackurls o gau."""
    with open(filename) as f:
        return [line.strip() for line in f if line.strip() and line.startswith("http")]

def check_urls_parallel(urls, max_workers=20):
    """
    Comprueba el estado actual de múltiples URLs en paralelo.
    Devuelve dict con URL → código de estado.
    """
    results = {}

    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        future_to_url = {
            executor.submit(check_url_current_status, url): url
            for url in urls
        }

        for i, future in enumerate(as_completed(future_to_url), 1):
            url = future_to_url[future]
            status = future.result()
            results[url] = status

            if i % 100 == 0:
                print(f"  Progreso: {i}/{len(urls)} URLs verificadas")

    return results

def analyze_results(results):
    """
    Clasifica los resultados por código de respuesta.
    Los 200 son los más interesantes — recursos que siguen vivos.
    Los 403 también son valiosos — el recurso existe pero requiere autenticación.
    """
    by_status = {}
    for url, status in results.items():
        if status not in by_status:
            by_status[status] = []
        by_status[status].append(url)

    return by_status

def main():
    if len(sys.argv) < 2:
        print("Uso: python3 wayback_diff.py <archivo_urls_historicas>")
        print("     El archivo se genera con: echo 'ejemplo.com' | waybackurls > urls.txt")
        sys.exit(1)

    urls_file = sys.argv[1]
    urls = load_historical_urls(urls_file)

    print(f"[*] Cargadas {len(urls)} URLs históricas")
    print(f"[*] Verificando estado actual en paralelo...")

    results = check_urls_parallel(urls)
    classified = analyze_results(results)

    print(f"\n{'='*60}")
    print("  ANÁLISIS DE ENDPOINTS HISTÓRICOS")
    print(f"{'='*60}")

    # Los 200 son los más interesantes — siguen vivos
    alive_200 = classified.get(200, [])
    print(f"\n[!] VIVOS (HTTP 200): {len(alive_200)} endpoints")
    for url in sorted(alive_200)[:50]:
        print(f"    {url}")

    # Los 403 también son valiosos
    forbidden_403 = classified.get(403, [])
    print(f"\n[+] PROTEGIDOS (HTTP 403): {len(forbidden_403)} endpoints")
    for url in sorted(forbidden_403)[:20]:
        print(f"    {url}")

    # Los 401 requieren autenticación
    unauth_401 = classified.get(401, [])
    print(f"\n[+] REQUIEREN AUTH (HTTP 401): {len(unauth_401)} endpoints")
    for url in sorted(unauth_401)[:20]:
        print(f"    {url}")

    # Los 301/302 redirigen — puede ser interesante ver adónde
    redirects = classified.get(301, []) + classified.get(302, [])
    print(f"\n[*] REDIRIGEN (301/302): {len(redirects)} endpoints")

    # Resumen de no encontrados
    not_found = classified.get(404, [])
    no_response = classified.get(None, [])
    print(f"\n[-] No encontrados (404): {len(not_found)}")
    print(f"[-] Sin respuesta: {len(no_response)}")

    # Guardar los endpoints vivos para análisis posterior
    output_file = "endpoints_vivos.txt"
    with open(output_file, "w") as f:
        for url in sorted(alive_200 + forbidden_403 + unauth_401):
            f.write(url + "\n")

    print(f"\n[+] Endpoints interesantes guardados en: {output_file}")

if __name__ == "__main__":
    main()
```

---

## 7. Flujo completo de reconocimiento histórico

Integrando todas las técnicas anteriores en un flujo de trabajo ordenado:

```bash
#!/bin/bash
# wayback_recon.sh — Reconocimiento histórico completo
# Uso: ./wayback_recon.sh example.com

TARGET=$1
OUTPUT="./wayback_recon_${TARGET}"
mkdir -p "$OUTPUT"/{urls,filtered,content}

echo "╔══════════════════════════════════════════╗"
echo "  Reconocimiento histórico: $TARGET"
echo "╚══════════════════════════════════════════╝"

# Fase 1: Recolección de URLs históricas
echo "[1/5] Recolectando URLs desde múltiples fuentes..."

# waybackurls
echo "  [+] Wayback Machine..."
echo "$TARGET" | waybackurls 2>/dev/null | sort -u > "$OUTPUT/urls/wayback.txt"
echo "      $(wc -l < $OUTPUT/urls/wayback.txt) URLs"

# gau (combina Wayback + Common Crawl + URLScan)
echo "  [+] gau (múltiples fuentes)..."
gau "$TARGET" 2>/dev/null | sort -u > "$OUTPUT/urls/gau.txt"
echo "      $(wc -l < $OUTPUT/urls/gau.txt) URLs"

# CDX API directa para subdominios
echo "  [+] CDX API (incluyendo subdominios)..."
curl -s "http://web.archive.org/cdx/search/cdx?url=*.$TARGET/*&output=text&fl=original&collapse=urlkey" \
    2>/dev/null | sort -u > "$OUTPUT/urls/cdx_subdominios.txt"
echo "      $(wc -l < $OUTPUT/urls/cdx_subdominios.txt) URLs"

# Combinar todo
cat "$OUTPUT/urls/"*.txt | sort -u > "$OUTPUT/urls/todas.txt"
TOTAL=$(wc -l < "$OUTPUT/urls/todas.txt")
echo "  [*] Total único: $TOTAL URLs"

# Fase 2: Filtrado por categorías
echo "[2/5] Filtrando URLs por categoría..."

grep -iE "\.js(\?|$)" "$OUTPUT/urls/todas.txt" > "$OUTPUT/filtered/js.txt"
echo "  JavaScript: $(wc -l < $OUTPUT/filtered/js.txt)"

grep -iE "\.json(\?|$)" "$OUTPUT/urls/todas.txt" > "$OUTPUT/filtered/json.txt"
echo "  JSON: $(wc -l < $OUTPUT/filtered/json.txt)"

grep -iE "\.(env|conf|config|ini|cfg|bak|backup|sql|log)(\?|$)" \
    "$OUTPUT/urls/todas.txt" > "$OUTPUT/filtered/sensibles.txt"
echo "  Archivos sensibles: $(wc -l < $OUTPUT/filtered/sensibles.txt)"

grep -iE "/(admin|panel|dashboard|backend|console|manage|api)" \
    "$OUTPUT/urls/todas.txt" > "$OUTPUT/filtered/admin_api.txt"
echo "  Admin/API: $(wc -l < $OUTPUT/filtered/admin_api.txt)"

grep -iE "[?&](id|user|file|url|path|redirect)=" \
    "$OUTPUT/urls/todas.txt" > "$OUTPUT/filtered/params_interesantes.txt"
echo "  Params interesantes: $(wc -l < $OUTPUT/filtered/params_interesantes.txt)"

# Fase 3: Verificar endpoints vivos
echo "[3/5] Verificando qué endpoints siguen vivos..."
if command -v httpx &> /dev/null; then
    # httpx es más rápido y fiable para esto
    cat "$OUTPUT/filtered/admin_api.txt" \
        "$OUTPUT/filtered/sensibles.txt" \
        | sort -u \
        | httpx -silent -status-code -o "$OUTPUT/filtered/endpoints_vivos.txt" 2>/dev/null
    echo "  Endpoints vivos: $(wc -l < $OUTPUT/filtered/endpoints_vivos.txt)"
else
    echo "  [-] httpx no instalado, omitiendo verificación activa"
fi

# Fase 4: Buscar subdominios en las URLs históricas
echo "[4/5] Extrayendo subdominios del historial..."
grep -oE "https?://[a-zA-Z0-9\-]+\.$TARGET" "$OUTPUT/urls/todas.txt" \
    | sed 's|https\?://||' \
    | sort -u > "$OUTPUT/urls/subdominios_historicos.txt"
echo "  Subdominios en historial: $(wc -l < $OUTPUT/urls/subdominios_historicos.txt)"

# Fase 5: Resumen
echo "[5/5] Generando resumen..."
echo ""
echo "╔══════════════════════════════════════════╗"
echo "  RECONOCIMIENTO HISTÓRICO COMPLETADO"
echo "╠══════════════════════════════════════════╣"
printf "  %-30s %s\n" "URLs totales históricas:" "$TOTAL"
printf "  %-30s %s\n" "Archivos JavaScript:" "$(wc -l < $OUTPUT/filtered/js.txt)"
printf "  %-30s %s\n" "Archivos sensibles:" "$(wc -l < $OUTPUT/filtered/sensibles.txt)"
printf "  %-30s %s\n" "Endpoints Admin/API:" "$(wc -l < $OUTPUT/filtered/admin_api.txt)"
printf "  %-30s %s\n" "Subdominios históricos:" "$(wc -l < $OUTPUT/urls/subdominios_historicos.txt)"
echo "╠══════════════════════════════════════════╣"
echo "  Resultados: $OUTPUT/"
echo "╚══════════════════════════════════════════╝"

echo ""
echo "[!] Revisar manualmente:"
echo "    $OUTPUT/filtered/sensibles.txt   → archivos que no deberían ser públicos"
echo "    $OUTPUT/filtered/admin_api.txt   → paneles y endpoints de API"
echo "    $OUTPUT/filtered/js.txt          → buscar secretos con js_secret_hunter.py"
```

---

## 8. Cheatsheet de referencia rápida

```bash
# ── WAYBACK MACHINE — API CDX ────────────────────────────────────────────
# Todas las URLs de un dominio
curl -s "http://web.archive.org/cdx/search/cdx?url=ejemplo.com/*&output=text&fl=original&collapse=urlkey"

# Solo HTTP 200
curl -s "http://web.archive.org/cdx/search/cdx?url=ejemplo.com/*&output=text&fl=original&filter=statuscode:200&collapse=urlkey"

# Solo JS
curl -s "http://web.archive.org/cdx/search/cdx?url=ejemplo.com/*.js&output=text&fl=original&collapse=urlkey"

# Con subdominios
curl -s "http://web.archive.org/cdx/search/cdx?url=*.ejemplo.com/*&output=text&fl=original&collapse=urlkey"

# ── HERRAMIENTAS CLI ──────────────────────────────────────────────────────
echo "ejemplo.com" | waybackurls > wayback.txt          # URLs de Wayback Machine
gau ejemplo.com > gau.txt                               # URLs de múltiples fuentes
cat wayback.txt gau.txt | sort -u > todas.txt           # Combinar y deduplicar

# ── FILTRADO ESENCIAL ─────────────────────────────────────────────────────
grep -E "\.js$" todas.txt                               # JavaScript
grep -iE "\.(env|bak|sql|config|conf)$" todas.txt       # Archivos sensibles
grep -iE "/(admin|api|panel|backend)" todas.txt         # Admin y API
grep -iE "[?&](id|url|file|redirect)=" todas.txt        # Parámetros peligrosos
grep -iE "/(login|auth|token|oauth)" todas.txt          # Autenticación

# ── VERIFICAR ENDPOINTS VIVOS ─────────────────────────────────────────────
cat urls_interesantes.txt | httpx -silent -status-code  # Con httpx
# Filtrar solo los que dan 200, 403 o 401:
cat urls.txt | httpx -silent -status-code | grep -E "\[200\]|\[403\]|\[401\]"

# ── CACHÉ DE MOTORES DE BÚSQUEDA ──────────────────────────────────────────
# Google: cache:https://ejemplo.com/endpoint/
# Bing: buscar en bing.com y usar "Cached" en los resultados
# URLScan: curl -s "https://urlscan.io/api/v1/search/?q=domain:ejemplo.com&size=100"

# ── INSTALACIÓN DE HERRAMIENTAS ───────────────────────────────────────────
go install github.com/tomnomnom/waybackurls@latest      # waybackurls
go install github.com/lc/gau/v2/cmd/gau@latest          # gau
go install github.com/projectdiscovery/httpx/cmd/httpx@latest # httpx

# ── RECURSOS Y REFERENCIAS ────────────────────────────────────────────────
https://web.archive.org                     → Wayback Machine web
https://web.archive.org/cdx/search/         → Documentación API CDX
https://index.commoncrawl.org               → Common Crawl API
https://urlscan.io/api/v1/                  → URLScan.io API
https://github.com/tomnomnom/waybackurls    → waybackurls
https://github.com/lc/gau                   → gau
https://github.com/hakluke/hakrawler        → hakrawler
```
