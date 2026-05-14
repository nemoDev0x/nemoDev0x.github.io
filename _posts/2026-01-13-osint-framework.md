---
layout: post
title: "Reconocimiento Pasivo con OSINT Framework"
date: 2026-01-13
categories: [reconocimiento]
tags: [osint, reconocimiento, recon, shodan, theHarvester, metadatos, subfinder, amass]
description: "Guía profesional de reconocimiento pasivo: metodología, teoría y práctica real de las técnicas OSINT usadas en Red Team y Bug Bounty."
---

## ¿Qué es el reconocimiento pasivo y por qué importa?

Imagina que vas a auditar la seguridad de una empresa. Lo primero que haría cualquier atacante real antes de intentar comprometer nada es observar desde la distancia — igual que un ladrón que estudia una joyería durante días antes de entrar. Eso es el reconocimiento pasivo: recopilar toda la información posible sobre el objetivo **sin interactuar directamente con él**.

La diferencia fundamental con el reconocimiento activo es que en el pasivo **no enviamos ni un solo paquete al objetivo**. Toda la información que obtenemos proviene de fuentes públicas: motores de búsqueda, registros DNS, bases de datos de certificados, redes sociales, repositorios de código, brechas de datos filtradas. El objetivo no puede detectar que le estamos investigando porque técnicamente no le estamos tocando.

Esto tiene implicaciones enormes en un entorno profesional. Un buen reconocimiento pasivo puede revelarte:

- El esquema de nombres de usuario que usa la empresa (para construir listas de fuerza bruta)
- Emails corporativos reales (para ataques de phishing dirigido o password spraying)
- Tecnologías y versiones en producción (para buscar CVEs aplicables)
- Empleados y sus roles (para ingeniería social)
- Credenciales filtradas en brechas antiguas (que quizás siguen siendo válidas)
- Subdominios olvidados con versiones antiguas de aplicaciones
- Infraestructura interna expuesta accidentalmente

Un pentest donde se omite el reconocimiento pasivo es un pentest incompleto. La diferencia entre un resultado mediocre y uno que realmente aporta valor al cliente a menudo reside en la calidad de esta fase.

---

## 1. OSINT Framework — El mapa del territorio

Antes de lanzar cualquier herramienta, necesitas saber qué existe. [OSINT Framework](https://osintframework.com) es un árbol visual interactivo que categoriza cientos de herramientas y fuentes de inteligencia de fuentes abiertas. No es una herramienta en sí misma — es una guía de recursos organizada según el tipo de información que buscas.

La utilidad de OSINT Framework reside en que te evita reinventar la rueda. Si necesitas encontrar emails, hay una categoría para eso. Si necesitas rastrear un número de teléfono, hay otra. Si quieres saber qué dispositivos tiene una empresa expuestos en internet, también está cubierto. Es el punto de partida antes de cualquier investigación.

La estructura principal del árbol es la siguiente:

```
OSINT Framework
├── Username                → Rastrear un alias en múltiples plataformas
├── Email Address           → Verificar, rastrear y enumerar emails
├── Domain Name             → Whois, DNS, subdominios, historial
├── IP Address              → Geolocalización, ASN, historial, reputación
├── Social Networks         → Twitter, LinkedIn, Facebook, Instagram
├── People Search Engines   → Spokeo, Pipl, WhitePages
├── Business Records        → Empresas, directivos, filiales, registros mercantiles
├── Transportation          → Matrículas, barcos, aviones
├── Geospatial              → Mapas, satélites, streetview
├── Images / Videos         → Búsqueda inversa, metadata EXIF
├── Documents               → Metadatos de PDFs, DOCx, presentaciones
├── Forums / Blogs          → Rastros digitales en comunidades
├── Dark Web                → Paste sites, mercados underground
├── Digital Currency        → Rastreo de wallets de criptomonedas
└── Threat Intelligence     → IOCs, malware, infraestructura de C2
```

En la práctica, cuando recibes un scope de un cliente, lo primero es sentarte frente a OSINT Framework e ir categoría por categoría preguntándote: ¿qué información de esta categoría podría tener el objetivo y cómo la obtengo? Este proceso mental antes de ejecutar nada es lo que separa un reconocimiento sistemático de un reconocimiento caótico.

---

## 2. Metodología: el ciclo de inteligencia

Un error muy común en gente que empieza en seguridad es lanzar herramientas sin orden. Corren Nmap, luego theHarvester, luego Shodan, sin un hilo conductor. El resultado es un montón de datos sin contexto que no llevan a ningún sitio.

Los equipos de Red Team profesionales aplican el **ciclo de inteligencia** militar adaptado al OSINT. Este ciclo tiene seis fases que se repiten iterativamente:

**Fase 1 — Planificación:** Definir exactamente qué información necesitas y por qué. ¿Qué vas a hacer con los emails que encuentres? ¿Qué harás si encuentras un subdominio con una versión vulnerable de Apache? Tener claro el objetivo final evita perder tiempo en información irrelevante.

**Fase 2 — Recolección:** Ejecutar las herramientas y técnicas para obtener los datos en bruto. En esta fase no analizas — solo recoges.

**Fase 3 — Procesamiento:** Limpiar, normalizar y estructurar los datos. Eliminar duplicados, separar lo relevante de lo irrelevante, convertir formatos.

**Fase 4 — Análisis:** Cruzar los datos entre sí para extraer conclusiones. Un email encontrado en theHarvester + el perfil de LinkedIn + una contraseña en Dehashed = vector de ataque concreto.

**Fase 5 — Diseminación:** Documentar los hallazgos de forma clara. En un pentest esto es el reporte; en un CTF es tu árbol de notas.

**Fase 6 — Retroalimentación:** Preguntarte qué te falta. ¿Tienes subdominios pero no emails? Volver a la fase de recolección específicamente para emails.

Este ciclo no es lineal — es iterativo. Conforme encuentras información nueva, vuelves a analizar y eso genera nuevas preguntas que te llevan a nueva recolección.

### Estructura de documentación recomendada

Antes de empezar cualquier reconocimiento, crea esta estructura de directorios. Tener los datos organizados desde el principio te ahorra horas de trabajo posterior:

```bash
mkdir -p recon_empresa/{dominios,subdominios,emails,empleados,infraestructura,credenciales,documentos,screenshots}
```

La estructura queda así:

```
recon_empresa/
├── dominios/
│   ├── whois.txt              # Resultados de WHOIS
│   └── dns_records.txt        # Registros DNS (A, MX, TXT, NS)
├── subdominios/
│   ├── subfinder.txt          # Output de subfinder
│   ├── amass.txt              # Output de amass
│   ├── crtsh.txt              # CT logs de crt.sh
│   └── all_unique.txt         # Todos deduplicados
├── emails/
│   ├── harvester.txt          # Output de theHarvester
│   ├── hunter.txt             # Output de Hunter.io
│   └── patron.md              # Patrón de emails identificado
├── empleados/
│   ├── linkedin.md            # Empleados encontrados en LinkedIn
│   └── organigrama.md         # Estructura inferida
├── infraestructura/
│   ├── shodan.txt             # Resultados de Shodan
│   ├── asn.txt                # Rangos de IP y ASN
│   └── tecnologias.md         # Stack tecnológico identificado
├── credenciales/
│   └── brechas.txt            # Datos encontrados en brechas
├── documentos/
│   ├── descargados/           # PDFs, DOCx, etc. del objetivo
│   └── metadatos.csv          # Metadatos extraídos
└── screenshots/               # Capturas de evidencias
```

---

## 3. Reconocimiento de dominios

El dominio es el punto de entrada natural de cualquier investigación sobre una empresa. Desde el dominio principal puedes expandirte hacia subdominios, infraestructura, emails, tecnologías y más. Hay varias capas de información que extraer.

### WHOIS — El registro de propiedad de dominio

Cuando alguien registra un dominio, deja un rastro en las bases de datos WHOIS. Históricamente esto incluía nombre, dirección, teléfono y email del registrante — información muy valiosa. Hoy muchas empresas usan servicios de privacidad que ocultan estos datos, pero incluso en ese caso WHOIS revela información útil: el registrar, las fechas de creación y expiración, y los servidores DNS.

Los servidores DNS (nameservers) son especialmente interesantes porque revelan qué proveedor gestiona el DNS del objetivo. Si el objetivo usa Cloudflare como DNS, probablemente también use Cloudflare como CDN, lo que afecta a cómo abordarás la fase activa. Si usa Route53 de AWS, es muy probable que parte de su infraestructura esté en Amazon.

```bash
# Consulta WHOIS básica al dominio principal
whois ejemplo.com
```

El resultado incluye campos como `Registrant Organization`, `Name Server`, `Creation Date` y `Expiration Date`. Fíjate especialmente en la fecha de creación — un dominio registrado hace dos semanas para una empresa que dice tener veinte años es una señal de alerta (posible phishing o infraestructura reciente).

```bash
# WHOIS de una IP — revela el propietario del bloque de IPs
whois 93.184.216.34

# WHOIS con servidor específico de IANA
whois -h whois.iana.org ejemplo.com
```

Cuando haces WHOIS de una IP obtienes el bloque CIDR completo asignado a esa organización. Esto te da el rango de IPs que pertenece al objetivo — información muy útil para la fase activa posterior.

### Subdominios — La superficie de ataque oculta

Los subdominios son probablemente el hallazgo más valioso en la fase de reconocimiento pasivo. Cada subdominio es potencialmente un servicio distinto, con su propio stack tecnológico, sus propias versiones y sus propias vulnerabilidades.

La razón por la que los subdominios son tan valiosos es que las empresas tienden a ser más descuidadas con ellos que con el dominio principal. El sitio web principal suele estar muy revisado, con actualizaciones frecuentes y revisiones de seguridad. Pero `dev.empresa.com`, `staging.empresa.com`, `old-panel.empresa.com` o `jenkins.empresa.com` a menudo se despliegan rápido, se olvidan y nunca se revisan. Son los puntos ciegos de seguridad de cualquier organización.

**Subfinder** es la herramienta más rápida para descubrimiento pasivo de subdominios. Consulta decenas de fuentes públicas simultáneamente: VirusTotal, SecurityTrails, crt.sh, Shodan, HackerTarget, entre otras.

```bash
# Búsqueda básica y silenciosa (sin output extra)
subfinder -d ejemplo.com -silent -o subdominios_subfinder.txt
```

Este comando consulta todas las fuentes configuradas y guarda los subdominios únicos en el archivo. El flag `-silent` suprime el banner y mensajes de progreso, lo que facilita procesar el output en scripts.

```bash
# Modo exhaustivo: consulta todas las fuentes disponibles
subfinder -d ejemplo.com -all -o subdominios_all.txt

# Modo recursivo: busca subdominios de subdominios
# Útil cuando encuentras cosas como *.dev.ejemplo.com
subfinder -d ejemplo.com -recursive -o subdominios_recursivo.txt

# Mostrar la fuente de cada subdominio encontrado
# Útil para saber qué fuente da más resultados
subfinder -d ejemplo.com -sources -o subdominios_fuentes.txt
```

**Amass** es más exhaustivo que Subfinder pero también más lento. Hace consultas a más fuentes y puede hacer resolución DNS activa (en modo pasivo solo consulta fuentes externas).

```bash
# Modo estrictamente pasivo (no contacta con el objetivo)
amass enum -passive -d ejemplo.com -o subdominios_amass.txt

# Mostrar la fuente de cada subdominio para entender de dónde vienen
amass enum -passive -d ejemplo.com -src -o amass_con_fuentes.txt

# Guardar el grafo de relaciones (útil para visualizar en Maltego)
amass enum -passive -d ejemplo.com -o amass.txt -oA amass_graph
```

Una vez tienes los resultados de ambas herramientas, combínalos y elimina duplicados. La suma de las dos herramientas siempre da más subdominios que cualquiera de las dos por separado porque cada una tiene acceso a fuentes distintas.

```bash
# Combinar ambos outputs y deduplicar
cat subdominios_subfinder.txt subdominios_amass.txt | sort -u > todos_subdominios.txt

# Ver cuántos encontramos en total
wc -l todos_subdominios.txt
```

### Certificate Transparency — Subdominios desde certificados SSL

Quizás la fuente más fiable y menos conocida de subdominios son los **Certificate Transparency logs**. Cuando cualquier CA (Certificate Authority) emite un certificado SSL/TLS, está obligada por el estándar RFC 6962 a registrarlo públicamente en logs de CT. Esto es una medida de seguridad del ecosistema web, pero para nosotros como investigadores es una mina de oro.

La razón por la que los CT logs son tan valiosos es que capturan subdominios que nunca aparecen en búsquedas web ni en DNS público — porque a veces son servicios internos expuestos accidentalmente, entornos de staging, o incluso servicios que ya no existen pero cuyos certificados siguen en el log.

[crt.sh](https://crt.sh) es la interfaz más popular para consultar estos logs. Puedes usarlo desde el navegador o desde la línea de comandos:

```bash
# Consulta directa a crt.sh via API (devuelve JSON)
curl -s "https://crt.sh/?q=%.ejemplo.com&output=json" \
  | jq -r '.[].name_value' \
  | sed 's/\*\.//g' \
  | sort -u > subdominios_crtsh.txt
```

Lo que hace este comando es: consultar todos los certificados emitidos para cualquier subdominio de `ejemplo.com` (el `%` es el wildcard de SQL que usa crt.sh), extraer el campo `name_value` que contiene el nombre del dominio, eliminar los wildcards `*.` que a veces aparecen, y ordenar/deduplicar.

A veces los certificados Wildcard (`*.ejemplo.com`) no revelan subdominios específicos, pero los certificados SAN (Subject Alternative Names) sí — porque un solo certificado puede cubrir decenas de subdominios y todos quedan registrados en el log.

---

## 4. Recolección de emails

Los emails corporativos son uno de los activos más valiosos en reconocimiento porque abren múltiples vectores de ataque: phishing dirigido (spear phishing), password spraying contra servicios como Office 365 o Gmail, o simplemente para confirmar que una cuenta existe antes de intentar acceder.

La clave no es solo encontrar emails — es encontrar suficientes para **inferir el patrón** que usa la organización. Una vez sabes que el patrón es `nombre.apellido@empresa.com`, puedes generar emails válidos para cualquier empleado que encuentres en LinkedIn, aunque nunca hayas visto su email publicado.

### theHarvester — La navaja suiza de la recolección de emails

theHarvester está incluido en Kali Linux y es la herramienta estándar para esta tarea. Su funcionamiento se basa en consultar múltiples motores de búsqueda y fuentes de datos simultáneamente, buscando menciones del dominio objetivo. Cuando un motor de búsqueda tiene indexado un documento que contiene un email de la empresa, theHarvester lo extrae.

```bash
# Búsqueda básica con los motores más comunes
theHarvester -d ejemplo.com -b google,bing,duckduckgo -l 500
```

El flag `-l 500` limita a 500 resultados por fuente. Para un reconocimiento exhaustivo puedes subir este número, aunque más resultados significa más tiempo de ejecución y más probabilidad de ser bloqueado por rate limiting de los motores de búsqueda.

```bash
# Usar todos los motores disponibles (más lento pero más completo)
theHarvester -d ejemplo.com -b all -l 300

# Guardar los resultados en HTML y XML para el reporte
theHarvester -d ejemplo.com -b google,bing -l 500 -f recon_emails

# Incluir Shodan para obtener también hosts expuestos
# (requiere API key de Shodan configurada)
theHarvester -d ejemplo.com -b google,shodan -l 500
```

El output de theHarvester te da dos tipos de información: emails encontrados y hosts descubiertos. Los hosts son subdominios que los motores de búsqueda tienen indexados — complementan perfectamente los resultados de Subfinder.

### Identificar el patrón de emails corporativos

Una vez tienes una lista de emails, el siguiente paso es identificar el patrón. Las empresas son consistentes — todos los empleados tienen el mismo formato de email. Si encuentras `jsmith@empresa.com` y `carlos.garcia@empresa.com`, ya sabes que el patrón es `nombre.apellido@empresa.com` o `inicialNombre+apellido@empresa.com`.

Los patrones más comunes en empresas españolas e internacionales son:

```
nombre.apellido@empresa.com        →  carlos.garcia@empresa.com
inicialNombre+apellido@empresa.com →  cgarcia@empresa.com
nombre_apellido@empresa.com        →  carlos_garcia@empresa.com
nombre@empresa.com                 →  carlos@empresa.com  (empresas pequeñas)
apellido@empresa.com               →  garcia@empresa.com
```

Con el patrón identificado, puedes usar este script para generar emails de cualquier empleado que encuentres en LinkedIn:

```python
#!/usr/bin/env python3
# generate_emails.py — Genera emails corporativos desde lista de nombres

# Lista de empleados obtenida de LinkedIn (nombre completo)
empleados = [
    "Carlos García López",
    "María Fernández Ruiz",
    "Javier Martínez Sánchez",
    "Ana López Gómez",
]

# Dominio y patrón de la empresa objetivo
DOMINIO = "empresa.com"
PATRON = "nombre.apellido"  # Cambiar según lo que hayas identificado

def generar_email(nombre_completo, dominio, patron):
    """Genera email según el patrón identificado."""
    # Normalizar: minúsculas, quitar acentos básicos
    partes = nombre_completo.lower().split()
    partes = [p.replace('á','a').replace('é','e').replace('í','i')
               .replace('ó','o').replace('ú','u').replace('ñ','n')
               for p in partes]

    nombre = partes[0]
    apellido = partes[1] if len(partes) > 1 else ""
    inicial = nombre[0] if nombre else ""

    patrones = {
        "nombre.apellido":    f"{nombre}.{apellido}@{dominio}",
        "inicialNombre+apellido": f"{inicial}{apellido}@{dominio}",
        "nombre_apellido":    f"{nombre}_{apellido}@{dominio}",
        "nombre":             f"{nombre}@{dominio}",
        "apellido":           f"{apellido}@{dominio}",
    }
    return patrones.get(patron, f"{nombre}.{apellido}@{dominio}")

print(f"[+] Generando emails con patrón: {PATRON}")
print(f"[+] Dominio: {DOMINIO}\n")

for empleado in empleados:
    email = generar_email(empleado, DOMINIO, PATRON)
    print(email)
```

```bash
python3 generate_emails.py > emails_generados.txt
cat emails_generados.txt
```

---

## 5. Shodan — El buscador de dispositivos expuestos

Shodan merece una explicación especial porque mucha gente no entiende bien cómo funciona y, por tanto, no lo usa correctamente. Shodan **no escanea el objetivo cuando haces una búsqueda**. Shodan tiene sus propios crawlers que escanean internet continuamente (los 65535 puertos de todas las IPs públicas) y almacena los resultados en su base de datos. Cuando tú haces una búsqueda, estás consultando esa base de datos, no tocando el objetivo.

Esto tiene dos implicaciones importantes: primero, la información puede tener horas o días de antigüedad (aunque Shodan actualiza con bastante frecuencia los rangos más populares). Segundo, desde el punto de vista del objetivo, tu búsqueda en Shodan es completamente invisible.

Lo que Shodan almacena de cada IP es el banner que devuelve cada servicio — el texto de respuesta inicial que envía el servidor cuando alguien se conecta. Un servidor HTTP devuelve sus cabeceras; un servidor SSH devuelve su versión; una base de datos expuesta devuelve su mensaje de bienvenida. De estos banners Shodan extrae información sobre el producto, la versión, el sistema operativo, los certificados SSL y mucho más.

### Búsquedas esenciales para reconocimiento de una empresa

La búsqueda más directa es por organización. El campo `org:` en Shodan filtra por el nombre de la organización registrada en el bloque de IPs (campo que viene del WHOIS de esas IPs).

```
org:"Nombre de la Empresa"
```

Es importante poner el nombre entre comillas para búsquedas exactas, y probar variantes — a veces el nombre registrado en WHOIS es ligeramente diferente al nombre comercial. Por ejemplo, una empresa llamada "Acme Corp" puede estar registrada como "Acme Corporation S.A." o "Acme Technologies".

Una vez localizas la organización, puedes combinar filtros para afinar. Por ejemplo, para ver todos los servidores web que tiene expuestos:

```
org:"Nombre de la Empresa" port:80,443
```

O para ver específicamente los servicios SSH (que podrían ser accesibles y son objetivo de ataques de fuerza bruta):

```
org:"Nombre de la Empresa" port:22
```

El filtro `hostname:` es especialmente útil cuando conoces el dominio del objetivo, porque Shodan resuelve hostnames y los incluye en sus resultados:

```
hostname:ejemplo.com
```

Esto muestra todos los servicios de todos los subdominios de `ejemplo.com` que Shodan ha encontrado. Es una forma muy rápida de ver toda la superficie de ataque expuesta públicamente.

### Shodan desde la línea de comandos

Trabajar desde CLI es más eficiente cuando necesitas procesar los resultados o integrarlos en un script de reconocimiento:

```bash
# Primero instalar el cliente y autenticarse
pip3 install shodan
shodan init TU_API_KEY
```

La API key gratuita de Shodan tiene límites, pero para reconocimiento básico es suficiente. El plan de pago desbloquea filtros más avanzados y más resultados.

```bash
# Búsqueda básica por organización
shodan search 'org:"Empresa Objetivo"'
```

El output por defecto es bastante verboso. Para procesarlo mejor, especifica los campos que te interesan:

```bash
# Mostrar solo IP, puerto, organización y hostname
shodan search --fields ip_str,port,org,hostname 'org:"Empresa Objetivo"'

# Guardar todos los resultados en un fichero para análisis posterior
shodan search --limit 1000 --fields ip_str,port,transport,product,version \
  'org:"Empresa Objetivo"' > shodan_resultados.txt
```

Para obtener información detallada de una IP específica que hayas identificado (por ejemplo, la IP del servidor de correo del objetivo), usa el subcomando `host`:

```bash
# Información completa de una IP: todos los puertos, banners, historial
shodan host 93.184.216.34
```

Este comando devuelve todos los puertos que Shodan ha visto abiertos en esa IP, con sus banners completos, el historial de cambios, los certificados SSL, etc. Es como hacer un escaneo de puertos pero sin tocar la IP.

### Encontrar tecnologías vulnerables

Shodan es especialmente potente cuando buscas versiones específicas de software que tienen vulnerabilidades conocidas. Por ejemplo, si sabes que Log4Shell (CVE-2021-44228) afecta a una versión concreta de Log4j, puedes buscar todos los servidores del objetivo que usen Java:

```
org:"Empresa Objetivo" product:"Apache Tomcat"
org:"Empresa Objetivo" product:"Apache httpd" version:"2.4.49"
```

Shodan también tiene un filtro `vuln:` que directamente filtra IPs con vulnerabilidades conocidas detectadas:

```
org:"Empresa Objetivo" vuln:CVE-2021-44228
org:"Empresa Objetivo" vuln:CVE-2017-0144
```

Ojo: este filtro requiere cuenta de pago en Shodan, pero es extremadamente potente porque te dice directamente qué IPs de tu objetivo son vulnerables a CVEs concretos.

---

## 6. Metadatos de documentos — La información que nadie ve

Cuando una empresa publica documentos en su web — informes anuales, presentaciones, contratos de ejemplo, manuales de producto — comete un error que muy poca gente considera: esos documentos llevan metadatos. Los metadatos son información sobre el archivo que no se ve al abrirlo normalmente, pero que está ahí y es extraíble con herramientas simples.

El tipo de información que puedes encontrar en los metadatos es sorprendentemente revelador:

**En documentos de Office (Word, Excel, PowerPoint):** El nombre del autor, el nombre de usuario del sistema operativo con el que se creó el archivo, la empresa registrada en el software, el nombre del equipo, rutas internas completas (por ejemplo `C:\Users\jsmith\Documents\Proyectos\Informe_Q3.docx`), el servidor de impresión de la empresa, la versión exacta de Office utilizada y los timestamps de creación y modificación.

**En imágenes (JPG, PNG, TIFF):** Coordenadas GPS exactas donde se tomó la foto, el modelo y marca de la cámara o teléfono, la versión del firmware, fecha y hora, y ajustes de la cámara.

**En PDFs:** Herramienta con la que se generó (Adobe Acrobat, LibreOffice, Word), autor, empresa, título del documento, y a veces rastros del servidor donde se generó.

Esta información es oro puro para un atacante. Las rutas internas revelan nombres de usuario del sistema (`jsmith` en `C:\Users\jsmith\`) que probablemente sean los mismos que en Active Directory. El nombre del equipo (`CORP-PC-JSMITH`) puede estar en la red interna. La versión de Office revela qué nivel de parcheo tienen los equipos de usuario final.

### Recolectar documentos con Google Dorks

El primer paso es encontrar documentos públicos del objetivo. Google los tiene indexados — solo hay que saber pedirlos:

```
site:ejemplo.com filetype:pdf
```

Este dork le dice a Google que muestre solo PDFs del dominio `ejemplo.com`. Puedes ampliarlo para otros formatos:

```
site:ejemplo.com filetype:doc OR filetype:docx
site:ejemplo.com filetype:xls OR filetype:xlsx
site:ejemplo.com filetype:ppt OR filetype:pptx
```

Una vez localizados, descárgalos todos antes de analizarlos (a veces los van retirando cuando detectan que alguien los está mirando):

```bash
# Descargar todos los PDFs del dominio
wget -r -l 2 -A pdf "https://ejemplo.com" -P ./documentos/
```

### Exiftool — Extracción de metadatos

Exiftool es la herramienta universal para leer, escribir y editar metadatos de prácticamente cualquier formato de archivo. Es de código abierto, está disponible en todos los sistemas operativos y es la referencia en análisis forense de metadatos.

```bash
# Instalar en sistemas basados en Debian/Ubuntu
apt install exiftool

# Analizar un único archivo — muestra todos los campos disponibles
exiftool documento.pdf
```

El output puede ser largo. Para un PDF típico de una empresa verás decenas de campos, pero los que más nos interesan son unos pocos específicos:

```bash
# Extraer solo los campos más relevantes para reconocimiento
exiftool -Author -Creator -CreatorTool -Producer \
         -LastModifiedBy -Company -CreationDate \
         documento.pdf
```

Cada campo tiene su significado:
- `Author`: Nombre del autor — puede ser un nombre real de empleado
- `Creator`: Software que creó el documento (`Microsoft Word 2016`)
- `LastModifiedBy`: Usuario que hizo la última modificación — nombre de usuario del sistema
- `Company`: Empresa registrada en el software de Office
- `CreatorTool`: En PDFs, la herramienta que lo generó

Para procesar un directorio entero de documentos de una vez y exportar a CSV (mucho más manejable para análisis):

```bash
# Analizar todos los archivos de un directorio y exportar a CSV
exiftool -csv ./documentos/ > metadatos_todos.csv

# Ver el CSV con columnas legibles
column -t -s ',' metadatos_todos.csv | less -S
```

El CSV te permite ordenar por columna — por ejemplo, ordenar por `Author` para ver todos los autores únicos, o por `Creator` para ver qué versiones de software usan.

Para las imágenes, los datos GPS son particularmente interesantes:

```bash
# Extraer coordenadas GPS de una imagen
exiftool -GPSLatitude -GPSLongitude foto.jpg

# En formato decimal (más fácil de pegar en Google Maps)
exiftool -n -GPSLatitude -GPSLongitude foto.jpg
```

Con esas coordenadas puedes ir directamente a Google Maps y ver exactamente dónde se tomó la foto — a veces revela la ubicación de las oficinas, el datacenter, o información sobre dónde trabajan los empleados.

---

## 7. Brechas de datos — Credenciales filtradas

Una de las fuentes de inteligencia más potentes y más ignoradas son las bases de datos de brechas de seguridad. Cada vez que una empresa sufre un data breach y los datos se filtran, esas credenciales quedan disponibles en internet — a veces en la dark web, a veces en foros públicos, a veces en repositorios de GitHub.

El valor de esta información es enorme: si un empleado de la empresa objetivo usó la misma contraseña en un servicio que fue comprometido hace tres años, es probable que esa misma contraseña (o una variante) siga siendo válida para sus cuentas corporativas. La reutilización de contraseñas es uno de los problemas de seguridad más persistentes en cualquier organización.

### Have I Been Pwned

[HIBP](https://haveibeenpwned.com) es la base de datos de brechas más conocida, creada por Troy Hunt. Tiene más de 12 mil millones de cuentas comprometidas de cientos de brechas distintas.

Para búsquedas individuales puedes usar la web directamente, pero para escalar la búsqueda a todos los emails de un dominio necesitas la API:

```bash
# Verificar un email individual (sin API key)
curl -s "https://haveibeenpwned.com/api/v3/breachedaccount/usuario@ejemplo.com" \
  -H "hibp-api-key: TU_KEY"
```

La respuesta te dice en qué brechas aparece ese email y qué tipo de datos fueron comprometidos (contraseñas, emails, nombres, tarjetas, etc.).

### Dehashed — Búsqueda con contraseñas

Dehashed va un paso más allá que HIBP: no solo te dice que un email fue comprometido, sino que muestra los datos reales filtrados, incluyendo contraseñas en texto plano o hashes. Esto lo hace mucho más potente para reconocimiento ofensivo:

```bash
# Buscar todos los registros asociados a un dominio
curl -H "Accept: application/json" \
     -u "tu_email:tu_api_key" \
     "https://api.dehashed.com/search?query=domain:ejemplo.com&size=100"
```

El resultado puede incluir filas como:
```json
{
  "email": "jsmith@ejemplo.com",
  "username": "jsmith",
  "password": "P@ssw0rd2019",
  "hashed_password": "...",
  "database_name": "SomeLeakedDB_2020"
}
```

Cuando encuentras contraseñas en texto plano de empleados de la empresa objetivo, el siguiente paso analítico es buscar patrones: ¿usan el año en la contraseña? ¿El nombre de la empresa? ¿Una palabra común más números? Estos patrones te sirven para construir reglas de Hashcat si necesitas crackear otros hashes, o para generar variantes de contraseñas en un ataque de password spraying.

---

## 8. Script de reconocimiento automatizado

Para terminar, aquí tienes un script que automatiza las fases principales de reconocimiento pasivo. El script es modular — puedes comentar las secciones que no necesites o para las que no tengas herramientas instaladas:

```bash
#!/bin/bash
# recon_pasivo.sh — Reconocimiento pasivo automatizado
# Uso: ./recon_pasivo.sh ejemplo.com

TARGET=$1
OUTDIR="recon_${TARGET}_$(date +%Y%m%d)"

# Verificar que se pasó un argumento
if [ -z "$TARGET" ]; then
    echo "[!] Uso: $0 <dominio>"
    echo "[!] Ejemplo: $0 empresa.com"
    exit 1
fi

# Crear estructura de directorios
mkdir -p "$OUTDIR"/{subdominios,emails,whois,wayback,metadatos}
echo "[*] Reconocimiento de: $TARGET"
echo "[*] Output en: $OUTDIR"
echo "[*] Inicio: $(date)"
echo ""

# ── 1. WHOIS ──────────────────────────────────────────────────
echo "[+] Obteniendo WHOIS..."
whois $TARGET > "$OUTDIR/whois/whois_dominio.txt" 2>/dev/null
echo "    Guardado en whois/whois_dominio.txt"

# ── 2. SUBDOMINIOS ────────────────────────────────────────────
echo "[+] Enumerando subdominios con subfinder..."
# -silent evita el banner; -all usa todas las fuentes disponibles
subfinder -d $TARGET -silent -all -o "$OUTDIR/subdominios/subfinder.txt" 2>/dev/null
echo "    Subfinder: $(wc -l < $OUTDIR/subdominios/subfinder.txt) subdominios"

echo "[+] Enumerando subdominios con amass (pasivo)..."
# -passive garantiza que no contactamos con el objetivo
amass enum -passive -d $TARGET -o "$OUTDIR/subdominios/amass.txt" 2>/dev/null
echo "    Amass: $(wc -l < $OUTDIR/subdominios/amass.txt) subdominios"

echo "[+] Consultando Certificate Transparency logs (crt.sh)..."
# Descargamos los certificados en JSON y extraemos los hostnames únicos
curl -s "https://crt.sh/?q=%.${TARGET}&output=json" 2>/dev/null \
    | jq -r '.[].name_value' \
    | sed 's/\*\.//g' \
    | sort -u > "$OUTDIR/subdominios/crtsh.txt"
echo "    crt.sh: $(wc -l < $OUTDIR/subdominios/crtsh.txt) subdominios"

# Combinar todos y deduplicar
cat "$OUTDIR/subdominios/"*.txt | sort -u > "$OUTDIR/subdominios/TODOS.txt"
TOTAL_SUBS=$(wc -l < "$OUTDIR/subdominios/TODOS.txt")
echo "    Total único: $TOTAL_SUBS subdominios"

# ── 3. EMAILS ─────────────────────────────────────────────────
echo "[+] Recolectando emails con theHarvester..."
# -b especifica los motores; -l limita resultados; -f guarda en HTML y XML
theHarvester -d $TARGET -b google,bing,duckduckgo -l 300 \
    -f "$OUTDIR/emails/harvester" > "$OUTDIR/emails/harvester_stdout.txt" 2>/dev/null

# Extraer solo los emails del output
grep -oE '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' \
    "$OUTDIR/emails/harvester_stdout.txt" | sort -u > "$OUTDIR/emails/emails.txt"
echo "    Emails: $(wc -l < $OUTDIR/emails/emails.txt) encontrados"

# ── 4. WAYBACK MACHINE ────────────────────────────────────────
echo "[+] Obteniendo URLs históricas de Wayback Machine..."
# La API CDX de archive.org devuelve todas las URLs capturadas
curl -s "http://web.archive.org/cdx/search/cdx?url=${TARGET}/*&output=text&fl=original&collapse=urlkey" \
    2>/dev/null | sort -u > "$OUTDIR/wayback/wayback_urls.txt"
WAYBACK=$(wc -l < "$OUTDIR/wayback/wayback_urls.txt")
echo "    URLs históricas: $WAYBACK"

# Filtrar URLs interesantes (APIs, paneles admin, configs)
grep -iE "api|admin|login|config|backup|\.env|\.sql|\.bak" \
    "$OUTDIR/wayback/wayback_urls.txt" > "$OUTDIR/wayback/wayback_interesantes.txt"
echo "    URLs interesantes: $(wc -l < $OUTDIR/wayback/wayback_interesantes.txt)"

# ── 5. RESUMEN FINAL ──────────────────────────────────────────
echo ""
echo "╔══════════════════════════════════════════╗"
echo "║         RECONOCIMIENTO COMPLETADO        ║"
echo "╠══════════════════════════════════════════╣"
printf "║  %-15s %-25s ║\n" "Subdominios:" "$TOTAL_SUBS encontrados"
printf "║  %-15s %-25s ║\n" "Emails:" "$(wc -l < $OUTDIR/emails/emails.txt) encontrados"
printf "║  %-15s %-25s ║\n" "URLs hist.:" "$WAYBACK encontradas"
printf "║  %-15s %-25s ║\n" "Directorio:" "$OUTDIR"
printf "║  %-15s %-25s ║\n" "Fin:" "$(date +%H:%M:%S)"
echo "╚══════════════════════════════════════════╝"
```

Para usarlo simplemente dale permisos de ejecución y pásale el dominio:

```bash
chmod +x recon_pasivo.sh
./recon_pasivo.sh empresa.com
```

El script creará un directorio con fecha (por ejemplo `recon_empresa.com_20250513`) con toda la información organizada por categorías. Cada vez que ejecutes el script contra el mismo objetivo se crea un directorio nuevo con la fecha, lo que te permite ver la evolución de la infraestructura a lo largo del tiempo.

---

## Cheatsheet de referencia rápida

```bash
# ── DOMINIOS Y SUBDOMINIOS ──────────────────────────────────
whois ejemplo.com                                      # Info de registro
subfinder -d ejemplo.com -silent -o subs.txt           # Subdominios rápido
amass enum -passive -d ejemplo.com -o amass.txt        # Subdominios exhaustivo
curl -s "https://crt.sh/?q=%.ejemplo.com&output=json" \
  | jq -r '.[].name_value' | sort -u                  # CT logs

# ── EMAILS ──────────────────────────────────────────────────
theHarvester -d ejemplo.com -b all -l 500              # Recolección masiva
# https://hunter.io                                    # Emails corporativos
# https://haveibeenpwned.com                           # Check en brechas
# https://dehashed.com                                 # Brechas con passwords

# ── SHODAN ──────────────────────────────────────────────────
shodan search 'org:"Empresa Objetivo"'                 # Por organización
shodan search 'hostname:ejemplo.com'                   # Por hostname
shodan host 93.184.216.34                              # Info detallada de IP
# org:"Empresa" port:22                               # SSH expuesto
# ssl.cert.subject.CN:ejemplo.com                     # Por certificado SSL

# ── METADATOS ───────────────────────────────────────────────
exiftool documento.pdf                                 # Ver todos los metadatos
exiftool -Author -Creator -LastModifiedBy doc.pdf      # Solo campos clave
exiftool -csv ./directorio/ > metadatos.csv           # Batch a CSV
exiftool -n -GPSLatitude -GPSLongitude foto.jpg       # GPS en decimal

# ── WAYBACK / URLS HISTÓRICAS ───────────────────────────────
echo "ejemplo.com" | waybackurls > wayback.txt         # URLs históricas
gau ejemplo.com > all_urls.txt                         # Get All URLs
curl -s "http://web.archive.org/cdx/search/cdx?url=ejemplo.com/*&output=text&fl=original&collapse=urlkey"


```

## FRAMEWORK Y RECURSOS
- **Google Hacking Database** — [exploit-db.com/google-hacking-database](https://www.exploit-db.com/google-hacking-database)
- **Mapa De Herramientas OSINT** - [osintframework.com](https://osintframework.com)
https://crt.sh              → Certificate Transparency logs
https://shodan.io           → dispositivos expuestos en internet
https://censys.io           → alternativa a Shodan, más enfocada a certs
https://dehashed.com        → brechas de datos con passwords
https://intelx.io           → paste sites + indexación dark web
https://hunter.io           → emails corporativos verificados
https://bgp.he.net          → ASN y rangos de IP por organización
https://securitytrails.com  → historial DNS y subdominios
https://viewdns.info        → múltiples herramientas DNS/IP
