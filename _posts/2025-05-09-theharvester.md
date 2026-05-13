---
layout: post
title: "theHarvester — Recolección de emails, subdominios y metadatos"
date: 2025-05-09
categories: [reconocimiento]
tags: [theHarvester, osint, emails, subdominios, reconocimiento, recon, hacking]
description: "Guía completa de theHarvester: instalación, fuentes, uso avanzado, integración con otras herramientas y automatización en reconocimiento profesional."
---

## ¿Qué es theHarvester?

theHarvester es una de las herramientas de reconocimiento pasivo más utilizadas en pentesting profesional. Recolecta automáticamente información de fuentes públicas como motores de búsqueda, bases de datos de certificados, APIs y servicios de inteligencia de amenazas.

Está desarrollada en Python, incluida por defecto en Kali Linux y Parrot OS, y es parte del arsenal estándar de cualquier pentest o ejercicio de Red Team en la fase de reconocimiento.

**Qué recolecta:**
- Emails corporativos
- Nombres de empleados
- Subdominios y hosts
- IPs y rangos de red
- URLs expuestas
- Puertos abiertos (cuando usa Shodan)
- Información de LinkedIn

---

## 1. Instalación y configuración

### En Kali Linux / Parrot OS

```bash
# Ya viene instalado — verificar versión
theHarvester --version

# Actualizar
cd /usr/lib/python3/dist-packages/theHarvester
sudo git pull

# O con pip
pip3 install theHarvester --upgrade
```

### Instalación manual

```bash
# Clonar repositorio
git clone https://github.com/laramies/theHarvester
cd theHarvester

# Instalar dependencias
pip3 install -r requirements/base.txt

# Ejecutar
python3 theHarvester.py -h
```

### Configuración de API keys

Para sacar el máximo partido a theHarvester necesitas configurar API keys de los servicios que las requieren. El archivo de configuración está en:

```bash
# Ubicación del archivo de configuración
cat /etc/theHarvester/api-keys.yaml
# O en instalación manual:
cat theHarvester/etc/api-keys.yaml
```

```yaml
# api-keys.yaml — configurar las que tengas disponibles
apikeys:
  bevigil:
    key: TU_BEVIGIL_KEY           # Gratuito en bevigil.com
  bing:
    key: TU_BING_KEY              # Azure Cognitive Services
  binaryedge:
    key: TU_BINARYEDGE_KEY        # Plan gratuito disponible
  bufferoverun:
    key: TU_BUFFEROVERUN_KEY      # Gratuito
  censys:
    id: TU_CENSYS_ID
    secret: TU_CENSYS_SECRET      # Gratuito en censys.io
  fullhunt:
    key: TU_FULLHUNT_KEY          # Gratuito
  github:
    key: TU_GITHUB_TOKEN          # Token personal de GitHub
  hunter:
    key: TU_HUNTER_KEY            # Gratuito en hunter.io
  intelx:
    key: TU_INTELX_KEY            # Gratuito limitado
  pentest-tools:
    key: TU_PT_KEY
  securityTrails:
    key: TU_ST_KEY                # Gratuito en securitytrails.com
  shodan:
    key: TU_SHODAN_KEY            # Gratuito en shodan.io
  virustotal:
    key: TU_VT_KEY                # Gratuito en virustotal.com
  zoomeye:
    key: TU_ZOOMEYE_KEY
```

> Con solo las keys gratuitas de Shodan, Hunter, SecurityTrails y VirusTotal ya multiplicas la efectividad enormemente.

---

## 2. Sintaxis y opciones

```bash
theHarvester -d <dominio> -b <fuentes> [opciones]
```

### Opciones principales

| Opción | Descripción |
|--------|-------------|
| `-d` | Dominio objetivo (obligatorio) |
| `-b` | Fuentes de datos separadas por coma |
| `-l` | Límite de resultados por fuente |
| `-f` | Guardar resultados en archivo (sin extensión) |
| `-v` | Verificar hosts con DNS |
| `-n` | Realizar búsqueda DNS |
| `-c` | Fuerza bruta de subdominios con DNS |
| `-p` | Escanear puertos de los hosts encontrados |
| `-s` | Inicio de la búsqueda (paginación) |
| `-e` | Servidor DNS a usar |
| `-t` | Habilitar TLD expansion |
| `--screenshot` | Capturar screenshot de hosts web |

---

## 3. Fuentes de datos disponibles

theHarvester soporta múltiples fuentes. Algunas no requieren API key, otras sí.

### Fuentes sin API key

```bash
# Motores de búsqueda
-b google          # Google Search
-b bing            # Microsoft Bing
-b yahoo           # Yahoo Search
-b duckduckgo      # DuckDuckGo
-b baidu           # Baidu (útil para objetivos en China)
-b ask             # Ask.com

# Bases de datos de certificados y DNS
-b anubis          # AnubisDB — subdominios de CT logs
-b dnsdumpster     # DNSdumpster.com
-b crtsh           # crt.sh — Certificate Transparency
-b hackertarget    # HackerTarget.com
-b rapiddns        # RapidDNS
-b sublist3r       # Sublist3r API
-b threatminer     # ThreatMiner.org
-b urlscan         # URLScan.io

# Redes sociales (sin key)
-b linkedin        # LinkedIn (resultados básicos)
-b twitter         # Twitter/X
```

### Fuentes con API key (gratuitas)

```bash
-b bevigil         # Inteligencia de apps móviles
-b bufferoverun    # DNS pasivo
-b fullhunt        # Descubrimiento de activos
-b github-code     # Búsqueda en código GitHub
-b hunter          # Hunter.io — emails corporativos
-b intelx          # Intelligence X
-b securityTrails  # Historial DNS y subdominios
-b shodan          # Dispositivos y servicios
-b virustotal      # VirusTotal subdominios
-b zoomeye         # Buscador de dispositivos chino
```

### Fuentes con API de pago

```bash
-b bing_api        # Bing API (Azure)
-b binaryedge      # BinaryEdge
-b censys          # Censys (plan básico gratuito)
-b pentest-tools   # Pentest-Tools.com
```

---

## 4. Uso práctico por escenario

### Reconocimiento inicial rápido

```bash
# Búsqueda rápida sin API keys — fuentes gratuitas
theHarvester -d example.com -b google,bing,duckduckgo -l 200

# Con más fuentes y límite mayor
theHarvester -d example.com -b google,bing,yahoo,duckduckgo,crtsh,anubis -l 500
```

### Reconocimiento completo con todas las fuentes

```bash
# Todas las fuentes disponibles
theHarvester -d example.com -b all -l 500

# Guardar resultados en XML y JSON
theHarvester -d example.com -b all -l 500 -f example_recon
# Genera: example_recon.xml y example_recon.json
```

### Enfoque en emails

```bash
# Fuentes especializadas en emails
theHarvester -d example.com -b google,bing,hunter,linkedin -l 300

# Con verificación DNS de hosts encontrados
theHarvester -d example.com -b google,hunter -v
```

### Enfoque en subdominios

```bash
# Fuentes especializadas en subdominios
theHarvester -d example.com -b crtsh,anubis,dnsdumpster,hackertarget,bufferoverun,securityTrails,virustotal -l 500

# Con resolución DNS para verificar cuáles están activos
theHarvester -d example.com -b crtsh,anubis,virustotal -v -n
```

### Fuerza bruta de subdominios (semi-activo)

```bash
# Combinar descubrimiento pasivo con brute force DNS
theHarvester -d example.com -b google,crtsh -c

# El flag -c usa el diccionario incluido en theHarvester
# Localización: /usr/lib/python3/dist-packages/theHarvester/discovery/
```

### Con escaneo de puertos

```bash
# Escanear puertos en los hosts descubiertos
theHarvester -d example.com -b google,shodan -p
```

### Con capturas de pantalla

```bash
# Capturar screenshots de servicios web encontrados
theHarvester -d example.com -b google,crtsh --screenshot /tmp/screenshots/
```

---

## 5. Análisis de resultados

### Formato de output en consola

```
[*] Emails found: 8
------------------
admin@example.com
info@example.com
j.smith@example.com
m.garcia@example.com
carlos.lopez@example.com
security@example.com
noreply@example.com
webmaster@example.com

[*] Hosts found: 24
-------------------
api.example.com:93.184.216.1
blog.example.com:93.184.216.2
cdn.example.com:104.16.132.229
dev.example.com:93.184.216.100
mail.example.com:93.184.216.5
staging.example.com:93.184.216.101
vpn.example.com:93.184.216.10
www.example.com:93.184.216.34

[*] IPs found: 6
-------------------
93.184.216.1
93.184.216.2
93.184.216.5
93.184.216.10
93.184.216.34
93.184.216.100
```

### Parsear el JSON de resultados

```python
import json

with open("example_recon.json") as f:
    data = json.load(f)

# Emails encontrados
print("=== EMAILS ===")
for email in data.get("emails", []):
    print(f"  {email}")

# Hosts y subdominios
print("\n=== HOSTS ===")
for host in data.get("hosts", []):
    print(f"  {host}")

# IPs
print("\n=== IPs ===")
for ip in data.get("ips", []):
    print(f"  {ip}")

# Estadísticas
print(f"\nTotal emails: {len(data.get('emails', []))}")
print(f"Total hosts: {len(data.get('hosts', []))}")
print(f"Total IPs: {len(data.get('ips', []))}")
```

### Extraer y analizar emails

```python
import json
import re
from collections import Counter

with open("example_recon.json") as f:
    data = json.load(f)

emails = data.get("emails", [])

# Identificar patrón de emails
def analyze_email_pattern(emails):
    """Analiza el patrón de nomenclatura de emails corporativos"""
    patterns = Counter()

    for email in emails:
        local = email.split("@")[0]

        if re.match(r'^[a-z]+\.[a-z]+$', local):
            patterns["firstname.lastname"] += 1
        elif re.match(r'^[a-z][a-z]+$', local) and len(local) > 3:
            patterns["firstinitial+lastname"] += 1
        elif re.match(r'^[a-z]+_[a-z]+$', local):
            patterns["firstname_lastname"] += 1
        elif re.match(r'^[a-z]+[0-9]*$', local):
            patterns["firstname_only"] += 1
        else:
            patterns["other"] += 1

    return patterns.most_common()

patterns = analyze_email_pattern(emails)
print("Patrones de email detectados:")
for pattern, count in patterns:
    print(f"  {pattern}: {count} ejemplos")

# Generar emails potenciales desde lista de nombres
def generate_emails(names, domain, pattern):
    """Genera emails según el patrón identificado"""
    generated = []
    for name in names:
        parts = name.lower().split()
        if len(parts) < 2:
            continue
        first, last = parts[0], parts[-1]

        if pattern == "firstname.lastname":
            generated.append(f"{first}.{last}@{domain}")
        elif pattern == "firstinitial+lastname":
            generated.append(f"{first[0]}{last}@{domain}")
        elif pattern == "firstname_lastname":
            generated.append(f"{first}_{last}@{domain}")

    return generated
```

---

## 6. Automatización y scripting

### Script de reconocimiento automatizado

```bash
#!/bin/bash
# harvester_recon.sh — Reconocimiento automatizado con theHarvester

TARGET=$1
OUTPUT_DIR="harvest_${TARGET}_$(date +%Y%m%d)"

if [ -z "$TARGET" ]; then
    echo "Uso: ./harvester_recon.sh dominio.com"
    exit 1
fi

mkdir -p "$OUTPUT_DIR"

echo "[*] Iniciando reconocimiento de: $TARGET"
echo "[*] Output: $OUTPUT_DIR/"

# Fase 1: Fuentes rápidas sin API (motores de búsqueda)
echo "[+] Fase 1: Motores de búsqueda..."
theHarvester -d "$TARGET" \
    -b google,bing,duckduckgo,yahoo \
    -l 300 \
    -f "$OUTPUT_DIR/phase1_search" 2>/dev/null

# Fase 2: Fuentes de certificados y DNS pasivo
echo "[+] Fase 2: CT logs y DNS pasivo..."
theHarvester -d "$TARGET" \
    -b crtsh,anubis,dnsdumpster,hackertarget,bufferoverun \
    -l 500 \
    -f "$OUTPUT_DIR/phase2_dns" 2>/dev/null

# Fase 3: APIs con key (si están configuradas)
echo "[+] Fase 3: APIs especializadas..."
theHarvester -d "$TARGET" \
    -b hunter,securityTrails,virustotal,shodan,intelx \
    -l 500 \
    -f "$OUTPUT_DIR/phase3_api" 2>/dev/null

# Combinar y deduplicar resultados
echo "[+] Consolidando resultados..."
python3 << PYEOF
import json, glob, os

all_emails = set()
all_hosts = set()
all_ips = set()

for f in glob.glob("$OUTPUT_DIR/*.json"):
    try:
        with open(f) as fh:
            data = json.load(fh)
        all_emails.update(data.get("emails", []))
        all_hosts.update(data.get("hosts", []))
        all_ips.update(data.get("ips", []))
    except:
        pass

# Guardar consolidados
with open("$OUTPUT_DIR/consolidated_emails.txt", "w") as f:
    f.write("\n".join(sorted(all_emails)))

with open("$OUTPUT_DIR/consolidated_hosts.txt", "w") as f:
    f.write("\n".join(sorted(all_hosts)))

with open("$OUTPUT_DIR/consolidated_ips.txt", "w") as f:
    f.write("\n".join(sorted(all_ips)))

print(f"  Emails únicos: {len(all_emails)}")
print(f"  Hosts únicos:  {len(all_hosts)}")
print(f"  IPs únicas:    {len(all_ips)}")
PYEOF

# Resumen final
echo ""
echo "══════════════════════════════════════"
echo "  RECONOCIMIENTO COMPLETADO"
echo "══════════════════════════════════════"
echo "  Emails:  $(wc -l < $OUTPUT_DIR/consolidated_emails.txt)"
echo "  Hosts:   $(wc -l < $OUTPUT_DIR/consolidated_hosts.txt)"
echo "  IPs:     $(wc -l < $OUTPUT_DIR/consolidated_ips.txt)"
echo "  Output:  $OUTPUT_DIR/"
echo "══════════════════════════════════════"
```

```bash
chmod +x harvester_recon.sh
./harvester_recon.sh example.com
```

### Pipeline completo con otras herramientas

```bash
TARGET="example.com"

# 1. theHarvester para emails y subdominios iniciales
theHarvester -d $TARGET -b google,bing,crtsh,hunter -l 300 -f initial_recon

# 2. Extraer hosts del JSON y pasarlos a subfinder
python3 -c "
import json
with open('initial_recon.json') as f:
    d = json.load(f)
for h in d.get('hosts', []):
    print(h.split(':')[0])
" > harvester_hosts.txt

# 3. Ampliar subdominios con subfinder
subfinder -d $TARGET -o subfinder_subs.txt

# 4. Combinar todo
cat harvester_hosts.txt subfinder_subs.txt | sort -u > all_hosts.txt

# 5. Resolver con dnsx cuáles están activos
cat all_hosts.txt | dnsx -silent -a -resp > alive_hosts.txt

# 6. Escanear puertos comunes en hosts activos
awk '{print $1}' alive_hosts.txt | nmap -iL - -sV --open -p 80,443,8080,8443 -oA web_services

echo "[+] Pipeline completado"
echo "    Hosts activos: $(wc -l < alive_hosts.txt)"
```

---

## 7. Integración con otras herramientas

### theHarvester + Maltego

Maltego puede importar resultados de theHarvester como entidades y visualizar las relaciones entre emails, hosts e IPs en un grafo interactivo.

```
Flujo:
1. Ejecutar theHarvester y guardar en XML (-f output)
2. Abrir Maltego → Import → Import from file
3. Seleccionar output.xml
4. Las entidades aparecen en el grafo:
   - Email entities
   - Domain entities
   - IP entities
5. Aplicar transforms adicionales de Maltego sobre cada entidad
```

### theHarvester + Metasploit

```bash
# Importar hosts encontrados en Metasploit
msfconsole -q
msf6 > workspace -a example_corp
msf6 > hosts -a $(cat consolidated_ips.txt | tr '\n' ',')

# O importar el XML directamente
msf6 > db_import /ruta/al/output.xml
msf6 > hosts     # Ver hosts importados
msf6 > services  # Ver servicios
```

### theHarvester + OSINT Framework

```
Flujo profesional completo:
1. theHarvester → emails + subdominios + IPs
2. Con emails:
   - Hunter.io → verificar validez
   - HaveIBeenPwned → comprobar brechas
   - LinkedIn → asociar a personas
3. Con subdominios:
   - Nmap → escanear servicios
   - Nuclei → buscar vulnerabilidades
   - EyeWitness → capturar screenshots
4. Con IPs:
   - Shodan → información de servicios
   - BGPView → identificar ASN y rangos
```

---

## 8. Limitaciones y soluciones

### Rate limiting de Google

Google bloquea las búsquedas automatizadas con CAPTCHA.

```bash
# Soluciones:
# 1. Usar proxies o Tor
theHarvester -d example.com -b google --proxy 127.0.0.1:9050

# 2. Reducir la velocidad con límite menor
theHarvester -d example.com -b google -l 50

# 3. Usar buscadores alternativos
theHarvester -d example.com -b bing,duckduckgo,yahoo

# 4. Usar fuentes que no tienen rate limiting
theHarvester -d example.com -b crtsh,anubis,hackertarget
```

### Resultados incompletos

```bash
# Problema: theHarvester solo encuentra 20-30 emails
# Solución: combinar múltiples herramientas

# theHarvester
theHarvester -d example.com -b all -l 500 -f theharvester_results

# hunter.io (API)
curl "https://api.hunter.io/v2/domain-search?domain=example.com&limit=100&api_key=TU_KEY" \
    | python3 -c "import json,sys; [print(e['value']) for e in json.load(sys.stdin)['data']['emails']]" \
    >> emails_combined.txt

# Extraer emails de theHarvester
python3 -c "
import json
with open('theharvester_results.json') as f:
    d = json.load(f)
for e in d.get('emails', []):
    print(e)
" >> emails_combined.txt

# Deduplicar
sort -u emails_combined.txt -o emails_final.txt
echo "Total emails únicos: $(wc -l < emails_final.txt)"
```

---

## 9. Buenas prácticas en entornos profesionales

### Documentar todo

```bash
# Ejecutar siempre con -f para guardar resultados
theHarvester -d example.com -b all -l 500 -f "recon_$(date +%Y%m%d)_example"

# Estructura de documentación recomendada
mkdir -p pentest_example/{recon,scanning,exploitation,post-exploitation,reporting}
mv *.json *.xml pentest_example/recon/
```

### Verificar que estás en scope

```bash
# Antes de cualquier recon, verificar el scope del engagement
cat scope.txt
# example.com          # Dominio principal
# *.example.com        # Todos los subdominios
# 93.184.216.0/24      # Rango de IPs

# Si dev.example.com NO está en scope, no incluirlo en el recon activo
# theHarvester es pasivo — no hay problema en encontrarlo
# El escaneo activo de hosts fuera de scope SÍ puede ser problema legal
```

### Flujo de trabajo en Red Team

```
1. KICK-OFF
   └── Recibir Rules of Engagement (RoE)
   └── Confirmar scope exacto con el cliente

2. RECONOCIMIENTO PASIVO (theHarvester aquí)
   └── theHarvester -b all → emails, hosts, IPs
   └── Shodan → servicios expuestos
   └── Google Dorks → información expuesta
   └── LinkedIn → empleados y cargos

3. ANÁLISIS
   └── Identificar patrón de emails
   └── Priorizar subdominios por interés
   └── Mapear la infraestructura

4. RECONOCIMIENTO ACTIVO (siguiente fase)
   └── Nmap en hosts en scope
   └── Nuclei/Nikto en servicios web
   └── EyeWitness para screenshots

5. DOCUMENTAR TODO antes de pasar a la siguiente fase
```

---

## 10. Cheatsheet de referencia rápida

```bash
# ── INSTALACIÓN ─────────────────────────────────────────
pip3 install theHarvester
git clone https://github.com/laramies/theHarvester

# ── SINTAXIS ────────────────────────────────────────────
theHarvester -d ejemplo.com -b fuentes -l 300 -f output

# ── FUENTES MÁS ÚTILES ──────────────────────────────────
# Sin API key:
-b google,bing,duckduckgo,crtsh,anubis,dnsdumpster,hackertarget

# Con API key gratuita:
-b hunter,securityTrails,virustotal,shodan,intelx

# Todas:
-b all

# ── COMANDOS POR OBJETIVO ───────────────────────────────
# Emails:
theHarvester -d ejemplo.com -b google,bing,hunter,linkedin -l 300

# Subdominios:
theHarvester -d ejemplo.com -b crtsh,anubis,dnsdumpster,virustotal,securityTrails -l 500

# Recon completo + guardar:
theHarvester -d ejemplo.com -b all -l 500 -f recon_output

# Con verificación DNS:
theHarvester -d ejemplo.com -b all -v

# Con brute force DNS (semi-activo):
theHarvester -d ejemplo.com -b google,crtsh -c

# ── PARSEAR RESULTADOS ───────────────────────────────────
python3 -c "
import json
with open('output.json') as f: d=json.load(f)
print('Emails:', len(d.get('emails',[])))
print('Hosts:', len(d.get('hosts',[])))
[print(e) for e in d.get('emails',[])]
"

# ── PIPELINE ─────────────────────────────────────────────
theHarvester -d $T -b all -l 500 -f recon && \
python3 -c "import json; [print(h.split(':')[0]) for h in json.load(open('recon.json')).get('hosts',[])]" | \
subfinder -dL /dev/stdin -silent | sort -u | dnsx -silent > alive_hosts.txt
```
