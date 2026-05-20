---
layout: post
title: "Censys y FOFA — Alternativas Avanzadas a Shodan"
date: 2025-05-17
categories: [reconocimiento]
tags: [censys, fofa, shodan, osint, reconocimiento, recon, infraestructura, certificados, internet-scan]
description: "Guía profesional de Censys y FOFA: búsqueda de infraestructura expuesta, análisis de certificados SSL, queries avanzadas y flujos reales para reconocimiento de infraestructura en pentesting y Red Team."
---

## El problema con depender solo de Shodan

Shodan es la herramienta más conocida para buscar dispositivos expuestos en internet, y por una buena razón: fue la pionera y tiene una interfaz muy accesible. Pero en reconocimiento profesional depender de una sola fuente es un error. Cada motor de escaneo de internet tiene su propio scanner, su propia frecuencia de actualización, sus propios rangos que prioriza, y su propio conjunto de datos. Lo que Shodan no tiene indexado puede que Censys sí lo tenga, y lo que ambos se pierden quizás FOFA lo captura.

En un engagement serio, la diferencia entre encontrar un panel de administración expuesto y no encontrarlo puede depender de qué herramienta uses. Hemos visto casos donde Censys mostraba un servicio que Shodan no había indexado en meses, y viceversa. Por eso los profesionales de Red Team usan los tres y cruzan los resultados.

Este post cubre Censys y FOFA en profundidad: cómo funcionan internamente, qué los diferencia de Shodan, cómo hacer las búsquedas más efectivas, y cómo integrarlos en un flujo de reconocimiento real.

---

## 1. Censys — El escáner académico que se hizo profesional

Censys nació como proyecto de investigación en la Universidad de Michigan. Su objetivo original era entender el estado real de seguridad de internet escaneando todo el espacio IPv4 de forma continua. Ese origen académico se nota en su diseño: Censys está obsesionado con la precisión, la cobertura y la estructura de los datos, más que con la accesibilidad inmediata.

Lo que hace único a Censys frente a Shodan es su **enfoque en certificados SSL/TLS y en la estructura de datos**. Censys no solo dice "este host tiene el puerto 443 abierto" — dice exactamente qué certificado tiene, cuándo expira, quién lo emitió, qué dominios cubre, qué algoritmos usa, si está en cadena de confianza o es autofirmado, y cuántas veces ha cambiado en los últimos meses. Para reconocimiento de infraestructura basado en certificados, Censys no tiene rival.

Censys escanea continuamente todo el espacio IPv4 (aproximadamente 4.000 millones de IPs) y también los puertos más comunes de IPv6. Los datos se actualizan con frecuencia variable según el servicio, pero en general la información tiene pocas horas de antigüedad para los puertos más comunes.

### Acceso a Censys

Censys tiene tres niveles de acceso:

```
Community (gratuito):
- Búsquedas ilimitadas en la web (search.censys.io)
- API: 250 queries al mes
- Acceso a datos de hosts e IPs
- Sin acceso a datos históricos

Individual (gratuito con registro):
- 250 queries API al mes
- Exportación limitada de resultados
- Acceso a algunos datos de certificados

Teams/Enterprise (pago):
- Queries API ilimitadas
- Datos históricos completos
- Alertas de cambios en infraestructura
- Acceso a todos los datasets
```

Para reconocimiento básico el plan gratuito es suficiente. Crea una cuenta en [search.censys.io](https://search.censys.io) y obtén tu API ID y API Secret desde el panel de cuenta.

### La interfaz web de Censys

La web de Censys tiene dos secciones principales que conviene conocer antes de usar la API:

**Hosts** (`search.censys.io/hosts`): busca en los datos de IPs y servicios. Aquí puedes encontrar qué puertos tiene abiertos una IP, qué banners devuelve cada servicio, qué certificados SSL tiene instalados, y cuál es la geolocalización y organización propietaria.

**Certificates** (`search.censys.io/certificates`): busca directamente en la base de datos de certificados SSL/TLS. Esta es la sección más potente y diferenciadora de Censys — puedes buscar certificados por organización, por dominio, por hash, por emisor, y obtener todos los dominios que aparecen en esos certificados.

---

## 2. Lenguaje de búsqueda de Censys

El lenguaje de búsqueda de Censys es más estructurado y preciso que el de Shodan. Usa una sintaxis similar a Elasticsearch donde los campos tienen nombres que reflejan exactamente la estructura de los datos.

### Búsquedas básicas en Hosts

```
# Por organización — campo autonomous_system.name
autonomous_system.name="Example Corp"

# Por rango de IPs (CIDR)
ip: 93.184.216.0/24

# Por ASN (número de sistema autónomo)
autonomous_system.asn=15169

# Por país
location.country="Spain"
location.country_code="ES"

# Por ciudad
location.city="Madrid"

# Servicios específicos — por protocolo y puerto
services.port=22
services.port=3389
services.transport_protocol="TCP"

# Combinaciones — infraestructura de una empresa con SSH expuesto
autonomous_system.name="Example Corp" and services.port=22

# Buscar por producto/software específico
services.software.product="nginx"
services.software.version="1.18.0"

# Buscar por banner — contenido del banner del servicio
services.banner="Example Corp Internal"
```

### Búsquedas avanzadas en Hosts

Aquí es donde Censys realmente brilla frente a otras herramientas. La profundidad de los datos que indexa permite hacer búsquedas muy específicas:

```
# Hosts con certificados SSL expirados — indica mantenimiento descuidado
services.tls.certificate.parsed.validity.end < "2024-01-01"

# Hosts con certificados autofirmados — potencialmente interesante
services.tls.certificate.parsed.issuer.common_name=
  services.tls.certificate.parsed.subject.common_name

# Hosts con certificados de una organización específica en el CN
services.tls.certificate.parsed.subject.organization="Example Corp"

# Hosts con un dominio específico en el certificado
# Esto encuentra la IP real aunque esté detrás de Cloudflare
services.tls.certificate.parsed.names="example.com"

# Hosts con certificados que cubren múltiples dominios (SANs)
services.tls.certificate.parsed.subject_alt_name.dns_names="*.example.com"

# Versiones de TLS antiguas — SSLv3, TLS 1.0, TLS 1.1 (vulnerables)
services.tls.version="TLSv1_0"
services.tls.version="SSLv3"

# Algoritmos de cifrado débiles
services.tls.cipher_selected="TLS_RSA_WITH_RC4_128_MD5"

# Hosts con HTTP Basic Auth activo
services.http.response.headers.www_authenticate="Basic*"

# Paneles de administración por título de página
services.http.response.html_title="Admin"
services.http.response.html_title="Dashboard"
services.http.response.html_title="phpMyAdmin"
services.http.response.html_title="Kibana"
services.http.response.html_title="Grafana"
services.http.response.html_title="Jenkins"

# Bases de datos expuestas
services.port=27017 and services.service_name="MONGODB"
services.port=9200 and services.service_name="ELASTICSEARCH"
services.port=6379 and services.service_name="REDIS"
services.port=5432 and services.service_name="POSTGRESQL"

# RDP expuesto a internet
services.port=3389 and services.service_name="RDP"

# Servidores con vulnerabilidades conocidas (si Censys las ha marcado)
services.truncated=false and autonomous_system.name="Example Corp"
```

### Búsquedas en Certificates

La búsqueda en certificados es la función más diferenciadora de Censys. Puedes buscar certificados por cualquier campo y obtener los dominios que cubren:

```
# Todos los certificados emitidos para un dominio
parsed.names="example.com"

# Certificados que cubren cualquier subdominio (wildcard)
parsed.names="*.example.com"

# Certificados de una organización específica
parsed.subject.organization="Example Corp"

# Certificados emitidos por una CA específica
parsed.issuer.organization="Let's Encrypt"
parsed.issuer.organization="DigiCert Inc"

# Certificados próximos a expirar (útil en programas de bug bounty)
parsed.validity.end:[now TO now+30d]

# Certificados expirados
parsed.validity.end:[* TO now]

# Certificados con algoritmos de firma débiles (SHA1 — deprecado)
parsed.signature.hash_algorithm="SHA1WithRSA"

# Buscar por fingerprint SHA256 del certificado
parsed.fingerprint_sha256="d41d8cd98f00b204e9800998ecf8427e..."

# Certificados que contienen una cadena en cualquier campo del subject
parsed.subject.common_name="internal*"
parsed.subject.common_name="staging*"
parsed.subject.common_name="dev*"
parsed.subject.common_name="admin*"
```

---

## 3. Censys desde la línea de comandos y la API

La interfaz web es útil para exploración, pero para reconocimiento serio necesitas la API. Censys tiene una librería Python oficial que hace muy fácil integrar las búsquedas en scripts.

### Instalación y configuración

```bash
# Instalar la librería oficial de Python
pip3 install censys

# Configurar las credenciales (API ID y API Secret de search.censys.io)
# Opción 1: variables de entorno (recomendado para scripts)
export CENSYS_API_ID="tu_api_id_aqui"
export CENSYS_API_SECRET="tu_api_secret_aqui"

# Opción 2: archivo de configuración
censys config
# Sigue las instrucciones para introducir ID y Secret

# Verificar que funciona correctamente
censys account
```

### Búsquedas con la CLI de Censys

```bash
# Buscar hosts de una organización
censys search "autonomous_system.name='Example Corp'" --index hosts

# Buscar certificados con un dominio
censys search "parsed.names='example.com'" --index certificates

# Obtener información detallada de una IP específica
censys view 93.184.216.34 --index hosts

# Buscar y exportar a JSON para procesamiento posterior
censys search "autonomous_system.name='Example Corp'" \
    --index hosts \
    --fields ip,services.port,services.service_name \
    > hosts_example_corp.json

# Limitar el número de resultados
censys search "autonomous_system.name='Example Corp'" \
    --index hosts \
    --pages 2    # Cada página tiene 100 resultados → 200 total
```

### Búsquedas con la API Python

La librería Python da acceso completo a todas las funcionalidades de Censys y es la mejor opción para automatizar el reconocimiento:

```python
import censys.search

# Inicializar el cliente de búsqueda de hosts.
# Las credenciales se toman automáticamente de las variables de entorno
# o del archivo de configuración creado con 'censys config'
h = censys.search.CensysHosts()

# Búsqueda básica: hosts de una organización
# query es la búsqueda en lenguaje Censys
# fields especifica qué campos quieres recibir (importante para no sobrecargar la API)
results = h.search(
    query='autonomous_system.name="Example Corp"',
    fields=["ip", "services.port", "services.service_name",
            "services.software.product", "location.country"]
)

# Iterar sobre los resultados e imprimir la información relevante
for host in results:
    ip = host.get("ip", "N/A")
    services = host.get("services", [])
    country = host.get("location.country", "N/A")

    print(f"\n[+] IP: {ip} ({country})")
    for service in services:
        port = service.get("port", "?")
        name = service.get("service_name", "?")
        product = service.get("software", [{}])[0].get("product", "?") if service.get("software") else "?"
        print(f"    Puerto {port}/tcp — {name} ({product})")
```

```python
# Buscar hosts con certificados que cubren un dominio específico.
# Esta es la técnica más útil para encontrar la IP real detrás de un CDN.
# Si el servidor tiene el certificado de example.com instalado,
# Censys lo habrá indexado aunque esté detrás de Cloudflare.

h = censys.search.CensysHosts()

results = h.search(
    query='services.tls.certificate.parsed.names="example.com"',
    fields=["ip", "services.port", "services.tls.certificate.parsed.names",
            "autonomous_system.name", "location.country"]
)

print("[+] IPs con certificado de example.com:")
for host in results:
    ip = host.get("ip")
    asn_name = host.get("autonomous_system.name", "Desconocido")
    ports = [str(s.get("port")) for s in host.get("services", [])]
    print(f"    {ip} | ASN: {asn_name} | Puertos: {', '.join(ports)}")
```

```python
# Buscar certificados para enumerar subdominios.
# Los certificados SAN (Subject Alternative Names) pueden contener
# docenas de subdominios en un solo certificado.

c = censys.search.CensysCertificates()

results = c.search(
    query='parsed.names="*.example.com"',
    fields=["parsed.names", "parsed.subject.common_name",
            "parsed.validity.end", "parsed.issuer.organization"]
)

# Recopilar todos los dominios encontrados en los certificados
all_domains = set()
for cert in results:
    names = cert.get("parsed.names", [])
    for name in names:
        # Filtrar wildcards y quedarnos con subdominios reales
        if not name.startswith("*.") and "example.com" in name:
            all_domains.add(name)

print(f"[+] Subdominios encontrados en certificados: {len(all_domains)}")
for domain in sorted(all_domains):
    print(f"    {domain}")
```

### Script completo de reconocimiento con Censys

```python
#!/usr/bin/env python3
"""
censys_recon.py — Reconocimiento de infraestructura con Censys
Uso: python3 censys_recon.py example.com "Example Corp"
"""

import sys
import json
import censys.search
from datetime import datetime

def recon_hosts_by_org(org_name):
    """
    Busca todos los hosts indexados por Censys para una organización.
    Devuelve una lista de dicts con la información de cada host.
    """
    h = censys.search.CensysHosts()
    hosts = []

    print(f"[*] Buscando hosts de: {org_name}")
    try:
        results = h.search(
            query=f'autonomous_system.name="{org_name}"',
            fields=["ip", "services.port", "services.service_name",
                    "services.software.product", "services.software.version",
                    "location.country", "location.city",
                    "autonomous_system.asn"]
        )

        for host in results:
            hosts.append({
                "ip": host.get("ip"),
                "asn": host.get("autonomous_system.asn"),
                "country": host.get("location.country"),
                "city": host.get("location.city"),
                "services": [
                    {
                        "port": s.get("port"),
                        "name": s.get("service_name"),
                        "product": s.get("software", [{}])[0].get("product") if s.get("software") else None
                    }
                    for s in host.get("services", [])
                ]
            })
    except Exception as e:
        print(f"[-] Error buscando hosts: {e}")

    return hosts

def find_real_ip_behind_cdn(domain):
    """
    Busca la IP real de un dominio que puede estar detrás de un CDN.
    Censys indexa el certificado SSL del servidor real, no del CDN.
    """
    h = censys.search.CensysHosts()
    ips = []

    print(f"[*] Buscando IP real detrás del CDN para: {domain}")
    try:
        results = h.search(
            query=f'services.tls.certificate.parsed.names="{domain}"',
            fields=["ip", "autonomous_system.name", "services.port",
                    "location.country"]
        )

        for host in results:
            ips.append({
                "ip": host.get("ip"),
                "org": host.get("autonomous_system.name"),
                "country": host.get("location.country"),
                "ports": [s.get("port") for s in host.get("services", [])]
            })
    except Exception as e:
        print(f"[-] Error buscando IP real: {e}")

    return ips

def find_subdomains_via_certs(domain):
    """
    Enumera subdominios buscando en certificados SSL que cubran el dominio.
    Más efectivo que algunos métodos DNS pasivos.
    """
    c = censys.search.CensysCertificates()
    subdomains = set()

    print(f"[*] Buscando subdominios en certificados SSL para: {domain}")
    try:
        results = c.search(
            query=f'parsed.names="{domain}" or parsed.names="*.{domain}"',
            fields=["parsed.names", "parsed.subject.common_name",
                    "parsed.validity.end"]
        )

        for cert in results:
            for name in cert.get("parsed.names", []):
                name = name.strip().lstrip("*.")
                if domain in name and name != domain:
                    subdomains.add(name)
    except Exception as e:
        print(f"[-] Error buscando en certificados: {e}")

    return sorted(subdomains)

def main():
    if len(sys.argv) < 3:
        print("Uso: python3 censys_recon.py <dominio> <nombre_organización>")
        print("Ej:  python3 censys_recon.py example.com 'Example Corp'")
        sys.exit(1)

    domain = sys.argv[1]
    org = sys.argv[2]
    timestamp = datetime.now().strftime("%Y%m%d_%H%M")
    output_file = f"censys_recon_{domain}_{timestamp}.json"

    results = {
        "target": domain,
        "organization": org,
        "timestamp": timestamp,
        "hosts": [],
        "real_ips": [],
        "subdomains": []
    }

    # 1. Buscar hosts por organización
    hosts = recon_hosts_by_org(org)
    results["hosts"] = hosts
    print(f"[+] Hosts encontrados: {len(hosts)}")

    # 2. Buscar IP real detrás de CDN
    real_ips = find_real_ip_behind_cdn(domain)
    results["real_ips"] = real_ips
    print(f"[+] IPs con certificado del dominio: {len(real_ips)}")

    # 3. Enumerar subdominios via certificados
    subdomains = find_subdomains_via_certs(domain)
    results["subdomains"] = subdomains
    print(f"[+] Subdominios en certificados: {len(subdomains)}")

    # Guardar resultados
    with open(output_file, "w") as f:
        json.dump(results, f, indent=2)

    print(f"\n[+] Resultados guardados en: {output_file}")

    # Mostrar resumen
    print("\n── HOSTS CON SERVICIOS EXPUESTOS ──")
    for host in hosts[:10]:  # Mostrar solo los primeros 10
        ports = [str(s["port"]) for s in host["services"]]
        print(f"  {host['ip']} | {host['country']} | Puertos: {', '.join(ports)}")

    print(f"\n── IPs REALES DETRÁS DEL CDN ──")
    for ip_info in real_ips:
        print(f"  {ip_info['ip']} | {ip_info['org']} | Puertos: {ip_info['ports']}")

    print(f"\n── SUBDOMINIOS VIA CERTIFICADOS ──")
    for sub in subdomains[:20]:  # Mostrar solo los primeros 20
        print(f"  {sub}")

if __name__ == "__main__":
    main()
```

---

## 4. FOFA — El motor de búsqueda chino de infraestructura

FOFA (Finger Of Fingerprint, o simplemente FOFA) es un motor de búsqueda de activos de internet desarrollado por la empresa de ciberseguridad china Baimaohui. Aunque menos conocido en el mundo occidental, FOFA tiene características únicas que lo hacen especialmente valioso en reconocimiento profesional.

La ventaja principal de FOFA frente a Shodan y Censys es su **extensísima cobertura de infraestructura asiática y de regiones que los motores occidentales indexan con menos frecuencia**. Para objetivos con presencia en China, Corea del Sur, Japón, o Sudeste Asiático, FOFA frecuentemente tiene datos que Shodan o Censys simplemente no tienen.

Pero incluso para infraestructura occidental, FOFA es valioso porque usa técnicas de fingerprinting diferentes. Donde Shodan identifica un servidor por su banner y Censys por sus certificados, FOFA puede identificarlo por el contenido HTML de la página de error 404, por las cabeceras HTTP específicas, por cookies de sesión características, o por recursos estáticos únicos. Esto permite encontrar instancias de software que los otros motores no reconocerían.

### Acceso a FOFA

FOFA está disponible en [fofa.info](https://fofa.info). El registro es gratuito y da acceso a búsquedas básicas con limitaciones. Los planes de pago amplían el número de resultados y desbloquean filtros avanzados.

```
Plan gratuito:
- 1 query por día
- Máximo 100 resultados por búsqueda
- Sin exportación de datos

Plan básico (pago):
- 5 queries por día
- Hasta 10.000 resultados
- Exportación a CSV

Plan profesional:
- Queries ilimitadas
- Hasta 1.000.000 resultados
- API completa
- Datos históricos
```

Para reconocimiento serio necesitas al menos el plan básico. La API de FOFA requiere una cuenta con créditos F-Points.

### El lenguaje de búsqueda de FOFA

FOFA usa una sintaxis basada en campos que son muy descriptivos sobre el contenido y las características del host. La diferencia más notable respecto a Shodan y Censys es el énfasis en el contenido de la respuesta HTTP:

```
# Campos básicos de infraestructura

# IP y rango
ip="93.184.216.34"
ip="93.184.216.0/24"

# Puerto
port="8080"
port="3389"

# País y ciudad
country="CN"
country="ES"
city="Madrid"

# Organización (ASN name)
org="Example Corp"
asn="AS15169"

# Dominio
domain="example.com"
host="example.com"          # Busca en hostname, también en subdominios
```

```
# Campos de contenido HTTP — la ventaja diferencial de FOFA

# Título de la página web
title="Admin Panel"
title="Login"
title="phpMyAdmin"
title="Kibana"
title="Grafana"
title="Jenkins"

# Contenido del body HTML — búsqueda en el contenido de la respuesta
body="Powered by Example Corp"
body="Internal Dashboard"
body="api_key"              # Busca esta cadena en el HTML de la respuesta

# Cabeceras HTTP
header="X-Powered-By: ASP.NET"
header="Server: nginx/1.18.0"
header="Set-Cookie: PHPSESSID"

# Certificados SSL
cert="example.com"
cert.subject="Example Corp"
cert.issuer="Let's Encrypt"

# Iconos favicon — fingerprint único por aplicación
# El hash del favicon identifica software específico
icon_hash="-247388890"      # Hash del favicon de Kibana
icon_hash="1278323681"      # Hash del favicon de GitLab
```

### Queries avanzadas en FOFA

Las búsquedas combinadas de FOFA usando el contenido HTTP son su característica más diferenciadora:

```
# Paneles de administración expuestos de una organización
org="Example Corp" && title="Admin"
org="Example Corp" && title="Login"
org="Example Corp" && port="3389"

# Bases de datos expuestas
port="27017" && org="Example Corp"     # MongoDB
port="9200" && title="Elasticsearch"   # Elasticsearch sin auth
port="6379" && org="Example Corp"      # Redis

# Software específico por fingerprint de favicon
# Los hashes de favicon son únicos para cada aplicación
icon_hash="-247388890"                 # Kibana
icon_hash="-635947571"                 # Grafana
icon_hash="1278323681"                 # GitLab
icon_hash="-663540307"                 # Jira
icon_hash="999357577"                  # Confluence

# Combinación poderosa: favicon + organización
icon_hash="-247388890" && org="Example Corp"   # Kibana de Example Corp

# Encontrar infraestructura de desarrollo expuesta
title="staging" && org="Example Corp"
title="dev" && org="Example Corp"
header="X-Environment: staging" && org="Example Corp"

# Servidores con tecnologías específicas
header="X-Powered-By: PHP/5" && country="ES"   # PHP antiguo en España
body="wp-content" && org="Example Corp"         # WordPress

# Servicios expuestos por certificado (igual que Censys)
cert="example.com" && port="443"

# Encontrar todos los subdominios por certificado
cert="*.example.com"
```

### Calcular el hash del favicon

Una de las técnicas más potentes en FOFA es la búsqueda por hash de favicon. Cada aplicación web tiene un favicon que es prácticamente único. Si calculas el hash del favicon de un panel de administración, puedes encontrar todas las instancias de ese panel en internet.

```python
#!/usr/bin/env python3
"""
Calcula el hash mmh3 del favicon de una URL para usarlo en FOFA.
FOFA usa este hash específico (Murmur Hash 3) para identificar favicons.
"""

import requests
import codecs
import mmh3     # pip3 install mmh3
import base64

def get_favicon_hash(url):
    """
    Descarga el favicon de la URL y calcula su hash mmh3.
    Este hash es el que usa FOFA en el campo icon_hash.
    """
    # Intentar primero el favicon estándar
    favicon_url = url.rstrip("/") + "/favicon.ico"

    try:
        response = requests.get(
            favicon_url,
            timeout=10,
            allow_redirects=True,
            headers={"User-Agent": "Mozilla/5.0"}
        )

        if response.status_code == 200:
            # Codificar el contenido en base64
            favicon_base64 = base64.encodebytes(response.content)
            # Calcular el hash mmh3 del base64
            favicon_hash = mmh3.hash(favicon_base64)
            return favicon_hash
        else:
            print(f"[-] Favicon no encontrado en {favicon_url} (HTTP {response.status_code})")
            return None

    except requests.RequestException as e:
        print(f"[-] Error descargando favicon: {e}")
        return None

# Ejemplos de uso
urls = [
    "https://www.example.com",
    "https://admin.example.com",
    "https://kibana.example.com:5601",
]

for url in urls:
    hash_value = get_favicon_hash(url)
    if hash_value:
        print(f"[+] {url}")
        print(f"    Favicon hash: {hash_value}")
        print(f"    Query FOFA: icon_hash=\"{hash_value}\"")
```

### FOFA desde la API con Python

```python
import requests
import base64
import json

class FOFAClient:
    """
    Cliente Python para la API de FOFA.
    Requiere email y API key de tu cuenta en fofa.info
    """

    def __init__(self, email, api_key):
        self.email = email
        self.api_key = api_key
        self.base_url = "https://fofa.info/api/v1"

    def search(self, query, fields="ip,port,host,title,country", page=1, size=100):
        """
        Ejecuta una búsqueda en FOFA.

        query: La búsqueda en lenguaje FOFA (ej: 'domain="example.com"')
        fields: Campos a devolver separados por coma
        page: Número de página (paginación)
        size: Resultados por página (máximo según tu plan)
        """
        # FOFA requiere que la query esté en base64
        query_b64 = base64.b64encode(query.encode()).decode()

        params = {
            "email": self.email,
            "key": self.api_key,
            "qbase64": query_b64,
            "fields": fields,
            "page": page,
            "size": size,
            "full": "false"     # True para incluir datos históricos (requiere plan)
        }

        response = requests.get(f"{self.base_url}/search/all", params=params)

        if response.status_code == 200:
            data = response.json()
            if data.get("error"):
                print(f"[-] Error de API: {data.get('errmsg')}")
                return None
            return data
        else:
            print(f"[-] Error HTTP: {response.status_code}")
            return None

    def get_host_info(self, host):
        """Obtiene información detallada de un host o dominio."""
        params = {
            "email": self.email,
            "key": self.api_key,
            "host": host,
            "detail": "true"
        }
        response = requests.get(f"{self.base_url}/host/detail", params=params)
        return response.json() if response.status_code == 200 else None


# Ejemplo de uso completo
fofa = FOFAClient(
    email="tu_email@ejemplo.com",
    api_key="tu_api_key_de_fofa"
)

# Buscar hosts de una organización
print("[*] Buscando infraestructura de Example Corp en FOFA...")
results = fofa.search(
    query='org="Example Corp"',
    fields="ip,port,host,title,country,protocol"
)

if results:
    print(f"[+] Total resultados: {results['size']}")
    print(f"[+] Mostrando {len(results['results'])} resultados:")

    for item in results["results"]:
        ip, port, host, title, country, protocol = item
        print(f"  {ip}:{port} | {host} | {title} | {country} | {protocol}")

# Buscar por certificado para encontrar IPs reales
print("\n[*] Buscando IPs con certificado de example.com...")
cert_results = fofa.search(
    query='cert="example.com" && port="443"',
    fields="ip,port,host,country,org"
)

if cert_results:
    for item in cert_results["results"]:
        ip, port, host, country, org = item
        print(f"  {ip}:{port} | {host} | {country} | {org}")
```

---

## 5. Comparativa práctica: Shodan vs Censys vs FOFA

Entender las diferencias reales entre los tres motores te ayuda a decidir cuándo usar cada uno en un engagement:

### Fortalezas de cada herramienta

```
SHODAN:
✓ La más rápida de usar — interfaz muy accesible
✓ Mejor cobertura de dispositivos IoT y OT/SCADA
✓ Alertas y monitorización de IPs en tiempo real
✓ La más conocida — mucha documentación y ejemplos
✓ Dorks muy documentados y comunidad activa
✗ Datos de certificados menos detallados que Censys
✗ Menor cobertura de infraestructura asiática que FOFA

CENSYS:
✓ Los mejores datos de certificados SSL/TLS de los tres
✓ Ideal para encontrar IPs reales detrás de CDN
✓ Ideal para enumerar subdominios via certificados
✓ Datos más estructurados y precisos
✓ API mejor documentada y librería Python oficial
✗ Interfaz menos amigable para principiantes
✗ Plan gratuito muy limitado (250 API calls/mes)

FOFA:
✓ Mejor cobertura de infraestructura asiática
✓ Búsqueda por contenido HTML y cabeceras HTTP
✓ Identificación por hash de favicon — muy potente
✓ Encuentra software que los otros no fingerprinting
✗ Documentación principalmente en chino
✗ Plan gratuito muy restrictivo (1 query/día)
✗ Menos conocido en la comunidad occidental
```

### Cuándo usar cada uno

```
Usar SHODAN cuando:
→ Buscas dispositivos IoT, cámaras, routers, SCADA
→ Quieres resultados rápidos sin mucha configuración
→ Buscas con vuln: filtros de CVEs conocidos
→ Necesitas monitorizar cambios en una IP (alertas)

Usar CENSYS cuando:
→ Quieres encontrar la IP real detrás de Cloudflare
→ Necesitas análisis detallado de certificados SSL
→ Enumeras subdominios via CT logs
→ Buscas configuraciones TLS débiles o algoritmos inseguros

Usar FOFA cuando:
→ El objetivo tiene infraestructura en Asia
→ Quieres buscar por contenido HTML o cabeceras HTTP
→ Necesitas identificar software por favicon hash
→ Los otros motores no encuentran lo que buscas
```

---

## 6. Flujo combinado de los tres motores

En un engagement profesional, el flujo óptimo usa los tres motores de forma complementaria. Este script automatiza la búsqueda en los tres:

```python
#!/usr/bin/env python3
"""
triple_search.py — Reconocimiento de infraestructura con Shodan + Censys + FOFA
Uso: python3 triple_search.py "Example Corp" example.com
"""

import sys
import json
import shodan
import censys.search
import requests
import base64
from datetime import datetime

def search_shodan(api_key, org_name, domain):
    """Busca infraestructura en Shodan por organización y certificado."""
    api = shodan.Shodan(api_key)
    results = {"hosts": [], "source": "shodan"}

    try:
        # Por organización
        search = api.search(f'org:"{org_name}"')
        for result in search["matches"]:
            results["hosts"].append({
                "ip": result["ip_str"],
                "port": result["port"],
                "org": result.get("org"),
                "country": result.get("location", {}).get("country_name"),
                "product": result.get("product"),
                "version": result.get("version")
            })

        # Por certificado SSL — encuentra IPs reales detrás de CDN
        cert_search = api.search(f'ssl.cert.subject.CN:{domain}')
        for result in cert_search["matches"]:
            # Evitar duplicados
            ip = result["ip_str"]
            if not any(h["ip"] == ip for h in results["hosts"]):
                results["hosts"].append({
                    "ip": ip,
                    "port": result["port"],
                    "org": result.get("org"),
                    "note": "encontrado via certificado SSL"
                })

    except shodan.APIError as e:
        print(f"  [-] Shodan error: {e}")

    return results

def search_censys(api_id, api_secret, org_name, domain):
    """Busca infraestructura en Censys por organización y certificado."""
    import os
    os.environ["CENSYS_API_ID"] = api_id
    os.environ["CENSYS_API_SECRET"] = api_secret

    h = censys.search.CensysHosts()
    results = {"hosts": [], "subdomains": [], "source": "censys"}

    try:
        # Por organización
        for host in h.search(
            f'autonomous_system.name="{org_name}"',
            fields=["ip", "services.port", "services.service_name", "location.country"]
        ):
            results["hosts"].append({
                "ip": host.get("ip"),
                "country": host.get("location.country"),
                "ports": [s.get("port") for s in host.get("services", [])],
                "services": [s.get("service_name") for s in host.get("services", [])]
            })

        # Subdominios via certificados
        c = censys.search.CensysCertificates()
        for cert in c.search(
            f'parsed.names="{domain}" or parsed.names="*.{domain}"',
            fields=["parsed.names"]
        ):
            for name in cert.get("parsed.names", []):
                name = name.strip().lstrip("*.")
                if domain in name and name != domain:
                    if name not in results["subdomains"]:
                        results["subdomains"].append(name)

    except Exception as e:
        print(f"  [-] Censys error: {e}")

    return results

def search_fofa(email, api_key, org_name, domain):
    """Busca infraestructura en FOFA por organización."""
    results = {"hosts": [], "source": "fofa"}

    query = f'org="{org_name}" || cert="{domain}"'
    query_b64 = base64.b64encode(query.encode()).decode()

    try:
        resp = requests.get(
            "https://fofa.info/api/v1/search/all",
            params={
                "email": email,
                "key": api_key,
                "qbase64": query_b64,
                "fields": "ip,port,host,title,country",
                "size": 100
            },
            timeout=30
        )
        data = resp.json()
        if not data.get("error"):
            for item in data.get("results", []):
                ip, port, host, title, country = item
                results["hosts"].append({
                    "ip": ip,
                    "port": port,
                    "host": host,
                    "title": title,
                    "country": country
                })

    except Exception as e:
        print(f"  [-] FOFA error: {e}")

    return results

def main():
    if len(sys.argv) < 3:
        print("Uso: python3 triple_search.py 'Nombre Organización' dominio.com")
        sys.exit(1)

    org = sys.argv[1]
    domain = sys.argv[2]

    # Configurar credenciales (idealmente desde variables de entorno)
    SHODAN_KEY = "TU_SHODAN_KEY"
    CENSYS_ID = "TU_CENSYS_ID"
    CENSYS_SECRET = "TU_CENSYS_SECRET"
    FOFA_EMAIL = "TU_EMAIL_FOFA"
    FOFA_KEY = "TU_FOFA_KEY"

    print(f"╔══════════════════════════════════════════╗")
    print(f"  Reconocimiento triple: {org}")
    print(f"  Dominio: {domain}")
    print(f"╚══════════════════════════════════════════╝\n")

    all_ips = set()
    all_subdomains = set()
    full_results = []

    # Búsqueda en los tres motores
    print("[1/3] Shodan...")
    shodan_data = search_shodan(SHODAN_KEY, org, domain)
    for h in shodan_data["hosts"]:
        all_ips.add(h["ip"])
    full_results.append(shodan_data)
    print(f"      Hosts: {len(shodan_data['hosts'])}")

    print("[2/3] Censys...")
    censys_data = search_censys(CENSYS_ID, CENSYS_SECRET, org, domain)
    for h in censys_data["hosts"]:
        all_ips.add(h["ip"])
    for s in censys_data["subdomains"]:
        all_subdomains.add(s)
    full_results.append(censys_data)
    print(f"      Hosts: {len(censys_data['hosts'])} | Subdominios: {len(censys_data['subdomains'])}")

    print("[3/3] FOFA...")
    fofa_data = search_fofa(FOFA_EMAIL, FOFA_KEY, org, domain)
    for h in fofa_data["hosts"]:
        all_ips.add(h["ip"])
    full_results.append(fofa_data)
    print(f"      Hosts: {len(fofa_data['hosts'])}")

    # Resumen consolidado
    print(f"\n╔══════════════════════════════════════════╗")
    print(f"  RESULTADOS CONSOLIDADOS")
    print(f"╠══════════════════════════════════════════╣")
    print(f"  IPs únicas totales:   {len(all_ips)}")
    print(f"  Subdominios únicos:   {len(all_subdomains)}")
    print(f"╚══════════════════════════════════════════╝")

    # Guardar resultados
    output = {
        "organization": org,
        "domain": domain,
        "timestamp": datetime.now().isoformat(),
        "unique_ips": list(all_ips),
        "unique_subdomains": list(all_subdomains),
        "raw_results": full_results
    }

    outfile = f"triple_recon_{domain}.json"
    with open(outfile, "w") as f:
        json.dump(output, f, indent=2)

    print(f"\n[+] Resultados guardados en: {outfile}")

if __name__ == "__main__":
    main()
```

---

## 7. Cheatsheet de referencia rápida

```
── CENSYS — BÚSQUEDAS EN HOSTS ─────────────────────────────────────────
autonomous_system.name="Example Corp"              → Hosts de la organización
ip: 93.184.216.0/24                                → Por rango de IPs
services.port=3389                                 → RDP expuesto
services.port=27017                                → MongoDB expuesto
services.tls.certificate.parsed.names="example.com" → IP real detrás de CDN
services.http.response.html_title="Admin"          → Paneles admin
services.tls.version="TLSv1_0"                    → TLS 1.0 obsoleto
services.tls.certificate.parsed.validity.end < "2024-01-01" → Certs expirados

── CENSYS — BÚSQUEDAS EN CERTIFICATES ──────────────────────────────────
parsed.names="example.com"                         → Certs del dominio
parsed.names="*.example.com"                       → Certs wildcard
parsed.subject.organization="Example Corp"         → Certs de la org
parsed.issuer.organization="Let's Encrypt"         → Por emisor
parsed.signature.hash_algorithm="SHA1WithRSA"      → Algoritmo débil

── CENSYS — CLI ─────────────────────────────────────────────────────────
censys search "autonomous_system.name='Corp'" --index hosts
censys view 93.184.216.34 --index hosts
censys search "parsed.names='example.com'" --index certificates

── FOFA — BÚSQUEDAS ─────────────────────────────────────────────────────
domain="example.com"                               → Por dominio
org="Example Corp"                                 → Por organización
title="Admin Panel"                                → Por título
body="Powered by Example"                          → Por contenido HTML
header="X-Powered-By: PHP"                        → Por cabecera HTTP
cert="example.com"                                 → Por certificado
icon_hash="-247388890"                             → Por favicon (Kibana)
icon_hash="-635947571"                             → Por favicon (Grafana)
icon_hash="1278323681"                             → Por favicon (GitLab)

── FOFA — COMBINACIONES ─────────────────────────────────────────────────
org="Example Corp" && title="Login"
org="Example Corp" && port="3389"
cert="example.com" && port="443"
icon_hash="-247388890" && country="ES"

── SHODAN — REFERENCIA RÁPIDA ───────────────────────────────────────────
org:"Example Corp"                                 → Por organización
ssl.cert.subject.CN:example.com                    → IP real tras CDN
ssl:"Example Corp"                                 → Por nombre en cert
vuln:CVE-2021-44228                                → Por CVE

── CALCULAR FAVICON HASH PARA FOFA ─────────────────────────────────────
pip3 install mmh3 requests
python3 -c "
import requests, base64, mmh3
r = requests.get('https://ejemplo.com/favicon.ico')
print(mmh3.hash(base64.encodebytes(r.content)))
"

── RECURSOS ─────────────────────────────────────────────────────────────
https://search.censys.io          → Censys web
https://docs.censys.io            → Documentación Censys
https://pypi.org/project/censys/  → Librería Python Censys
https://fofa.info                 → FOFA web
https://fofa.info/api             → API FOFA
https://shodan.io                 → Shodan web
https://beta.shodan.io/search     → Shodan nuevo interfaz
```
