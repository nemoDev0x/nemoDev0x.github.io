---
layout: post
title: "Subfinder y Amass — Enumeración Masiva de Subdominios"
date: 2025-05-16
categories: [reconocimiento]
tags: [subfinder, amass, subdominios, dns, osint, reconocimiento, recon, bug-bounty]
description: "Guía profesional de Subfinder y Amass: instalación, configuración, fuentes pasivas, técnicas activas, automatización y flujos reales para enumeración de subdominios en pentesting y Bug Bounty."
---

## Por qué los subdominios son el objetivo número uno del reconocimiento

Cuando un pentester recibe un dominio como objetivo, lo primero que piensa no es en el sitio web principal. El sitio principal está bien mantenido, tiene WAF, tiene CDN, tiene un equipo que lo monitoriza. Lo interesante está en los subdominios: `dev.empresa.com` que olvidaron deshabilitar, `staging.empresa.com` con credenciales por defecto, `admin-old.empresa.com` con una versión sin parchear de un CMS, `api-v1.empresa.com` que quedó expuesto cuando migraron a la v2. La superficie de ataque real de cualquier organización está en su infraestructura olvidada, y los subdominios son la puerta de entrada a esa infraestructura.

La enumeración de subdominios tiene dos enfoques fundamentalmente distintos, y un buen reconocimiento usa ambos:

**Enumeración pasiva**: consultamos fuentes externas que ya tienen información sobre el objetivo — Certificate Transparency logs, motores de búsqueda, bases de datos de DNS pasivo, repositorios de threat intelligence. No enviamos ningún paquete al objetivo. Es completamente silenciosa.

**Enumeración activa**: enviamos peticiones DNS directamente, intentamos transferencias de zona, hacemos fuerza bruta de nombres. Genera tráfico hacia el objetivo y puede ser detectada, pero descubre subdominios que ninguna fuente pasiva conoce.

Subfinder y Amass son las dos herramientas más usadas en el sector para esta tarea, y se complementan perfectamente: Subfinder es más rápido y excelente en fuentes pasivas, Amass es más exhaustivo y tiene mejores capacidades activas y de análisis de grafos. En un engagement real se usan los dos y se combinan los resultados.

---

## 1. Subfinder — Reconocimiento pasivo de alta velocidad

Subfinder es una herramienta escrita en Go desarrollada por el equipo de ProjectDiscovery, los mismos que crearon Nuclei y otras herramientas que se han convertido en estándar en la industria. Está diseñada específicamente para ser rápida y para agregar el máximo número de fuentes pasivas posible.

El concepto central de Subfinder es simple: en lugar de que tú tengas que ir a cada fuente de subdominios por separado (crt.sh, SecurityTrails, VirusTotal, Shodan, etc.), Subfinder consulta todas en paralelo automáticamente y te devuelve una lista unificada y deduplicada. Lo que manualmente te llevaría una hora, Subfinder lo hace en segundos.

### Instalación de Subfinder

Subfinder está escrito en Go, lo que significa que el binario es un único archivo sin dependencias externas. La instalación más limpia es con `go install`:

```bash
# Método recomendado: instalar con go install
# Requiere Go 1.19 o superior instalado
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest

# Verificar que está en el PATH
subfinder -version

# En Kali Linux viene preinstalado, pero puede ser una versión antigua
# Para actualizar a la última versión:
sudo apt remove subfinder
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest

# Instalación con apt en Kali (versión del repositorio, puede ser antigua)
sudo apt update && sudo apt install subfinder

# En macOS con Homebrew
brew install subfinder

# Descarga directa del binario desde GitHub Releases
# Útil en servidores sin Go instalado
wget https://github.com/projectdiscovery/subfinder/releases/latest/download/subfinder_linux_amd64.zip
unzip subfinder_linux_amd64.zip
chmod +x subfinder
sudo mv subfinder /usr/local/bin/
```

### Configuración de API keys en Subfinder

Subfinder funciona sin ninguna API key — las fuentes públicas gratuitas ya dan buenos resultados. Pero si añades API keys de servicios premium, la cobertura mejora considerablemente. Las keys se configuran en un archivo YAML en el directorio de configuración de Subfinder.

El archivo de configuración se crea automáticamente la primera vez que ejecutas Subfinder:

```bash
# La primera ejecución crea el directorio de configuración
subfinder -d example.com

# El archivo de configuración está en:
cat ~/.config/subfinder/provider-config.yaml
```

El archivo tiene esta estructura. Puedes añadir las keys que tengas disponibles y dejar en blanco las que no:

```yaml
# ~/.config/subfinder/provider-config.yaml
# Cada servicio tiene su sección con la API key correspondiente.
# Puedes tener múltiples keys para el mismo servicio (útil para evitar rate limits).

binaryedge:
  - TU_API_KEY_BINARYEDGE

certspotter:
  - TU_API_KEY_CERTSPOTTER

censys:
  - TU_API_KEY_ID:TU_API_KEY_SECRET    # Formato especial: ID:SECRET

chaos:
  - TU_API_KEY_CHAOS                   # ProjectDiscovery Chaos Dataset

github:
  - TU_GITHUB_TOKEN_1
  - TU_GITHUB_TOKEN_2                  # Puedes poner múltiples tokens

hunter:
  - TU_API_KEY_HUNTER

passivetotal:
  - TU_USERNAME:TU_API_KEY             # Formato: usuario:key

securitytrails:
  - TU_API_KEY_SECURITYTRAILS

shodan:
  - TU_API_KEY_SHODAN

virustotal:
  - TU_API_KEY_VIRUSTOTAL

# Fuentes gratuitas que no necesitan key:
# crt.sh, hackertarget, dnsdumpster, rapiddns, bufferover, etc.
# Subfinder las usa automáticamente sin configuración
```

Para obtener estas keys, las fuentes más importantes son:

```bash
# SecurityTrails — 50 queries gratuitas al mes
# https://securitytrails.com/app/account/credentials

# VirusTotal — gratuito con registro, límite de 4 req/min
# https://www.virustotal.com/gui/my-apikey

# GitHub — token gratuito, muy útil para encontrar subdominios en repos
# https://github.com/settings/tokens
# Permisos necesarios: solo lectura (public_repo)

# Shodan — gratuito con registro
# https://account.shodan.io

# Chaos Dataset de ProjectDiscovery — requiere solicitar acceso
# https://chaos.projectdiscovery.io
```

### Uso básico de Subfinder

Una vez instalado, el uso más simple es darle un dominio y obtener los subdominios:

```bash
# Escaneo básico — usa todas las fuentes disponibles con las keys configuradas
# El output va directamente a la terminal, un subdominio por línea
subfinder -d example.com

# Guardar resultados en un archivo para procesarlos después
# -o especifica el archivo de salida
subfinder -d example.com -o subdominios.txt

# Modo silencioso — solo muestra los subdominios, sin el banner ni los mensajes de estado
# Útil cuando pipeas el output a otro comando
subfinder -d example.com -silent

# Activar todas las fuentes disponibles, incluso las más lentas
# Sin -all, Subfinder usa solo las fuentes más rápidas
subfinder -d example.com -all -o subdominios_completo.txt

# Ver de qué fuente proviene cada subdominio
# Útil para entender qué tan expuesto está el objetivo y qué fuentes son más productivas
subfinder -d example.com -v

# Combinar -all y -v para máxima información
subfinder -d example.com -all -v -o subdominios_verbose.txt
```

### Escaneo de múltiples dominios

En engagements reales raramente tienes un solo dominio. Las empresas grandes tienen múltiples dominios, y el scope puede incluir decenas. Subfinder puede procesarlos todos en paralelo:

```bash
# Múltiples dominios directamente en la línea de comandos
subfinder -d example.com -d example.net -d subsidiary.com -o todos_los_subdominios.txt

# Desde un archivo de texto con un dominio por línea
# Este es el método más cómodo cuando tienes muchos dominios
cat dominios_objetivo.txt
# example.com
# example.net
# subsidiary.com
# acquired-company.com

subfinder -dL dominios_objetivo.txt -o todos_los_subdominios.txt

# Ver el progreso mientras se ejecuta
subfinder -dL dominios_objetivo.txt -all -o resultado.txt -v

# Controlar el nivel de paralelismo (threads)
# Por defecto usa 10 threads, aumentar si tienes buena conexión
subfinder -d example.com -t 50 -o subdominios.txt
```

### Filtrado y procesamiento de resultados

Subfinder tiene opciones para filtrar el output directamente, evitando que tengas que procesar la lista después:

```bash
# Excluir subdominios específicos del resultado
# Útil cuando algunos subdominios están fuera de scope
subfinder -d example.com -exclude-sources github,dnsdumpster -o resultado.txt

# Resolver los subdominios encontrados y mostrar solo los activos
# Combina la enumeración con una verificación DNS
subfinder -d example.com -all -o subdominios.txt
# Luego resolver con dnsx:
cat subdominios.txt | dnsx -silent -o subdominios_activos.txt

# Filtrar wildcards — algunos dominios tienen *.example.com que genera falsos positivos
subfinder -d example.com -all | grep -v "^\*\." > subdominios_limpios.txt
```

---

## 2. Amass — Reconocimiento exhaustivo y análisis de grafos

Amass es una herramienta desarrollada por OWASP que va mucho más allá de simplemente listar subdominios. Mientras Subfinder es un buscador de subdominios muy rápido, Amass es un motor de inteligencia completo que puede construir un grafo de relaciones entre entidades, mapear ASNs completos, hacer fuerza bruta inteligente, y generar visualizaciones de la infraestructura del objetivo.

La diferencia fundamental en la filosofía de diseño es esta: Subfinder te da una lista, Amass te da entendimiento. Amass entiende las relaciones entre los subdominios, los rangos de IPs, los sistemas autónomos y las organizaciones, y puede usar esas relaciones para descubrir más activos que un simple listado de subdominios nunca revelaría.

La contrapartida es que Amass es significativamente más lento que Subfinder. En un dominio grande, una enumeración completa con Amass puede tardar horas. Esto es normal y esperado — está haciendo un trabajo mucho más profundo.

### Instalación de Amass

```bash
# Instalación con go install (recomendado para tener la última versión)
go install -v github.com/owasp-amass/amass/v4/...@master

# En Kali Linux
sudo apt update && sudo apt install amass

# En macOS con Homebrew
brew install amass

# Con Docker (útil si no quieres instalar Go)
docker pull caffix/amass
docker run -v OUTPUT_DIR_FULL_PATH:/root/output caffix/amass enum -d example.com

# Verificar la instalación
amass -version
```

### Subcomandos de Amass

A diferencia de Subfinder que hace una sola cosa, Amass tiene múltiples subcomandos, cada uno para una tarea diferente:

```bash
# enum — el subcomando principal de enumeración
amass enum -d example.com

# intel — recolección de información sobre la organización (ASN, rangos IP, etc.)
amass intel -org "Example Corp"

# viz — generar visualizaciones del grafo de infraestructura
amass viz -d3 -o grafo.html

# track — comparar dos enumeraciones para detectar cambios
amass track -d example.com

# db — gestionar la base de datos local de Amass
amass db -list     # listar enumeraciones guardadas
amass db -show     # mostrar resultados de una enumeración
```

### El modo pasivo de Amass

El modo pasivo es el punto de partida. Amass consulta fuentes externas sin enviar ningún paquete al objetivo. La opción `-passive` lo fuerza a modo pasivo puro:

```bash
# Modo pasivo — sin contacto con el objetivo
# Solo consulta fuentes externas: CT logs, VirusTotal, Shodan, etc.
amass enum -passive -d example.com

# Guardar resultado en un archivo
amass enum -passive -d example.com -o subdominios_pasivo.txt

# Mostrar la fuente de cada subdominio encontrado
# Muy útil para entender qué fuentes son más productivas para este objetivo
amass enum -passive -d example.com -src

# Output con la fuente:
# [CertSpotter]     www.example.com
# [VirusTotal]      api.example.com
# [crt.sh]          staging.example.com
# [SecurityTrails]  dev-internal.example.com

# Guardar en formato JSON para procesamiento posterior
amass enum -passive -d example.com -json resultados.json

# Múltiples dominios
amass enum -passive -d example.com -d example.net -o todos.txt
```

### El modo activo de Amass

El modo activo va más allá: hace resolución DNS directa, intenta transferencias de zona, y puede hacer fuerza bruta de subdominios. Esto genera tráfico hacia el objetivo y puede ser detectado, pero descubre subdominios que las fuentes pasivas no conocen.

```bash
# Modo activo (es el modo por defecto sin -passive)
# Combina fuentes pasivas con resolución DNS activa
amass enum -d example.com

# Especificar resolvers DNS personalizados
# Útil para evitar rate limiting en los DNS públicos
# o para usar resolvers de alta velocidad
amass enum -d example.com -r 8.8.8.8,1.1.1.1,9.9.9.9

# Desde un archivo con lista de resolvers
amass enum -d example.com -rf /usr/share/wordlists/resolvers.txt

# Fuerza bruta de subdominios con wordlist
# Genera muchas peticiones DNS — usar con precaución en engagements reales
amass enum -brute -d example.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# Fuerza bruta recursiva — cuando encuentra un subdominio válido,
# intenta hacer fuerza bruta en ese subdominio también
# Por ejemplo: si encuentra api.example.com, intenta v1.api.example.com, v2.api.example.com...
amass enum -brute -d example.com -w wordlist.txt -rf resolvers.txt

# Alteraciones — Amass puede mutar subdominios conocidos para descubrir variantes
# Por ejemplo, de dev.example.com genera dev1, dev2, dev-new, dev-old, etc.
amass enum -d example.com -alts
```

### Configuración avanzada de Amass

Amass tiene un archivo de configuración YAML que permite configurar fuentes de datos, API keys, resolvers y comportamiento:

```bash
# Crear el directorio de configuración si no existe
mkdir -p ~/.config/amass

# El archivo de configuración principal
cat > ~/.config/amass/config.yaml << 'CONFIG'
# Configuración de Amass
# Referencia: https://github.com/owasp-amass/amass/blob/master/examples/config.yaml

# Resolvers DNS de alta velocidad y confianza
resolvers:
  - 8.8.8.8       # Google
  - 8.8.4.4       # Google secundario
  - 1.1.1.1       # Cloudflare
  - 1.0.0.1       # Cloudflare secundario
  - 9.9.9.9       # Quad9
  - 64.6.64.6     # Verisign

# Límite de peticiones DNS por segundo (por defecto: sin límite)
# Reducir en engagements donde la discreción es importante
max_dns_queries: 500

# Fuentes de datos y sus API keys
data_sources:
  - name: AlienVault
    ttl: 4320

  - name: Censys
    ttl: 4320
    creds:
      key: TU_CENSYS_API_ID
      secret: TU_CENSYS_API_SECRET

  - name: GitHub
    ttl: 4320
    creds:
      token: TU_GITHUB_TOKEN

  - name: SecurityTrails
    ttl: 4320
    creds:
      key: TU_SECURITYTRAILS_KEY

  - name: Shodan
    ttl: 4320
    creds:
      key: TU_SHODAN_KEY

  - name: VirusTotal
    ttl: 4320
    creds:
      key: TU_VIRUSTOTAL_KEY
CONFIG

# Usar la configuración en una enumeración
amass enum -config ~/.config/amass/config.yaml -d example.com
```

### Intel — Descubrimiento de ASN y rangos IP

El subcomando `intel` de Amass es una joya poco conocida. En lugar de partir de un dominio, parte del nombre de una organización y descubre qué bloques de IPs y ASNs están registrados a su nombre. Esto es especialmente útil cuando el scope dice "toda la infraestructura de Example Corp" porque puede revelar IPs y dominios que nunca encontrarías partiendo solo del dominio web.

```bash
# Buscar ASNs registrados a nombre de una organización
# Amass consulta los registros de ARIN, RIPE, APNIC, etc.
amass intel -org "Example Corp"

# Output:
# AS12345 - EXAMPLE-CORP-ASN
#   93.184.216.0/24
#   93.184.217.0/24

# Una vez tienes el ASN, buscar todos los dominios que Amass conoce para ese ASN
amass intel -asn 12345

# Una vez tienes rangos de IPs, hacer reverse DNS en todos ellos
# Para descubrir dominios alojados en esos rangos
amass intel -cidr 93.184.216.0/24

# Combinar: partir del nombre de empresa y llegar a todos sus dominios
amass intel -org "Example Corp" -whois
```

---

## 3. Combinando Subfinder y Amass — El flujo profesional

En la práctica real, Subfinder y Amass no se usan de forma excluyente sino complementaria. Cada herramienta tiene acceso a fuentes distintas y usa técnicas distintas, por lo que la unión de ambas da una cobertura mucho mayor que cualquiera de ellas por separado.

El flujo profesional estándar es este:

```bash
TARGET="example.com"
OUTPUT_DIR="./recon_${TARGET}"
mkdir -p "$OUTPUT_DIR"

echo "[*] Fase 1: Subfinder — fuentes pasivas rápidas"
subfinder -d "$TARGET" -all -silent -o "$OUTPUT_DIR/subfinder.txt"
echo "[+] Subfinder: $(wc -l < $OUTPUT_DIR/subfinder.txt) subdominios"

echo "[*] Fase 2: Amass pasivo — fuentes pasivas exhaustivas"
amass enum -passive -d "$TARGET" -o "$OUTPUT_DIR/amass_passive.txt" 2>/dev/null
echo "[+] Amass pasivo: $(wc -l < $OUTPUT_DIR/amass_passive.txt) subdominios"

echo "[*] Fase 3: CT logs directos — crt.sh sin intermediarios"
curl -s "https://crt.sh/?q=%.${TARGET}&output=json" 2>/dev/null \
    | jq -r '.[].name_value' \
    | sed 's/\*\.//g' \
    | grep "\.${TARGET}$" \
    | sort -u > "$OUTPUT_DIR/crtsh.txt"
echo "[+] crt.sh: $(wc -l < $OUTPUT_DIR/crtsh.txt) subdominios"

echo "[*] Combinando y deduplicando todas las fuentes..."
cat "$OUTPUT_DIR/subfinder.txt" \
    "$OUTPUT_DIR/amass_passive.txt" \
    "$OUTPUT_DIR/crtsh.txt" \
    | sort -u > "$OUTPUT_DIR/all_subdomains.txt"

TOTAL=$(wc -l < "$OUTPUT_DIR/all_subdomains.txt")
echo "[+] Total subdominios únicos: $TOTAL"
echo "[+] Resultados en: $OUTPUT_DIR/all_subdomains.txt"
```

### Verificación de subdominios activos

Tener una lista de subdominios no es suficiente — muchos pueden estar dados de baja, apuntar a IPs inexistentes, o ser errores. El siguiente paso es verificar cuáles están realmente activos:

```bash
# dnsx — resolución DNS masiva de alta velocidad (también de ProjectDiscovery)
go install -v github.com/projectdiscovery/dnsx/cmd/dnsx@latest

# Resolver todos los subdominios y quedarse solo con los que tienen respuesta DNS
# -silent elimina el banner, -a solicita registros A (IPv4)
cat all_subdomains.txt | dnsx -silent -a -o subdominios_con_ip.txt

# Ver también los registros CNAME — revelan servicios de terceros
# Un CNAME a s3.amazonaws.com puede indicar un bucket S3 apuntable
cat all_subdomains.txt | dnsx -silent -a -cname -o subdominios_dns_completo.txt

# Filtrar solo los subdominios que resuelven a IPs privadas (infraestructura interna)
cat subdominios_con_ip.txt | grep -E "10\.|172\.(1[6-9]|2[0-9]|3[01])\.|192\.168\."
```

### Identificar subdominios apuntables — Subdomain Takeover

Un subdominio apuntable (subdomain takeover) ocurre cuando un subdominio apunta mediante CNAME a un servicio externo que ya no existe o cuya cuenta fue cancelada. Si el servicio permite registrar ese nombre, un atacante puede tomar control del subdominio.

```bash
# Instalar subjack para detectar subdominios apuntables
go install github.com/haccer/subjack@latest

# Buscar subdomain takeovers en la lista de subdominios
subjack -w all_subdomains.txt -t 100 -timeout 30 -o takeovers.txt -ssl

# Nuclei también tiene templates para subdomain takeover
nuclei -l all_subdomains.txt -t ~/nuclei-templates/takeovers/ -o nuclei_takeovers.txt

# CNAMEs más comunes que pueden ser apuntables:
# *.s3.amazonaws.com      → bucket S3 eliminado
# *.github.io             → repositorio de GitHub borrado
# *.herokuapp.com         → app de Heroku eliminada
# *.azurewebsites.net     → app de Azure eliminada
# *.netlify.app           → sitio de Netlify eliminado
# *.pantheonsite.io       → sitio de Pantheon eliminado
```

---

## 4. Fuerza bruta inteligente de subdominios

La enumeración pasiva solo encuentra subdominios que ya están indexados en alguna fuente pública. Para descubrir subdominios que nunca han sido indexados — entornos de desarrollo internos, APIs privadas, paneles de administración — necesitamos fuerza bruta DNS.

### La wordlist correcta marca la diferencia

La calidad de la wordlist determina directamente cuántos subdominios nuevos encuentras. SecLists tiene las mejores wordlists específicamente para subdominios:

```bash
# Instalar SecLists si no lo tienes
sudo apt install seclists
# O clonar el repositorio completo
git clone https://github.com/danielmiessler/SecLists.git /usr/share/seclists

# Las wordlists más usadas para subdominios, ordenadas de más ligera a más pesada:
ls /usr/share/seclists/Discovery/DNS/
# subdomains-top1million-5000.txt      → 5.000 entradas — escaneo rápido
# subdomains-top1million-20000.txt     → 20.000 entradas — equilibrio calidad/velocidad
# subdomains-top1million-110000.txt    → 110.000 entradas — cobertura alta
# bitquark-subdomains-top100000.txt    → basada en datos reales de internet
# dns-Jhaddix.txt                      → de Jason Haddix, muy usada en bug bounty
```

### Fuerza bruta con puredns

puredns es actualmente la herramienta más recomendada para fuerza bruta DNS porque maneja correctamente los wildcards y los falsos positivos, un problema que arruina los resultados de otras herramientas:

```bash
# Instalar puredns
go install github.com/d3mondev/puredns/v2@latest

# También necesitas massdns para la resolución masiva
sudo apt install massdns

# Descargar una lista de resolvers públicos de confianza
# Crucial para hacer fuerza bruta a alta velocidad sin sobrecargar un solo resolver
wget https://raw.githubusercontent.com/trickest/resolvers/main/resolvers.txt

# Fuerza bruta básica
# puredns resuelve cada entrada de la wordlist como subdominio del objetivo
puredns bruteforce /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
    example.com \
    -r resolvers.txt \
    -o puredns_resultados.txt

# Verificar una lista existente de subdominios (los ya encontrados)
# Útil para resolver rápidamente la lista combinada de Subfinder + Amass
puredns resolve all_subdomains.txt -r resolvers.txt -o subdominios_resueltos.txt
```

### El problema de los wildcards

Un wildcard DNS es cuando el dominio tiene configurado `*.example.com → alguna IP`. Esto significa que cualquier subdominio que no exista responde igualmente con una IP, lo que hace que la fuerza bruta genere miles de falsos positivos — todos los subdominios que pruebas parecen existir.

Identificar y manejar correctamente los wildcards es crítico:

```bash
# Detectar si hay wildcard en el dominio objetivo
# Si un subdominio aleatorio (que definitivamente no existe) resuelve a una IP,
# hay un wildcard configurado
dig nonexistent-random-subdomain-xyz.example.com

# Si la respuesta devuelve una IP → hay wildcard
# Si la respuesta es NXDOMAIN → no hay wildcard

# puredns maneja wildcards automáticamente
# Detecta el patrón del wildcard y filtra los falsos positivos
puredns bruteforce wordlist.txt example.com -r resolvers.txt --wildcard-tests 10

# Gobuster también tiene manejo de wildcards
gobuster dns -d example.com -w wordlist.txt --wildcard

# massdns: resolución DNS masiva pura (sin manejo de wildcards)
# Usar solo cuando no hay wildcard, o combinado con un filtro posterior
massdns -r resolvers.txt -t A -o S -w massdns_output.txt subdomains_to_resolve.txt
```

---

## 5. Certificate Transparency en profundidad

Mencionamos crt.sh como fuente pasiva, pero merece una sección propia porque es una de las fuentes más ricas y menos explotadas de subdominios. Los Certificate Transparency logs son registros públicos donde se registra cada certificado SSL/TLS emitido, incluyendo los Subject Alternative Names que pueden contener docenas de dominios adicionales por certificado.

### Consultar crt.sh directamente

```bash
# Consulta básica con la API JSON de crt.sh
curl -s "https://crt.sh/?q=%.example.com&output=json" \
    | jq -r '.[].name_value' \
    | sed 's/\*\.//g' \
    | sort -u

# Versión más robusta con manejo de errores y deduplicación completa
curl -s --retry 3 "https://crt.sh/?q=%.example.com&output=json" \
    | python3 -c "
import sys, json

try:
    data = json.load(sys.stdin)
except json.JSONDecodeError:
    sys.exit(1)

subdomains = set()
for cert in data:
    # name_value puede tener múltiples dominios separados por newlines
    for name in cert['name_value'].split('\n'):
        name = name.strip()
        # Eliminar wildcards
        name = name.lstrip('*.')
        # Solo subdominios del dominio objetivo
        if 'example.com' in name:
            subdomains.add(name)

for s in sorted(subdomains):
    print(s)
" > crtsh_subdominios.txt

# Consultar también los certificados de organizaciones relacionadas
# A veces las filiales usan el mismo certificado que la empresa principal
curl -s "https://crt.sh/?q=Example+Corp&output=json" \
    | jq -r '.[].name_value' \
    | sort -u
```

### Otras fuentes de CT logs

crt.sh no es la única fuente de Certificate Transparency. Estas son las alternativas más completas:

```bash
# Censys — búsqueda en certificados con más filtros
# https://search.censys.io/certificates
# Consulta via API:
curl -s --user "API_ID:API_SECRET" \
    "https://search.censys.io/api/v1/search/certificates?q=parsed.names%3A+example.com&fields=parsed.names" \
    | jq -r '.results[].parsed.names[]' | sort -u

# Facebook CT logs (han indexado más certificados que crt.sh en algunos casos)
curl -s "https://graph.facebook.com/certificates?query=example.com&fields=domains&access_token=TOKEN" \
    | jq -r '.data[].domains[]' | sort -u

# Google CT logs via Transparencyreport
# https://transparencyreport.google.com/https/certificates
```

---

## 6. Automatización completa — Script profesional de enumeración

Este script integra todas las técnicas anteriores en un flujo reproducible y bien documentado. Lo uso como punto de partida en todos los engagements:

```bash
#!/bin/bash
# subdomain_enum.sh — Enumeración exhaustiva de subdominios
# Combina Subfinder, Amass, crt.sh, fuerza bruta y verificación
# Uso: ./subdomain_enum.sh example.com
# Requiere: subfinder, amass, puredns, dnsx, jq, curl

TARGET=$1
TIMESTAMP=$(date +%Y%m%d_%H%M)
OUTPUT_DIR="./recon_${TARGET}_${TIMESTAMP}"
RESOLVERS="/usr/share/seclists/Miscellaneous/dns-resolvers.txt"
WORDLIST="/usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt"

# ── VALIDACIÓN ────────────────────────────────────────────────────────
if [ -z "$TARGET" ]; then
    echo "[-] Error: especifica un dominio"
    echo "    Uso: $0 example.com"
    exit 1
fi

mkdir -p "$OUTPUT_DIR"/{passive,active,resolved,reports}

echo "╔══════════════════════════════════════════════╗"
echo "  Enumeración de subdominios: $TARGET"
echo "  Output: $OUTPUT_DIR"
echo "╚══════════════════════════════════════════════╝"

# ── FASE 1: ENUMERACIÓN PASIVA ────────────────────────────────────────
echo ""
echo "[1/5] Enumeración pasiva — sin contacto con el objetivo"

# Subfinder: rápido, muchas fuentes
echo "  [+] Subfinder..."
subfinder -d "$TARGET" -all -silent -o "$OUTPUT_DIR/passive/subfinder.txt" 2>/dev/null
echo "      Resultado: $(wc -l < $OUTPUT_DIR/passive/subfinder.txt) subdominios"

# Amass: exhaustivo, más lento
echo "  [+] Amass pasivo..."
amass enum -passive -d "$TARGET" -o "$OUTPUT_DIR/passive/amass.txt" 2>/dev/null
echo "      Resultado: $(wc -l < $OUTPUT_DIR/passive/amass.txt) subdominios"

# crt.sh: CT logs directos
echo "  [+] crt.sh (Certificate Transparency)..."
curl -s --retry 3 "https://crt.sh/?q=%.${TARGET}&output=json" 2>/dev/null \
    | jq -r '.[].name_value' \
    | sed 's/\*\.//g' \
    | grep "\.${TARGET}$" \
    | sort -u > "$OUTPUT_DIR/passive/crtsh.txt"
echo "      Resultado: $(wc -l < $OUTPUT_DIR/passive/crtsh.txt) subdominios"

# Combinar fuentes pasivas
cat "$OUTPUT_DIR/passive/"*.txt | sort -u > "$OUTPUT_DIR/passive/all_passive.txt"
echo "  [*] Total pasivo único: $(wc -l < $OUTPUT_DIR/passive/all_passive.txt)"

# ── FASE 2: VERIFICACIÓN DNS ──────────────────────────────────────────
echo ""
echo "[2/5] Verificando subdominios activos con resolución DNS..."
if command -v dnsx &> /dev/null; then
    cat "$OUTPUT_DIR/passive/all_passive.txt" \
        | dnsx -silent -a -resp -o "$OUTPUT_DIR/resolved/passive_resolved.txt" 2>/dev/null
    echo "  [+] Subdominios activos: $(wc -l < $OUTPUT_DIR/resolved/passive_resolved.txt)"
else
    echo "  [-] dnsx no instalado, copiando lista sin verificar"
    cp "$OUTPUT_DIR/passive/all_passive.txt" "$OUTPUT_DIR/resolved/passive_resolved.txt"
fi

# ── FASE 3: DETECCIÓN DE WILDCARD ────────────────────────────────────
echo ""
echo "[3/5] Detectando wildcards DNS..."
RANDOM_SUB="nonexistent-$(openssl rand -hex 8).${TARGET}"
WILDCARD_TEST=$(dig +short "$RANDOM_SUB" A)

if [ -n "$WILDCARD_TEST" ]; then
    echo "  [!] WILDCARD DETECTADO: *.${TARGET} → ${WILDCARD_TEST}"
    echo "  [!] La fuerza bruta puede generar falsos positivos"
    echo "  [*] Usando puredns con detección automática de wildcards..."
    WILDCARD_FLAG="--wildcard-tests 10"
else
    echo "  [+] Sin wildcard detectado — fuerza bruta limpia"
    WILDCARD_FLAG=""
fi

# ── FASE 4: FUERZA BRUTA DNS ─────────────────────────────────────────
echo ""
echo "[4/5] Fuerza bruta de subdominios..."
if command -v puredns &> /dev/null && [ -f "$RESOLVERS" ]; then
    puredns bruteforce "$WORDLIST" "$TARGET" \
        -r "$RESOLVERS" \
        $WILDCARD_FLAG \
        -o "$OUTPUT_DIR/active/puredns_bruteforce.txt" \
        --quiet 2>/dev/null
    echo "  [+] Fuerza bruta: $(wc -l < $OUTPUT_DIR/active/puredns_bruteforce.txt) nuevos"
else
    echo "  [-] puredns o resolvers no disponibles, omitiendo fuerza bruta"
    touch "$OUTPUT_DIR/active/puredns_bruteforce.txt"
fi

# ── FASE 5: CONSOLIDACIÓN FINAL ──────────────────────────────────────
echo ""
echo "[5/5] Consolidando todos los resultados..."

# Combinar pasivo + fuerza bruta
cat "$OUTPUT_DIR/passive/all_passive.txt" \
    "$OUTPUT_DIR/active/puredns_bruteforce.txt" \
    | sort -u > "$OUTPUT_DIR/reports/all_subdomains_final.txt"

TOTAL_FINAL=$(wc -l < "$OUTPUT_DIR/reports/all_subdomains_final.txt")

# Extraer IPs únicas encontradas
grep -oE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b" \
    "$OUTPUT_DIR/resolved/passive_resolved.txt" \
    | sort -u > "$OUTPUT_DIR/reports/ips_found.txt" 2>/dev/null

# ── RESUMEN FINAL ─────────────────────────────────────────────────────
echo ""
echo "╔══════════════════════════════════════════════╗"
echo "  ENUMERACIÓN COMPLETADA"
echo "╠══════════════════════════════════════════════╣"
printf "  %-25s %s\n" "Subfinder:" "$(wc -l < $OUTPUT_DIR/passive/subfinder.txt)"
printf "  %-25s %s\n" "Amass:" "$(wc -l < $OUTPUT_DIR/passive/amass.txt)"
printf "  %-25s %s\n" "crt.sh:" "$(wc -l < $OUTPUT_DIR/passive/crtsh.txt)"
printf "  %-25s %s\n" "Fuerza bruta:" "$(wc -l < $OUTPUT_DIR/active/puredns_bruteforce.txt)"
printf "  %-25s %s\n" "TOTAL ÚNICO FINAL:" "$TOTAL_FINAL"
printf "  %-25s %s\n" "IPs únicas:" "$(wc -l < $OUTPUT_DIR/reports/ips_found.txt 2>/dev/null || echo 0)"
echo "╠══════════════════════════════════════════════╣"
echo "  Resultados en: $OUTPUT_DIR/reports/"
echo "╚══════════════════════════════════════════════╝"
```

---

## 7. Integración con el resto del pipeline de reconocimiento

La lista de subdominios es solo el primer paso. Lo que haces con ella después determina el valor real del reconocimiento. Aquí están las integraciones más habituales:

### Descubrimiento de servicios web activos

```bash
# httpx — verificar qué subdominios tienen servicios HTTP/HTTPS
go install -v github.com/projectdiscovery/httpx/cmd/httpx@latest

# Verificar todos los subdominios en puertos web comunes
# -title muestra el título de la página
# -status-code muestra el código HTTP
# -tech-detect detecta las tecnologías del sitio
cat all_subdomains.txt | httpx -silent \
    -title \
    -status-code \
    -tech-detect \
    -o servicios_web.txt

# Ver solo los que devuelven 200 OK
cat servicios_web.txt | grep "\[200\]"

# Ver solo los que tienen paneles de administración por el título
cat servicios_web.txt | grep -i "admin\|login\|dashboard\|panel"
```

### Captura de pantallas de todos los servicios web

```bash
# gowitness — captura screenshots de URLs de forma masiva
go install github.com/sensepost/gowitness@latest

# Capturar screenshots de todos los servicios web encontrados
cat servicios_web.txt | awk '{print $1}' | gowitness scan file -f - --screenshot-path ./screenshots/

# Ver el reporte HTML con todos los screenshots
gowitness report serve --screenshot-path ./screenshots/
# Abre http://localhost:7171
```

### Escaneo de vulnerabilidades con Nuclei

```bash
# Nuclei — escaneo de vulnerabilidades basado en templates
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest

# Actualizar los templates a la última versión
nuclei -update-templates

# Escanear todos los subdominios activos buscando vulnerabilidades comunes
cat servicios_web.txt | awk '{print $1}' | nuclei \
    -t ~/nuclei-templates/ \
    -severity medium,high,critical \
    -o vulnerabilidades_nuclei.txt

# Solo templates de exposición de información sensible (menos ruidoso)
cat servicios_web.txt | awk '{print $1}' | nuclei \
    -t ~/nuclei-templates/exposures/ \
    -t ~/nuclei-templates/misconfiguration/ \
    -o exposiciones.txt
```

---

## 8. Cheatsheet de referencia rápida

```bash
# ── SUBFINDER ────────────────────────────────────────────────────────────
subfinder -d ejemplo.com                          # Básico
subfinder -d ejemplo.com -all -o subs.txt         # Todas las fuentes
subfinder -d ejemplo.com -all -v                  # Con fuentes visibles
subfinder -dL dominios.txt -all -o todos.txt      # Múltiples dominios
subfinder -d ejemplo.com -t 50 -o subs.txt        # 50 threads

# ── AMASS ────────────────────────────────────────────────────────────────
amass enum -passive -d ejemplo.com                # Solo pasivo
amass enum -d ejemplo.com -src                    # Mostrar fuentes
amass enum -d ejemplo.com -o subs.txt             # Guardar resultado
amass enum -brute -d ejemplo.com -w wordlist.txt  # Con fuerza bruta
amass intel -org "Empresa"                        # Descubrir ASN/IPs
amass intel -asn 12345                            # Dominios de un ASN
amass intel -cidr 93.184.216.0/24                 # Reverse DNS en rango

# ── VERIFICACIÓN DNS ─────────────────────────────────────────────────────
cat subs.txt | dnsx -silent -a -o activos.txt     # Resolver IPs
cat subs.txt | dnsx -silent -cname -o cnames.txt  # Ver CNAMEs

# ── FUERZA BRUTA ─────────────────────────────────────────────────────────
puredns bruteforce wordlist.txt ejemplo.com -r resolvers.txt -o brute.txt

# ── DETECCIÓN WILDCARD ───────────────────────────────────────────────────
dig +short "random-xyz.ejemplo.com" A             # Si responde → wildcard

# ── SERVICIOS WEB ────────────────────────────────────────────────────────
cat subs.txt | httpx -silent -title -status-code -tech-detect
cat urls.txt | gowitness scan file -f - --screenshot-path ./shots/

# ── SUBDOMAIN TAKEOVER ───────────────────────────────────────────────────
subjack -w subs.txt -t 100 -o takeovers.txt -ssl

# ── CT LOGS ──────────────────────────────────────────────────────────────
curl -s "https://crt.sh/?q=%.ejemplo.com&output=json" \
  | jq -r '.[].name_value' | sed 's/\*\.//g' | sort -u

# ── FLUJO COMPLETO (one-liner) ────────────────────────────────────────────
subfinder -d ejemplo.com -all -silent | \
  dnsx -silent -a | \
  httpx -silent -title -status-code | \
  tee servicios_web.txt

# ── WORDLISTS MÁS USADAS ─────────────────────────────────────────────────
/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
/usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt
/usr/share/seclists/Discovery/DNS/dns-Jhaddix.txt
/usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt

# ── RECURSOS Y HERRAMIENTAS DEL ECOSISTEMA ───────────────────────────────
https://github.com/projectdiscovery/subfinder     → Subfinder
https://github.com/owasp-amass/amass              → Amass
https://github.com/d3mondev/puredns               → puredns (brute)
https://github.com/projectdiscovery/dnsx          → dnsx (resolver)
https://github.com/projectdiscovery/httpx         → httpx (web check)
https://github.com/projectdiscovery/nuclei        → Nuclei (vulns)
https://github.com/sensepost/gowitness            → Screenshots
https://github.com/haccer/subjack                 → Takeover detection
https://crt.sh                                    → CT logs
https://chaos.projectdiscovery.io                 → Dataset subdominios
https://github.com/danielmiessler/SecLists        → Wordlists
```
