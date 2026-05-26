---
layout: post
title: "Gobuster y Feroxbuster — Fuzzing de Directorios y Archivos"
date: 2026-05-26
categories: [Escaneo y enumeracion]
tags: [gobuster, feroxbuster, fuzzing, directorios, web, enumeración, wordlists, seclists, pentesting]
description: "Guía profesional de Gobuster y Feroxbuster: fuzzing de directorios, archivos, subdominios y parámetros web con wordlists optimizadas, técnicas de evasión y flujos reales para pentesting web."
---

## La enumeración web como base del pentesting

Cuando tienes acceso a un servidor web, lo que ves en el navegador es solo una fracción de lo que existe. Las aplicaciones web modernas tienen directorios de administración, endpoints de API, archivos de configuración, versiones antiguas de páginas, paneles de backup, y docenas de recursos que nunca están enlazados desde la interfaz principal pero que el servidor sirve igualmente si conoces la ruta exacta.

El fuzzing de directorios es el proceso de probar sistemáticamente miles de rutas posibles para descubrir estos recursos ocultos. No es elegante y no requiere comprensión profunda de la aplicación — es fuerza bruta de rutas. Pero es extraordinariamente efectivo: en la mayoría de los engagements web, el acceso inicial o la escalada de privilegios viene de algo encontrado en esta fase.

Gobuster y Feroxbuster son las dos herramientas dominantes para esta tarea. Gobuster, escrito en Go, es más rápido y simple. Feroxbuster, también en Go, es más completo con recursión automática y mejor manejo de respuestas. Los dos se complementan y en un engagement serio se usan ambos.

Lo que diferencia un buen fuzzing de uno malo no es la herramienta sino tres factores: la **wordlist** (qué rutas pruebas), los **filtros** (cómo distingues resultados válidos de ruido), y la **configuración de velocidad** (cómo no romper la aplicación ni activar rate limiting). Este post cubre los tres en profundidad.

---

## 1. Gobuster — rápido, directo y fiable

Gobuster es una herramienta Go creada por OJ Reeves que soporta varios modos de operación: fuzzing de directorios/ficheros (`dir`), subdominios DNS (`dns`), hosts virtuales (`vhost`), buckets S3 (`s3`), y servidores GCS (`gcs`). El modo `dir` es el más usado en pentesting web.

### Instalación

```bash
# En Kali Linux — viene preinstalado
gobuster version

# Instalar o actualizar con go
go install github.com/OJ/gobuster/v3@latest

# Con apt en sistemas Debian/Ubuntu
sudo apt update && sudo apt install gobuster

# Verificar
gobuster --help
```

### Modo dir — Fuzzing de directorios y ficheros

El modo `dir` es el núcleo de Gobuster. Prueba cada entrada de la wordlist como ruta en el servidor y reporta las que devuelven respuestas válidas.

```bash
# Escaneo básico — lo mínimo necesario
gobuster dir -u http://10.10.10.10 -w /usr/share/wordlists/dirb/common.txt

# Los flags más importantes explicados:
# -u → URL objetivo (incluir el esquema http:// o https://)
# -w → wordlist a usar (el factor más crítico del escaneo)
# -t → número de threads (por defecto 10, aumentar para mayor velocidad)
# -o → guardar resultados en un fichero
# -x → extensiones a añadir a cada palabra de la wordlist
# -s → códigos de respuesta a mostrar (por defecto 200,204,301,302,307,401,403)
# -b → códigos de respuesta a ignorar (blacklist)
# -k → ignorar errores de certificado SSL (para HTTPS con cert autofirmado)
# --no-error → no mostrar errores en pantalla (output más limpio)

# Escaneo estándar para un engagement real
gobuster dir \
    -u https://example.com \
    -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt \
    -t 50 \
    -x php,html,txt,bak,js,json \
    -o gobuster_results.txt \
    -k \
    --no-error

# Explicación de las opciones elegidas:
# -t 50 → 50 threads es un buen balance entre velocidad y no romper el servidor
# -x php,html,txt,bak,js,json → buscar estas extensiones también
#   php,html → páginas comunes
#   txt → ficheros de texto (robots.txt, notas, credenciales)
#   bak → backups de ficheros (config.bak, .bak)
#   js → JavaScript con lógica de cliente y posibles endpoints
#   json → ficheros de configuración y datos
# -k → crítico para HTTPS con certificados autofirmados (muy común en labs)
```

### Añadir extensiones — el multiplicador de cobertura

El flag `-x` es uno de los más importantes porque multiplica el número de rutas probadas. Sin `-x`, Gobuster solo prueba `/admin`, `/login`, `/config`. Con `-x php,bak`, también prueba `/admin.php`, `/admin.bak`, `/login.php`, `/login.bak`, etc.

```bash
# Extensiones según el stack tecnológico detectado:

# Servidor PHP (detectado en cabecera Server o X-Powered-By)
gobuster dir -u http://target -w wordlist.txt -x php,php3,php4,php5,phtml

# Servidor ASP.NET / IIS
gobuster dir -u http://target -w wordlist.txt -x asp,aspx,ashx,asmx,config

# Java / Tomcat
gobuster dir -u http://target -w wordlist.txt -x jsp,do,action,java,class

# Node.js / Express
gobuster dir -u http://target -w wordlist.txt -x js,json,ts

# Ficheros de backup y configuración (siempre útil independientemente del stack)
gobuster dir -u http://target -w wordlist.txt \
    -x bak,backup,old,orig,copy,swp,save,tmp,~

# Documentos y datos (útil en aplicaciones empresariales)
gobuster dir -u http://target -w wordlist.txt \
    -x pdf,xlsx,docx,csv,xml,log,sql
```

### Filtrar por código de respuesta — separar señal del ruido

Por defecto Gobuster muestra todos los códigos 200, 204, 301, 302, 307, 401, 403. En algunos servidores esto genera mucho ruido (el servidor responde 403 a casi todo). Los flags `-s` y `-b` permiten controlar qué respuestas se muestran:

```bash
# Mostrar SOLO respuestas 200 (páginas que existen y son accesibles)
gobuster dir -u http://target -w wordlist.txt -s 200

# Mostrar 200 y 301 (páginas que existen, incluyendo redirecciones)
gobuster dir -u http://target -w wordlist.txt -s 200,301,302

# Ocultar respuestas 404 y 403 (reducir ruido)
gobuster dir -u http://target -w wordlist.txt -b 404,403

# Filtrar por tamaño de respuesta — muy útil cuando el servidor devuelve
# 200 para todas las rutas (página de error personalizada con código 200)
# Primero identificar el tamaño de la página de "no encontrado":
curl -s http://target/ruta-que-no-existe-12345 | wc -c
# Output: 1823 → ese es el tamaño de la página de error

# Luego filtrar respuestas de ese tamaño exacto
gobuster dir -u http://target -w wordlist.txt --exclude-length 1823
```

### Fuzzing con autenticación

Muchos paneles de administración requieren autenticación básica HTTP antes de mostrar el contenido. Gobuster puede incluir cabeceras de autenticación:

```bash
# HTTP Basic Authentication
gobuster dir -u http://target/admin/ -w wordlist.txt \
    -U admin -P password123

# Con cookie de sesión (después de hacer login manual)
gobuster dir -u http://target -w wordlist.txt \
    -c "session=abc123xyz; csrftoken=def456"

# Con cabecera Authorization personalizada (JWT, API key, etc.)
gobuster dir -u http://target -w wordlist.txt \
    -H "Authorization: Bearer eyJhbGc..." \
    -H "X-API-Key: mi_api_key"

# Con User-Agent personalizado (para evadir WAF que bloquean User-Agents de herramientas)
gobuster dir -u http://target -w wordlist.txt \
    -a "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
```

### Modo dns — Fuzzing de subdominios

Gobuster también puede hacer fuerza bruta de subdominios DNS, que complementa las técnicas pasivas de Subfinder y Amass:

```bash
# Fuzzing de subdominios DNS
gobuster dns \
    -d example.com \
    -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
    -t 50 \
    --no-error

# Mostrar también las IPs resueltas
gobuster dns -d example.com -w wordlist.txt --show-ips

# Con resolver DNS específico (para evitar rate limiting en el DNS del ISP)
gobuster dns -d example.com -w wordlist.txt -r 8.8.8.8
```

### Modo vhost — Virtual Host fuzzing

Los virtual hosts permiten que un solo servidor web sirva múltiples sitios web diferentes. El fuzzing de virtual hosts descubre sitios que comparten la misma IP pero usan diferentes nombres de host:

```bash
# Fuzzing de virtual hosts
# Gobuster modifica la cabecera Host en cada petición
gobuster vhost \
    -u http://10.10.10.10 \
    -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
    --append-domain \
    -t 50

# --append-domain añade el dominio base a cada palabra
# Sin esta opción prueba "admin", "dev", "staging"
# Con esta opción prueba "admin.example.com", "dev.example.com", "staging.example.com"
```

---

## 2. Feroxbuster — recursión automática y análisis profundo

Feroxbuster fue creado por epi052 como alternativa a Gobuster con recursión automática incorporada. Su característica más importante es que cuando descubre un directorio nuevo, automáticamente empieza a fuzzear ese directorio también, sin que tengas que configurarlo manualmente. Esto lo hace ideal para descubrir estructuras de directorios profundas.

### Instalación

```bash
# En Kali Linux
sudo apt install feroxbuster

# Con cargo (gestor de paquetes de Rust)
cargo install feroxbuster

# Binario precompilado desde GitHub
wget https://github.com/epi052/feroxbuster/releases/latest/download/x86_64-linux-feroxbuster.zip
unzip x86_64-linux-feroxbuster.zip
chmod +x feroxbuster
sudo mv feroxbuster /usr/local/bin/

# Verificar
feroxbuster --version
```

### Uso básico

```bash
# Escaneo básico con recursión automática
feroxbuster -u http://10.10.10.10 -w /usr/share/seclists/Discovery/Web-Content/common.txt

# Los flags principales de Feroxbuster:
# -u → URL objetivo
# -w → wordlist
# -t → threads (por defecto 50)
# -x → extensiones
# -o → fichero de salida
# -k → ignorar errores SSL
# --depth → profundidad máxima de recursión (por defecto sin límite)
# --filter-status → excluir códigos de respuesta
# --filter-size → excluir por tamaño de respuesta
# --filter-words → excluir por número de palabras en la respuesta
# --quiet → output mínimo (solo resultados)
# --smart-filter → filtrado automático de respuestas similares

# Escaneo estándar con configuración óptima
feroxbuster \
    -u https://example.com \
    -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt \
    -x php,html,txt,js,bak \
    -t 50 \
    --depth 3 \
    -k \
    -o ferox_results.txt
```

### Recursión automática — la ventaja principal

La recursión es lo que diferencia fundamentalmente a Feroxbuster de Gobuster. Cuando Feroxbuster encuentra `/admin/` con código 200 o 301, automáticamente empieza a fuzzear `/admin/` con la misma wordlist. Esto descubre estructuras como:

```
/admin/                     → 301 (directorio encontrado, recursión automática)
/admin/users/               → 200 (encontrado por recursión)
/admin/users/edit.php       → 200
/admin/config/              → 200 (encontrado por recursión de nivel 2)
/admin/config/database.php  → 200
/admin/config/database.bak  → 200 ← ESTO es el hallazgo valioso
```

Sin recursión (Gobuster por defecto), solo habrías encontrado `/admin/`. Con recursión, llegas hasta `database.bak`.

```bash
# Controlar la profundidad de recursión
# Sin límite puede ser muy lento en sitios grandes
feroxbuster -u http://target -w wordlist.txt --depth 2   # Max 2 niveles
feroxbuster -u http://target -w wordlist.txt --depth 4   # Max 4 niveles

# Desactivar recursión (comportamiento como Gobuster)
feroxbuster -u http://target -w wordlist.txt --depth 1

# Limitar el número de peticiones por segundo para no sobrecargar el servidor
feroxbuster -u http://target -w wordlist.txt --rate-limit 100
```

### Filtrado avanzado de resultados

Feroxbuster tiene opciones de filtrado más granulares que Gobuster, especialmente útiles cuando el servidor devuelve muchos falsos positivos:

```bash
# Filtrar por código de respuesta
feroxbuster -u http://target -w wordlist.txt \
    --filter-status 404,403,400

# Filtrar por tamaño exacto de respuesta (en bytes)
# Primero identificar el tamaño de la página de "no encontrado":
curl -s http://target/pagina-inexistente-xyz | wc -c
# Luego filtrar ese tamaño:
feroxbuster -u http://target -w wordlist.txt --filter-size 1823

# Filtrar por número de palabras en la respuesta
# Útil cuando todas las páginas de error tienen el mismo número de palabras
feroxbuster -u http://target -w wordlist.txt --filter-words 45

# Filtrar por número de líneas en la respuesta
feroxbuster -u http://target -w wordlist.txt --filter-lines 12

# Filtrar por expresión regular en el body de la respuesta
# Por ejemplo, excluir páginas que contienen "Page Not Found"
feroxbuster -u http://target -w wordlist.txt \
    --filter-regex "Page Not Found|404|not found"

# Smart filter — Feroxbuster detecta automáticamente patrones de respuesta similares
# y los filtra para reducir el ruido (recomendado para sitios con WAF)
feroxbuster -u http://target -w wordlist.txt --smart-filter
```

### Múltiples wordlists y targets

```bash
# Múltiples URLs objetivo desde un fichero
feroxbuster -u http://target -w wordlist.txt \
    --stdin < urls_objetivo.txt

# Scan de múltiples subdominios activos
cat subdominios_activos.txt | feroxbuster \
    --stdin \
    -w /usr/share/seclists/Discovery/Web-Content/common.txt \
    -t 50 \
    -k \
    -o ferox_multi.txt
```

---

## 3. Wordlists — el factor más importante

La wordlist determina más que cualquier otro factor el éxito del fuzzing. Una herramienta mediocre con una buena wordlist supera a una herramienta excelente con una wordlist pobre. SecLists de Daniel Miessler es la colección de referencia absoluta.

### Instalar SecLists

```bash
# En Kali Linux — suele estar preinstalado en /usr/share/seclists
ls /usr/share/seclists/Discovery/Web-Content/

# Si no está instalado:
sudo apt install seclists

# O clonar el repositorio completo (más de 1GB)
git clone https://github.com/danielmiessler/SecLists.git /usr/share/seclists
```

### Las wordlists más usadas para fuzzing web

```bash
# ── DIRECTORIOS GENERALES ─────────────────────────────────────────────────

# common.txt — ~4.600 entradas — ideal para escaneo rápido inicial
/usr/share/seclists/Discovery/Web-Content/common.txt

# directory-list-2.3-small.txt — ~87.000 entradas — balance calidad/velocidad
/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt

# directory-list-2.3-medium.txt — ~220.000 entradas — estándar en pentesting
# La más usada en engagements reales — buen equilibrio cobertura/tiempo
/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt

# directory-list-2.3-big.txt — ~1.273.000 entradas — cobertura máxima
# Para cuando el tiempo no es un factor y necesitas cobertura total
/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt

# ── APIS Y ENDPOINTS ──────────────────────────────────────────────────────

# api/objects.txt — endpoints de API REST comunes
/usr/share/seclists/Discovery/Web-Content/api/objects.txt

# burp-parameter-names.txt — nombres de parámetros comunes
/usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt

# ── FICHEROS ESPECÍFICOS ──────────────────────────────────────────────────

# CommonBackdoors-PHP.txt — webshells y backdoors PHP comunes
/usr/share/seclists/Discovery/Web-Content/CommonBackdoors-PHP.txt

# quickhits.txt — rutas de alto impacto, pocas entradas
# Ideal para un primer escaneo rápido buscando los hallazgos más críticos
/usr/share/seclists/Discovery/Web-Content/quickhits.txt

# ── TECNOLOGÍAS ESPECÍFICAS ───────────────────────────────────────────────

# CMS-Based — wordlists específicas por CMS
/usr/share/seclists/Discovery/Web-Content/CMS/

# spring-boot.txt — endpoints actuator de Spring Boot
/usr/share/seclists/Discovery/Web-Content/spring-boot.txt

# ── SUBDOMINIOS ───────────────────────────────────────────────────────────
/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
/usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt
```

### Crear wordlists personalizadas

Las wordlists genéricas son el punto de partida, pero las wordlists personalizadas basadas en el objetivo específico suelen dar los mejores resultados:

```bash
# Generar wordlist desde el contenido de la web objetivo
# cewl scrapea el sitio y extrae palabras del contenido
cewl http://example.com -d 3 -m 5 -w wordlist_custom.txt
# -d 3 → profundidad de crawl
# -m 5 → longitud mínima de palabras

# Combinar la wordlist generada con una estándar
cat wordlist_custom.txt /usr/share/seclists/Discovery/Web-Content/common.txt \
    | sort -u > wordlist_combinada.txt

# Generar variaciones de una palabra base
# Por ejemplo, si la empresa se llama "ExampleCorp":
cat << 'EOF' > variantes_empresa.txt
examplecorp
example-corp
example_corp
ExampleCorp
examplecorp_admin
examplecorp_backup
examplecorp_dev
admin_examplecorp
backup_examplecorp
example
corp
EOF

# Añadir sufijos comunes a una lista existente
while read word; do
    for suffix in "" "_backup" "_old" "_test" "_dev" "_bak" ".bak" ".old"; do
        echo "${word}${suffix}"
    done
done < wordlist_base.txt | sort -u > wordlist_con_sufijos.txt
```

---

## 4. Técnicas avanzadas de fuzzing

### Fuzzing de parámetros con ffuf

ffuf (Fuzz Faster U Fool) es la herramienta más potente para fuzzing de parámetros y valores. Aunque Gobuster y Feroxbuster son mejores para directorios, ffuf brilla en el fuzzing de parámetros GET/POST, cabeceras, y valores:

```bash
# Instalar ffuf
go install github.com/ffuf/ffuf/v2@latest

# Fuzzing de directorios básico (similar a Gobuster)
ffuf -u http://target/FUZZ -w wordlist.txt

# La palabra FUZZ es el placeholder — se reemplaza por cada entrada de la wordlist

# Fuzzing de parámetros GET
# ¿Qué parámetros acepta este endpoint?
ffuf -u "http://target/search?FUZZ=test" \
    -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
    -fs 1823   # Filtrar respuestas del tamaño de la página de error

# Fuzzing de valores de un parámetro conocido
# ¿Qué valores acepta el parámetro id?
ffuf -u "http://target/user?id=FUZZ" \
    -w /usr/share/seclists/Fuzzing/Integers.txt \
    -fs 0 \
    -mc 200

# Fuzzing POST
ffuf -u http://target/login \
    -X POST \
    -d "username=admin&password=FUZZ" \
    -w /usr/share/seclists/Passwords/probable-v2-top1575.txt \
    -H "Content-Type: application/x-www-form-urlencoded" \
    -fs 1823   # Tamaño de la respuesta de error de login

# Fuzzing de cabeceras HTTP
ffuf -u http://target/ \
    -H "X-FUZZ: test" \
    -w /usr/share/seclists/Discovery/Web-Content/BurpSuite-ParamMiner/uppercase-headers.txt \
    -fs 1823

# Fuzzing doble — dos posiciones a la vez (para combinar usuario/contraseña)
ffuf -u http://target/login \
    -X POST \
    -d "username=FUZZ1&password=FUZZ2" \
    -w usuarios.txt:FUZZ1 \
    -w /usr/share/seclists/Passwords/probable-v2-top1575.txt:FUZZ2 \
    -H "Content-Type: application/x-www-form-urlencoded" \
    -fs 1823
```

### Evasión de WAF durante el fuzzing

Los WAF (Web Application Firewalls) pueden bloquear el fuzzing si detectan el volumen o el patrón de peticiones. Estas técnicas ayudan a evadir la detección:

```bash
# Reducir la velocidad — lo más efectivo para evadir rate limiting
gobuster dir -u http://target -w wordlist.txt \
    -t 10 \          # Pocos threads
    --delay 200ms    # Espera 200ms entre peticiones

feroxbuster -u http://target -w wordlist.txt \
    --rate-limit 30  # Máximo 30 peticiones por segundo

# Cambiar el User-Agent para no aparecer como herramienta conocida
gobuster dir -u http://target -w wordlist.txt \
    -a "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36"

feroxbuster -u http://target -w wordlist.txt \
    --user-agent "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"

# Añadir cabeceras que parecen tráfico legítimo
gobuster dir -u http://target -w wordlist.txt \
    -H "Accept: text/html,application/xhtml+xml,application/xml" \
    -H "Accept-Language: es-ES,es;q=0.9" \
    -H "Referer: https://www.google.com"

# Usar proxy para pasar el tráfico por Burp Suite
# (permite ver las peticiones y modificarlas manualmente)
gobuster dir -u http://target -w wordlist.txt \
    --proxy http://127.0.0.1:8080

feroxbuster -u http://target -w wordlist.txt \
    --proxy http://127.0.0.1:8080
```

---

## 5. Flujo completo de enumeración web

Este es el flujo de trabajo que uso en engagements reales, combinando múltiples herramientas en orden de mayor a menor velocidad:

```bash
#!/bin/bash
# web_enum.sh — Enumeración web completa
# Uso: ./web_enum.sh http://10.10.10.10
# Requiere: gobuster, feroxbuster, ffuf, seclists

TARGET=$1
OUTDIR="./web_enum_$(date +%Y%m%d_%H%M)"
mkdir -p "$OUTDIR"

if [ -z "$TARGET" ]; then
    echo "[-] Uso: $0 <URL>"
    exit 1
fi

# Extraer el dominio/IP del target para los scans DNS
DOMAIN=$(echo "$TARGET" | sed 's|https\?://||' | cut -d'/' -f1)

echo "╔══════════════════════════════════════════╗"
echo "  Enumeración web: $TARGET"
echo "╚══════════════════════════════════════════╝"

# ── FASE 1: ESCANEO RÁPIDO INICIAL ───────────────────────────────────
echo ""
echo "[1/5] Gobuster — quickhits (rutas críticas rápidas)..."
gobuster dir \
    -u "$TARGET" \
    -w /usr/share/seclists/Discovery/Web-Content/quickhits.txt \
    -t 50 \
    -k \
    --no-error \
    -o "$OUTDIR/01_quickhits.txt" 2>/dev/null

echo "  Hallazgos: $(grep -c 'Status' $OUTDIR/01_quickhits.txt 2>/dev/null || echo 0)"

# ── FASE 2: FUZZING ESTÁNDAR CON GOBUSTER ────────────────────────────
echo ""
echo "[2/5] Gobuster — directorios comunes (medium)..."
gobuster dir \
    -u "$TARGET" \
    -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt \
    -x php,html,txt,js,bak,json,xml \
    -t 50 \
    -k \
    --no-error \
    -b 404,400 \
    -o "$OUTDIR/02_gobuster_medium.txt" 2>/dev/null

echo "  Hallazgos: $(grep -c 'Status' $OUTDIR/02_gobuster_medium.txt 2>/dev/null || echo 0)"

# ── FASE 3: FUZZING RECURSIVO CON FEROXBUSTER ────────────────────────
echo ""
echo "[3/5] Feroxbuster — recursión automática (profundidad 3)..."
feroxbuster \
    -u "$TARGET" \
    -w /usr/share/seclists/Discovery/Web-Content/common.txt \
    -x php,html,txt,bak,js \
    -t 40 \
    --depth 3 \
    -k \
    --smart-filter \
    --quiet \
    -o "$OUTDIR/03_feroxbuster_recursive.txt" 2>/dev/null

echo "  Hallazgos: $(wc -l < $OUTDIR/03_feroxbuster_recursive.txt 2>/dev/null || echo 0)"

# ── FASE 4: FUZZING DE PARÁMETROS CON FFUF ───────────────────────────
echo ""
echo "[4/5] ffuf — fuzzing de parámetros..."
ffuf \
    -u "${TARGET}/FUZZ" \
    -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
    -t 40 \
    -mc 200,301,302,401,403 \
    -o "$OUTDIR/04_ffuf_params.json" \
    -of json \
    -s 2>/dev/null

echo "  Hallazgos: $(python3 -c "import json; d=json.load(open('$OUTDIR/04_ffuf_params.json')); print(len(d.get('results',[])))" 2>/dev/null || echo 0)"

# ── FASE 5: SUBDOMINIOS VIRTUALES ────────────────────────────────────
echo ""
echo "[5/5] Gobuster — virtual hosts..."
gobuster vhost \
    -u "$TARGET" \
    -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
    -t 50 \
    -k \
    --append-domain \
    --no-error \
    -o "$OUTDIR/05_vhosts.txt" 2>/dev/null

echo "  Hallazgos: $(grep -c 'Found' $OUTDIR/05_vhosts.txt 2>/dev/null || echo 0)"

# ── RESUMEN ──────────────────────────────────────────────────────────
echo ""
echo "╔══════════════════════════════════════════╗"
echo "  ENUMERACIÓN COMPLETADA"
echo "╠══════════════════════════════════════════╣"
printf "  %-28s %s\n" "Quickhits:" "$(grep -c 'Status' $OUTDIR/01_quickhits.txt 2>/dev/null || echo 0)"
printf "  %-28s %s\n" "Gobuster medium:" "$(grep -c 'Status' $OUTDIR/02_gobuster_medium.txt 2>/dev/null || echo 0)"
printf "  %-28s %s\n" "Feroxbuster recursivo:" "$(wc -l < $OUTDIR/03_feroxbuster_recursive.txt 2>/dev/null || echo 0)"
printf "  %-28s %s\n" "Virtual hosts:" "$(grep -c 'Found' $OUTDIR/05_vhosts.txt 2>/dev/null || echo 0)"
echo "╠══════════════════════════════════════════╣"
echo "  Resultados en: $OUTDIR/"
echo "╚══════════════════════════════════════════╝"

# Mostrar los hallazgos más interesantes
echo ""
echo "[!] Rutas con potencial alto (backups, configs, admin):"
grep -hiE "\.(bak|backup|old|sql|env|config|conf)|\/(admin|login|panel|dashboard|backup|upload)" \
    "$OUTDIR"/*.txt 2>/dev/null | grep -v "^#" | sort -u | head -20
```

---

## 6. Cheatsheet de referencia rápida

```bash
# ── GOBUSTER DIR ──────────────────────────────────────────────────────────
gobuster dir -u http://target -w wordlist.txt                    # Básico
gobuster dir -u http://target -w wordlist.txt -x php,bak,txt    # Con extensiones
gobuster dir -u http://target -w wordlist.txt -t 50 -k -o out   # Flags estándar
gobuster dir -u http://target -w wordlist.txt -b 404,403         # Filtrar códigos
gobuster dir -u http://target -w wordlist.txt --exclude-length 1823 # Filtrar tamaño
gobuster dir -u http://target -w wordlist.txt -U admin -P pass   # Con auth básica
gobuster dir -u http://target -w wordlist.txt -c "sess=abc"      # Con cookie
gobuster dir -u http://target -w wordlist.txt -H "Auth: Bearer TOKEN"

# ── GOBUSTER DNS / VHOST ──────────────────────────────────────────────────
gobuster dns -d example.com -w subdomains.txt -t 50 --show-ips
gobuster vhost -u http://target -w subdomains.txt --append-domain

# ── FEROXBUSTER ───────────────────────────────────────────────────────────
feroxbuster -u http://target -w wordlist.txt                     # Básico + recursión
feroxbuster -u http://target -w wordlist.txt --depth 3           # Limitar recursión
feroxbuster -u http://target -w wordlist.txt --filter-status 404,403
feroxbuster -u http://target -w wordlist.txt --filter-size 1823
feroxbuster -u http://target -w wordlist.txt --smart-filter      # Filtro automático
feroxbuster -u http://target -w wordlist.txt --rate-limit 50     # Limitar velocidad

# ── FFUF ──────────────────────────────────────────────────────────────────
ffuf -u http://target/FUZZ -w wordlist.txt                       # Directorios
ffuf -u "http://target/page?FUZZ=val" -w params.txt -fs 1823     # Parámetros GET
ffuf -u http://target/ -H "Host: FUZZ.example.com" -w subs.txt  # Virtual hosts
ffuf -u http://target/login -X POST -d "user=a&pass=FUZZ" \
     -w passwords.txt -H "Content-Type: application/x-www-form-urlencoded"

# ── WORDLISTS ESENCIALES ──────────────────────────────────────────────────
/usr/share/seclists/Discovery/Web-Content/quickhits.txt          # Rápido, impacto alto
/usr/share/seclists/Discovery/Web-Content/common.txt             # General, rápido
/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt  # Estándar
/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt     # Exhaustivo
/usr/share/seclists/Discovery/Web-Content/spring-boot.txt        # Spring Actuator
/usr/share/seclists/Discovery/Web-Content/api/objects.txt        # API endpoints
/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# ── EXTENSIONES POR STACK ─────────────────────────────────────────────────
PHP:    -x php,php3,php4,phtml,bak,old
ASP:    -x asp,aspx,ashx,asmx,config
Java:   -x jsp,do,action,java
Node:   -x js,json,ts
Backup: -x bak,backup,old,orig,swp,tmp,~
Docs:   -x pdf,xlsx,docx,csv,xml,log,sql

# ── RECURSOS ──────────────────────────────────────────────────────────────
https://github.com/OJ/gobuster              → Gobuster
https://github.com/epi052/feroxbuster       → Feroxbuster
https://github.com/ffuf/ffuf                → ffuf
https://github.com/danielmiessler/SecLists  → SecLists (wordlists)
https://cewl.readthedocs.io                 → cewl (wordlists personalizadas)
```
