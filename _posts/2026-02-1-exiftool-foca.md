---
layout: post
title: "Metadata Extraction — Exiftool y FOCA"
date: 2026-02-01
categories: [reconocimiento]
tags: [exiftool, foca, metadatos, osint, reconocimiento, documentos, pdf, office, imagenes]
description: "Guía profesional de extracción de metadatos con Exiftool y FOCA: qué revelan los documentos corporativos, cómo extraerlos, analizarlos y usarlos en reconocimiento de infraestructura y personas."
---

## Los metadatos como ventana a la infraestructura interna

Hay una paradoja curiosa en la seguridad corporativa: las empresas invierten enormes recursos en firewalls, VPNs, y controles de acceso, pero luego publican en su web pública documentos que contienen rutas completas de sus servidores internos, nombres de usuario de Active Directory, versiones exactas del software que usan, y a veces hasta la estructura de sus carpetas compartidas en red.

Los metadatos son información sobre la información. Cuando un empleado guarda un documento Word, el software registra automáticamente quién lo creó, en qué equipo, cuándo, con qué versión del software, y en qué ruta del sistema de archivos estaba guardado. Cuando alguien sube ese documento a la web corporativa, todos esos datos internos van incluidos. Nadie los ve al abrir el documento normalmente, pero están ahí, accesibles para cualquier herramienta que sepa extraerlos.

En un engagement de reconocimiento pasivo, los metadatos de documentos públicos pueden revelar:

- **Nombres de usuario de Active Directory** — exactamente como están escritos en el dominio
- **Rutas de red internas** — `\\fileserver-01\departamentos\finanzas\contratos\`
- **Nombres de equipos** — `CORP-LAPTOP-JSMITH-01`, que revelan el naming convention
- **Versiones exactas de software** — qué versión de Office usan, qué printer drivers, qué PDF creator
- **Estructura organizativa** — los nombres de carpetas revelan departamentos y jerarquías
- **Proveedores y contratistas** — los documentos de terceros que la empresa re-publica
- **Coordenadas GPS** — las fotografías tomadas con móvil guardan la ubicación exacta
- **Timestamps** — revelan horarios de trabajo, zonas horarias, y cuándo se trabaja realmente

Todo esto sin enviar un solo paquete al objetivo. Es reconocimiento completamente pasivo.

---

## 1. Qué son los metadatos y dónde se esconden

Antes de usar ninguna herramienta, conviene entender exactamente qué tipos de metadatos existen y en qué formatos de archivo se encuentran. No todos los archivos tienen los mismos metadatos, y la riqueza de información varía enormemente según el tipo.

### Metadatos en documentos de Office (DOC, DOCX, XLS, XLSX, PPT, PPTX)

Los documentos de Microsoft Office son los más ricos en metadatos corporativos. El formato DOCX es en realidad un ZIP que contiene varios archivos XML, y uno de ellos (`docProps/core.xml`) almacena los metadatos principales:

```xml
<!-- Ejemplo de core.xml en un DOCX corporativo -->
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<cp:coreProperties>
  <dc:title>Informe Financiero Q3 2024</dc:title>
  <dc:subject>Resultados trimestrales</dc:subject>
  <dc:creator>John Smith</dc:creator>
  <cp:lastModifiedBy>María García</cp:lastModifiedBy>
  <cp:revision>14</cp:revision>
  <dcterms:created>2024-09-15T09:23:00Z</dcterms:created>
  <dcterms:modified>2024-10-02T16:47:00Z</dcterms:modified>
</cp:coreProperties>
```

Y otro archivo (`docProps/app.xml`) contiene metadatos de la aplicación:

```xml
<Properties>
  <Application>Microsoft Office Word</Application>
  <AppVersion>16.0000</AppVersion>  <!-- Office 2016/365 -->
  <Company>Example Corp</Company>
  <Template>Normal</Template>
  <TotalTime>847</TotalTime>         <!-- minutos de edición -->
  <Pages>12</Pages>
  <Words>3847</Words>
</Properties>
```

Un tercer archivo puede contener rutas absolutas del sistema si el documento tiene recursos incrustados (imágenes, objetos OLE):

```
word/media/image1.png  → ruta: C:\Users\jsmith\Desktop\logo_empresa.png
```

### Metadatos en PDFs

Los PDFs tienen sus propios metadatos en dos lugares:

**Document Information Dictionary** — el formato antiguo, visible con cualquier visor:

```
Title:          Contrato de Servicios 2024
Author:         jsmith
Subject:        Acuerdo Marco
Keywords:       confidencial, interno
Creator:        Microsoft Word 2016
Producer:       Microsoft: Print To PDF
CreationDate:   D:20240915092300+02'00'
ModDate:        D:20241002164700+02'00'
```

**XMP Metadata** — el formato moderno, más detallado y estructurado, que puede incluir el historial completo de ediciones, los programas que procesaron el documento, y metadatos personalizados.

Lo más revelador en PDFs corporativos suele ser el campo **Creator** (qué software generó el documento original) y el campo **Producer** (qué software lo convirtió a PDF), que juntos dicen exactamente qué stack de software usa la empresa.

### Metadatos en imágenes (JPG, PNG, TIFF)

Las imágenes tienen tres tipos de metadatos:

**EXIF** (Exchangeable Image File Format) — el más interesante para reconocimiento:

```
Camera Make:      Apple
Camera Model:     iPhone 14 Pro
Software:         17.1.1
Date/Time:        2024:09:15 14:23:47
GPS Latitude:     40° 25' 46.80" N
GPS Longitude:    3° 41' 33.48" W
GPS Altitude:     667 m Above Sea Level
GPS Speed:        0 km/h
```

Las coordenadas GPS en una fotografía de la fachada del edificio, o tomada desde la ventana de la oficina, revelan la ubicación exacta de las instalaciones de la empresa. Las imágenes tomadas en eventos corporativos o ferias pueden revelar la ubicación de stand, almacenes o centros de datos.

**IPTC** — metadatos editoriales (caption, copyright, créditos del fotógrafo).

**XMP** — metadatos extensibles que pueden contener información personalizada del software de edición.

---

## 2. Exiftool — La navaja suiza de los metadatos

Exiftool es la herramienta de referencia absoluta para lectura, escritura y edición de metadatos. Fue creada por Phil Harvey y soporta más de 22.000 tags en cientos de formatos de archivo distintos. No tiene interfaz gráfica — es una herramienta de línea de comandos, lo que la hace perfecta para automatización y procesamiento en batch.

Lo que hace especial a Exiftool no es solo su cobertura de formatos sino su capacidad para leer metadatos que otros programas ignoran: streams de datos alternativos en NTFS, metadatos XMP embebidos, bloques IPTC, datos EXIF en RAW de cámara, y docenas de formatos propietarios.

### Instalación

```bash
# En Kali Linux y distribuciones basadas en Debian
sudo apt update && sudo apt install exiftool

# En macOS con Homebrew
brew install exiftool

# En cualquier sistema con Perl instalado (Exiftool está escrito en Perl)
# Descarga desde https://exiftool.org y ejecuta el script directamente
wget https://exiftool.org/Image-ExifTool-12.70.tar.gz
tar -xzf Image-ExifTool-12.70.tar.gz
cd Image-ExifTool-12.70
sudo cp exiftool /usr/local/bin/
sudo cp -r lib /usr/local/lib/

# Verificar instalación
exiftool -version
```

### Extracción básica de metadatos

La forma más simple de usar Exiftool es pasarle un archivo y ver todo lo que devuelve:

```bash
# Analizar un archivo y mostrar todos sus metadatos
# El output puede ser muy largo — Exiftool extrae absolutamente todo
exiftool documento.pdf

# La salida tendrá este aspecto:
# ExifTool Version Number         : 12.70
# File Name                       : documento.pdf
# Directory                       : .
# File Size                       : 2.3 MB
# File Type                       : PDF
# PDF Version                     : 1.7
# Author                          : jsmith
# Creator                         : Microsoft Word 2019
# Producer                        : Microsoft: Print To PDF
# Create Date                     : 2024:09:15 09:23:00+02:00
# Modify Date                     : 2024:10:02 16:47:00+02:00
# Title                           : Informe Q3
# Company                         : Example Corp

# Para una imagen con GPS:
exiftool foto_oficina.jpg
# GPS Latitude                    : 40 deg 25' 46.80" N
# GPS Longitude                   : 3 deg 41' 33.48" W
# GPS Altitude                    : 667 m Above Sea Level
# Camera Model Name               : iPhone 14 Pro
# Date/Time Original              : 2024:09:15 14:23:47
```

### Extraer solo los campos que nos interesan

En reconocimiento queremos información específica, no todos los 200 campos técnicos que Exiftool puede devolver. Podemos filtrar exactamente los campos relevantes:

```bash
# Extraer solo los campos más útiles para reconocimiento de personas e infraestructura
# El guión antes del nombre del campo selecciona solo ese campo
exiftool -Author -Creator -LastModifiedBy -Company \
         -Producer -CreatorTool -Template documento.docx

# Para imágenes: coordenadas GPS en formato legible
exiftool -GPSLatitude -GPSLongitude -GPSAltitude -DateTimeOriginal \
         -Make -Model foto.jpg

# Coordenadas en formato decimal (más fácil para Google Maps)
# La opción -n devuelve valores numéricos en lugar de texto formateado
exiftool -n -GPSLatitude -GPSLongitude foto.jpg
# GPS Latitude  : 40.4296667
# GPS Longitude : -3.6926333
# → Pegar en Google Maps: 40.4296667,-3.6926333

# Buscar rutas de sistema de archivos — muy revelador en documentos de Office
exiftool -HyperlinkBase -Directory -SourceFile documento.docx

# Para PDFs: el campo XMPToolkit puede revelar el software completo de procesamiento
exiftool -XMPToolkit -PDFVersion -Linearized documento.pdf
```

### Procesamiento en batch — múltiples archivos

El reconocimiento real implica analizar decenas o cientos de documentos descargados del dominio objetivo. Exiftool está diseñado para esto y procesa archivos en batch con mucha eficiencia:

```bash
# Analizar todos los PDFs del directorio actual
exiftool *.pdf

# Analizar todos los archivos de un directorio recursivamente
# -r entra en subdirectorios
exiftool -r /ruta/documentos/

# Exportar metadatos de todos los archivos a CSV
# Perfecto para analizar en Excel o importar en scripts Python
exiftool -csv /ruta/documentos/ > metadatos_completos.csv

# CSV con solo los campos que nos interesan
exiftool -csv -Author -Creator -LastModifiedBy -Company \
         -CreatorTool -Producer *.docx *.pdf > metadatos_relevantes.csv

# Procesar solo ciertos tipos de archivo en un directorio mixto
exiftool -ext pdf -ext docx -ext xlsx /ruta/documentos/ -csv > metadatos.csv

# Buscar un valor específico en todos los metadatos
# Por ejemplo, encontrar todos los archivos que mencionan un usuario concreto
exiftool -r /ruta/ | grep -i "jsmith"

# Encontrar todos los archivos que contienen coordenadas GPS
exiftool -r /ruta/ -if '$GPSLatitude' -GPSLatitude -GPSLongitude -FileName

# Listar solo los archivos que tienen el campo Author relleno
exiftool -r /ruta/ -if '$Author' -Author -FileName -FileType
```

### Análisis de rutas de sistema de archivos

Esta es una de las técnicas más valiosas y menos conocidas. Cuando un documento de Office tiene imágenes incrustadas o vínculos a otros archivos, los metadatos pueden contener la ruta absoluta de esos recursos en el sistema de archivos original:

```bash
# Extraer referencias a rutas de sistema de archivos en documentos Word
exiftool -HyperlinkBase -PartnerGUID documento.docx

# En algunos casos las rutas aparecen en el campo Subject o en campos XMP personalizados
exiftool -XMP:* documento.pdf | grep -i "path\|ruta\|server\|\\\\\\|C:\\"

# Descomprimir el DOCX manualmente para ver las rutas en los XML internos
# Un DOCX es en realidad un ZIP — podemos inspeccionarlo directamente
unzip -p documento.docx word/document.xml | grep -o 'Target="[^"]*"'
unzip -p documento.docx word/_rels/document.xml.rels | python3 -m json.tool

# Lo anterior puede revelar líneas como:
# Target="file:///C:/Users/jsmith/Desktop/logo_empresa.png"
# Target="\\fileserver-01\departamentos\finanzas\plantilla.dotx"
```

Cuando encuentras una ruta como `\\fileserver-01\departamentos\finanzas\`, has descubierto:
- El nombre del servidor de ficheros: `fileserver-01`
- La estructura de carpetas compartidas por departamento
- El nombre del departamento y posiblemente su estructura jerárquica

### Exiftool para eliminar metadatos (defensivo)

Esta sección es útil tanto para ti como pentester (cuando creas documentos para el cliente que no deben revelar tu infraestructura) como para el conocimiento defensivo del informe:

```bash
# Eliminar TODOS los metadatos de un archivo
# Crea una copia limpia con el sufijo _original en el original
exiftool -all= documento.pdf

# Eliminar metadatos manteniendo el original sin cambios
exiftool -all= -o documento_limpio.pdf documento.pdf

# Eliminar metadatos de múltiples archivos en batch
exiftool -all= *.pdf *.docx *.xlsx

# Eliminar solo campos específicos (por ejemplo, el Author pero mantener el título)
exiftool -Author= -LastModifiedBy= documento.docx

# Eliminar solo metadatos GPS de imágenes (mantener otros metadatos)
exiftool -GPS*= foto.jpg

# Verificar que los metadatos se eliminaron correctamente
exiftool -Author -Creator -GPSLatitude documento_limpio.pdf
```

---

## 3. Recolección de documentos objetivo

Antes de poder analizar metadatos necesitas los documentos. La recolección es parte del proceso de reconocimiento y puede hacerse de varias formas.

### Google Dorks para documentos

Los Google Dorks son la forma más efectiva de encontrar documentos indexados de un dominio objetivo. Google indexa el contenido y los metadatos de los documentos que encuentra, lo que te permite hacer búsquedas muy específicas:

```bash
# PDFs del dominio objetivo — los más ricos en metadatos corporativos
site:example.com filetype:pdf

# Documentos de Office — DOCX y DOC
site:example.com filetype:docx
site:example.com filetype:doc

# Hojas de cálculo — pueden contener estructuras organizativas, listas de empleados
site:example.com filetype:xlsx
site:example.com filetype:xls

# Presentaciones — frecuentemente contienen información estratégica
site:example.com filetype:pptx
site:example.com filetype:ppt

# Buscar documentos que parecen internos pero están indexados por error
site:example.com filetype:pdf "internal use only"
site:example.com filetype:pdf "confidential"
site:example.com filetype:pdf "draft"
site:example.com filetype:docx "not for distribution"

# Documentos con contenido específico (útil cuando conoces terminología interna)
site:example.com filetype:pdf "Q3 2024" OR "tercer trimestre"
site:example.com filetype:pdf "presupuesto" OR "budget"
```

### Descarga automatizada de documentos

Una vez tienes las URLs de los documentos, puedes descargarlos en batch:

```bash
# Wget para descargar un documento específico
wget "https://example.com/docs/informe_anual.pdf" -O informe.pdf

# Descargar todos los PDFs de una lista de URLs
# Primero crear el archivo con las URLs (una por línea)
wget -i lista_urls_pdfs.txt -P ./documentos/

# wget con parámetros para ser respetuoso con el servidor
# --wait evita sobrecarga, --random-wait añade variación
wget -i urls.txt -P ./documentos/ --wait=2 --random-wait

# Con curl para una URL específica
curl -o informe.pdf "https://example.com/docs/informe.pdf"

# httrack para descargar todos los documentos de un sitio
# (usa con cuidado y solo en engagements autorizados)
httrack "https://example.com" -O ./sitio_completo/ "+*.pdf" "+*.docx" "+*.xlsx"
```

### Script Python para recolección y análisis automatizado

```python
#!/usr/bin/env python3
"""
doc_collector.py — Descarga y analiza metadatos de documentos de un dominio
Requiere: requests, googlesearch-python, exiftool instalado
pip3 install requests googlesearch-python
"""

import os
import subprocess
import json
import requests
from pathlib import Path
from urllib.parse import urlparse

def download_document(url, output_dir):
    """
    Descarga un documento de una URL y lo guarda en el directorio de salida.
    Devuelve la ruta local del archivo descargado o None si falla.
    """
    try:
        # Extraer el nombre del archivo de la URL
        parsed = urlparse(url)
        filename = os.path.basename(parsed.path)
        if not filename:
            filename = "documento_" + str(hash(url)) + ".pdf"

        output_path = os.path.join(output_dir, filename)

        # Descargar con un User-Agent de navegador real para evitar bloqueos
        headers = {
            "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
                         "AppleWebKit/537.36 (KHTML, like Gecko) "
                         "Chrome/120.0.0.0 Safari/537.36"
        }

        response = requests.get(url, headers=headers, timeout=30, stream=True)

        if response.status_code == 200:
            with open(output_path, "wb") as f:
                for chunk in response.iter_content(chunk_size=8192):
                    f.write(chunk)
            return output_path
        else:
            print(f"  [-] HTTP {response.status_code}: {url}")
            return None

    except Exception as e:
        print(f"  [-] Error descargando {url}: {e}")
        return None

def extract_metadata_exiftool(filepath):
    """
    Ejecuta exiftool sobre un archivo y devuelve los metadatos como diccionario.
    Usa el flag -json para obtener un formato estructurado y fácil de parsear.
    """
    try:
        result = subprocess.run(
            ["exiftool", "-json", "-Author", "-Creator", "-LastModifiedBy",
             "-Company", "-Producer", "-CreatorTool", "-Template",
             "-GPSLatitude", "-GPSLongitude", "-DateTimeOriginal",
             "-Make", "-Model", "-Software", "-XMPToolkit", filepath],
            capture_output=True, text=True, timeout=30
        )

        if result.stdout:
            data = json.loads(result.stdout)
            return data[0] if data else {}
        return {}

    except Exception as e:
        print(f"  [-] Error extrayendo metadatos de {filepath}: {e}")
        return {}

def analyze_collection(download_dir):
    """
    Analiza todos los documentos descargados y agrupa los hallazgos por categoría.
    Devuelve un resumen de toda la información recopilada.
    """
    # Diccionarios para agrupar hallazgos únicos
    findings = {
        "authors": {},           # autor → lista de archivos
        "companies": set(),      # empresas encontradas
        "software": set(),       # software identificado
        "usernames": set(),      # usernames extraídos
        "gps_locations": [],     # coordenadas GPS
        "internal_paths": set(), # rutas de red internas
        "all_metadata": []       # todos los metadatos crudos
    }

    # Campos que más nos interesan para reconocimiento
    interesting_fields = {
        "Author", "Creator", "LastModifiedBy", "Company",
        "Producer", "CreatorTool", "Template", "Software",
        "GPSLatitude", "GPSLongitude", "DateTimeOriginal",
        "Make", "Model"
    }

    # Procesar cada archivo del directorio
    for filepath in Path(download_dir).rglob("*"):
        if filepath.is_file() and filepath.suffix.lower() in \
                [".pdf", ".doc", ".docx", ".xls", ".xlsx", ".ppt", ".pptx", ".jpg", ".jpeg", ".png"]:

            print(f"  [*] Analizando: {filepath.name}")
            metadata = extract_metadata_exiftool(str(filepath))

            if not metadata:
                continue

            # Registrar metadatos completos
            metadata["_filename"] = filepath.name
            findings["all_metadata"].append(metadata)

            # Clasificar hallazgos por categoría
            author = metadata.get("Author")
            if author:
                if author not in findings["authors"]:
                    findings["authors"][author] = []
                findings["authors"][author].append(filepath.name)
                # Extraer posible username del nombre del autor
                if " " not in author:  # Si no tiene espacios, probablemente es un username
                    findings["usernames"].add(author)

            company = metadata.get("Company")
            if company:
                findings["companies"].add(company)

            # Registrar software identificado (útil para inferir vulnerabilidades)
            for sw_field in ["Creator", "CreatorTool", "Producer", "Software"]:
                sw = metadata.get(sw_field)
                if sw:
                    findings["software"].add(sw)

            # Registrar coordenadas GPS si están presentes
            lat = metadata.get("GPSLatitude")
            lon = metadata.get("GPSLongitude")
            if lat and lon:
                findings["gps_locations"].append({
                    "file": filepath.name,
                    "lat": lat,
                    "lon": lon,
                    "maps_url": f"https://www.google.com/maps?q={lat},{lon}"
                })

    return findings

def print_summary(findings, domain):
    """Imprime un resumen de los hallazgos más relevantes para el reporte."""

    print(f"\n{'='*60}")
    print(f"  RESUMEN DE METADATOS — {domain}")
    print(f"{'='*60}")

    print(f"\n[+] AUTORES IDENTIFICADOS ({len(findings['authors'])}):")
    for author, files in sorted(findings["authors"].items()):
        print(f"    {author}")
        for f in files[:3]:  # Máximo 3 archivos por autor
            print(f"      → {f}")

    print(f"\n[+] EMPRESAS ({len(findings['companies'])}):")
    for company in sorted(findings["companies"]):
        print(f"    {company}")

    print(f"\n[+] SOFTWARE IDENTIFICADO ({len(findings['software'])}):")
    for sw in sorted(findings["software"]):
        print(f"    {sw}")

    print(f"\n[+] USERNAMES POSIBLES ({len(findings['usernames'])}):")
    for username in sorted(findings["usernames"]):
        print(f"    {username}")

    if findings["gps_locations"]:
        print(f"\n[+] COORDENADAS GPS ({len(findings['gps_locations'])}):")
        for loc in findings["gps_locations"]:
            print(f"    {loc['file']}: {loc['lat']}, {loc['lon']}")
            print(f"    Maps: {loc['maps_url']}")

# Ejemplo de uso
if __name__ == "__main__":
    domain = "example.com"
    download_dir = f"./docs_{domain}"
    os.makedirs(download_dir, exist_ok=True)

    # Lista de URLs a descargar (obtenidas previamente con Google Dorks)
    urls = [
        "https://example.com/docs/informe_anual_2024.pdf",
        "https://example.com/media/presentacion_inversores.pptx",
        "https://example.com/files/politica_privacidad.pdf",
    ]

    print(f"[*] Descargando {len(urls)} documentos...")
    for url in urls:
        path = download_document(url, download_dir)
        if path:
            print(f"  [+] Descargado: {os.path.basename(path)}")

    print(f"\n[*] Analizando metadatos...")
    findings = analyze_collection(download_dir)

    print_summary(findings, domain)

    # Guardar resultados completos en JSON
    output_file = f"metadatos_{domain}.json"
    with open(output_file, "w", encoding="utf-8") as f:
        json.dump(findings, f, indent=2, ensure_ascii=False, default=str)

    print(f"\n[+] Resultados completos guardados en: {output_file}")
```

---

## 4. FOCA — Fingerprinting Organizations with Collected Archives

FOCA es una herramienta desarrollada por la empresa española ElevenPaths (ahora Telefónica Tech) que automatiza exactamente el proceso que hemos estado describiendo: descarga documentos de un dominio objetivo, extrae sus metadatos, los analiza, y construye un mapa de la infraestructura interna inferida.

Lo que diferencia a FOCA de simplemente usar Exiftool en batch es su capacidad de **correlacionar los hallazgos entre múltiples documentos** y construir una imagen coherente: si cinco documentos diferentes tienen el campo `LastModifiedBy` con el valor `jsmith`, FOCA lo agrupa. Si tres de esos documentos tienen rutas que apuntan a `\\fileserver-01\`, infiere que `fileserver-01` es un servidor real de la infraestructura. Si las versiones de software varían entre documentos, puede indicar que hay equipos con diferentes parches aplicados.

### Instalación y entorno

FOCA es una aplicación de Windows desarrollada en .NET. La instalación es sencilla:

```bash
# FOCA se descarga desde el repositorio oficial de ElevenPaths / Telefónica
# https://github.com/ElevenPaths/FOCA

# Requisitos del sistema:
# - Windows 7/8/10/11
# - .NET Framework 4.7.2 o superior
# - Conexión a internet para las búsquedas en motores

# En Kali Linux con Wine (alternativa, con limitaciones):
sudo apt install wine wine64
wine FOCA.exe
# No todos los módulos funcionan perfectamente bajo Wine

# Alternativa nativa en Linux:
# Usar el script de Python que veremos más adelante para replicar las funciones principales
```

### Flujo de trabajo en FOCA

Aunque FOCA es una herramienta gráfica, su flujo de trabajo tiene pasos muy definidos que conviene entender:

**1. Crear proyecto** — FOCA organiza el trabajo en proyectos. Creas uno por dominio objetivo con el nombre del dominio y el nombre de la empresa. Esto permite gestionar múltiples engagements y guardar los resultados.

**2. Búsqueda de documentos** — FOCA busca automáticamente en Google, Bing y DuckDuckGo usando dorks predefinidos para el dominio dado. La búsqueda es configurable: puedes elegir qué motores usar y qué tipos de archivo buscar.

```
Tipos de archivo que busca FOCA por defecto:
.doc  .docx  .xls  .xlsx  .ppt  .pptx
.pdf  .odt   .ods  .odp
.svg  .indd
.wpd  .odf
```

**3. Descarga de documentos** — Los documentos encontrados se descargan automáticamente. FOCA muestra el progreso y marca cuáles se descargaron con éxito.

**4. Extracción de metadatos** — Una vez descargados, FOCA extrae los metadatos de cada documento. Este proceso se puede ejecutar en batch sobre todos los documentos a la vez.

**5. Análisis y correlación** — Este es el paso diferencial de FOCA. Agrupa todos los hallazgos:

```
Pestaña "Users":
  jsmith (encontrado en 12 documentos)
  mgarcia (encontrado en 3 documentos)
  admin (encontrado en 1 documento)

Pestaña "Folders":
  C:\Users\jsmith\Documents\Contratos\
  \\fileserver-01\departamentos\finanzas\
  \\printserver-madrid\HP_Color_5550\

Pestaña "Printers":
  \\printserver-madrid\HP_Color_LaserJet_5550
  \\printserver-bcn\Ricoh_MP_C3003

Pestaña "Software":
  Microsoft Office Word 2016 (5 docs)
  Microsoft Office Word 2019 (3 docs)
  Adobe Acrobat 11.0 (2 docs)

Pestaña "Emails":
  jsmith@example.com
  mgarcia@example.com
```

**6. Inferencia de red** — FOCA intenta resolver los nombres de servidor encontrados en las rutas para verificar si son accesibles desde internet, y los añade a un mapa de red inferido.

### Replicar las funciones de FOCA en Linux con Python

Para quienes trabajan en Linux y no pueden usar FOCA directamente, este script replica las funciones principales:

```python
#!/usr/bin/env python3
"""
foca_lite.py — Replicar las funciones principales de FOCA en Python/Linux
Extrae metadatos de documentos y construye un mapa de infraestructura interna
Requiere: exiftool instalado, pip3 install requests
"""

import os
import re
import json
import subprocess
import socket
from pathlib import Path
from collections import defaultdict

class FOCALite:
    """
    Versión simplificada de FOCA para sistemas Linux.
    Analiza una colección de documentos y extrae infraestructura interna.
    """

    def __init__(self, documents_dir):
        self.docs_dir = documents_dir
        # Estructuras para acumular hallazgos
        self.users = defaultdict(list)       # usuario → [archivos]
        self.servers = defaultdict(list)     # servidor → [archivos]
        self.paths = defaultdict(list)       # ruta → [archivos]
        self.printers = defaultdict(list)    # impresora → [archivos]
        self.emails = set()                  # emails encontrados
        self.software = defaultdict(list)    # software → [archivos]
        self.gps = []                        # ubicaciones GPS

    def extract_all_metadata(self, filepath):
        """
        Extrae TODOS los metadatos de un archivo con exiftool.
        Usamos -j para JSON y sin filtro de campos para no perdernos nada.
        """
        try:
            result = subprocess.run(
                ["exiftool", "-j", "-a", "-u", str(filepath)],
                capture_output=True, text=True, timeout=60
            )
            if result.stdout:
                data = json.loads(result.stdout)
                return data[0] if data else {}
        except Exception:
            pass
        return {}

    def find_unc_paths(self, text):
        """
        Busca rutas UNC (\\servidor\compartido) en cualquier cadena de texto.
        Las rutas UNC revelan servidores de ficheros e impresoras internas.
        """
        if not text:
            return []
        # Patrón para rutas UNC: \\servidor\recurso o //servidor/recurso
        pattern = r'\\\\([a-zA-Z0-9_\-\.]+)\\([a-zA-Z0-9_\-\. \\]+)'
        return re.findall(pattern, str(text))

    def find_local_paths(self, text):
        """
        Busca rutas locales de Windows en el texto (C:\Users\... D:\...)
        Revelan nombres de usuario y estructura del sistema de archivos.
        """
        if not text:
            return []
        pattern = r'[A-Za-z]:\\(?:[^\\/:*?"<>|\r\n]+\\)*[^\\/:*?"<>|\r\n]*'
        return re.findall(pattern, str(text))

    def find_emails_in_text(self, text):
        """Extrae emails de cualquier campo de texto."""
        if not text:
            return []
        pattern = r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b'
        return re.findall(pattern, str(text))

    def extract_username_from_path(self, path):
        """
        Extrae el nombre de usuario de una ruta Windows típica.
        C:\\Users\\jsmith\\Documents → jsmith
        """
        # Patrón: C:\Users\<username>\ o C:\Documents and Settings\<username>\
        match = re.search(r'[Uu]sers?\\([^\\]+)\\', path)
        if match:
            return match.group(1)
        match = re.search(r'Documents and Settings\\([^\\]+)\\', path)
        if match:
            return match.group(1)
        return None

    def analyze_document(self, filepath):
        """
        Análisis completo de un documento:
        1. Extrae todos los metadatos con exiftool
        2. Busca patrones de infraestructura en cada campo
        3. Acumula los hallazgos en las estructuras de datos
        """
        filename = os.path.basename(filepath)
        metadata = self.extract_all_metadata(filepath)

        if not metadata:
            return

        # Analizar cada campo de metadatos
        for field, value in metadata.items():
            value_str = str(value) if value else ""

            # Buscar rutas UNC (servidores e impresoras)
            unc_paths = self.find_unc_paths(value_str)
            for server, share in unc_paths:
                full_path = f"\\\\{server}\\{share}"
                self.servers[server].append(filename)
                self.paths[full_path].append(filename)
                # Detectar impresoras por nombres típicos
                printer_keywords = ["print", "hp", "ricoh", "epson", "canon",
                                   "laser", "copier", "xerox", "brother"]
                if any(kw in server.lower() or kw in share.lower()
                       for kw in printer_keywords):
                    self.printers[full_path].append(filename)

            # Buscar rutas locales (para extraer usernames)
            local_paths = self.find_local_paths(value_str)
            for path in local_paths:
                self.paths[path].append(filename)
                username = self.extract_username_from_path(path)
                if username and username.lower() not in ["windows", "program files",
                                                          "users", "public", "default"]:
                    self.users[username].append(filename)

            # Buscar emails
            emails = self.find_emails_in_text(value_str)
            for email in emails:
                self.emails.add(email)

        # Campos específicos de metadatos
        author = metadata.get("Author") or metadata.get("Creator")
        if author:
            self.users[author].append(filename)

        last_mod = metadata.get("LastModifiedBy")
        if last_mod:
            self.users[last_mod].append(filename)

        # Software
        for sw_field in ["CreatorTool", "Producer", "Software", "Application"]:
            sw = metadata.get(sw_field)
            if sw:
                self.software[str(sw)].append(filename)

        # GPS
        lat = metadata.get("GPSLatitude")
        lon = metadata.get("GPSLongitude")
        if lat and lon:
            self.gps.append({
                "file": filename,
                "latitude": lat,
                "longitude": lon,
                "google_maps": f"https://maps.google.com/?q={lat},{lon}"
            })

    def try_resolve_servers(self):
        """
        Intenta resolver los nombres de servidor encontrados.
        Si resuelven a una IP, pueden ser accesibles desde internet.
        """
        resolved = {}
        for server in self.servers:
            try:
                ip = socket.gethostbyname(server)
                resolved[server] = ip
                print(f"  [!] Servidor resuelto desde internet: {server} → {ip}")
            except socket.gaierror:
                resolved[server] = None  # No resuelve — probablemente interno

        return resolved

    def run(self):
        """
        Ejecutar el análisis completo sobre todos los documentos del directorio.
        """
        extensions = {".pdf", ".doc", ".docx", ".xls", ".xlsx",
                     ".ppt", ".pptx", ".odt", ".ods", ".odp",
                     ".jpg", ".jpeg", ".png", ".tiff", ".gif"}

        files = [f for f in Path(self.docs_dir).rglob("*")
                 if f.is_file() and f.suffix.lower() in extensions]

        print(f"[*] Analizando {len(files)} documentos...")
        for i, filepath in enumerate(files, 1):
            print(f"  [{i}/{len(files)}] {filepath.name}")
            self.analyze_document(str(filepath))

        print(f"\n[*] Intentando resolver servidores en DNS...")
        resolved = self.try_resolve_servers()

        return self.generate_report(resolved)

    def generate_report(self, resolved_servers):
        """Genera un reporte estructurado con todos los hallazgos."""

        report = {
            "summary": {
                "unique_users": len(self.users),
                "unique_servers": len(self.servers),
                "unique_paths": len(self.paths),
                "unique_emails": len(self.emails),
                "gps_locations": len(self.gps)
            },
            "users": dict(self.users),
            "servers": {
                server: {
                    "files": files,
                    "resolved_ip": resolved_servers.get(server),
                    "internet_accessible": resolved_servers.get(server) is not None
                }
                for server, files in self.servers.items()
            },
            "printers": dict(self.printers),
            "emails": list(self.emails),
            "software": dict(self.software),
            "gps_locations": self.gps,
            "paths": dict(self.paths)
        }

        # Imprimir resumen en consola
        print(f"\n{'='*60}")
        print("  RESUMEN DE HALLAZGOS (estilo FOCA)")
        print(f"{'='*60}")

        print(f"\n[+] USUARIOS ({len(self.users)}):")
        for user, files in sorted(self.users.items(),
                                   key=lambda x: len(x[1]), reverse=True):
            print(f"    {user:<30} ({len(files)} documentos)")

        print(f"\n[+] SERVIDORES INTERNOS ({len(self.servers)}):")
        for server, files in sorted(self.servers.items(),
                                     key=lambda x: len(x[1]), reverse=True):
            ip = resolved_servers.get(server)
            status = f"→ {ip} [ACCESIBLE DESDE INTERNET]" if ip else "(solo interno)"
            print(f"    {server:<35} {status}")

        if self.printers:
            print(f"\n[+] IMPRESORAS ({len(self.printers)}):")
            for printer, files in self.printers.items():
                print(f"    {printer}")

        if self.emails:
            print(f"\n[+] EMAILS ({len(self.emails)}):")
            for email in sorted(self.emails):
                print(f"    {email}")

        print(f"\n[+] SOFTWARE IDENTIFICADO ({len(self.software)}):")
        for sw, files in sorted(self.software.items(),
                                  key=lambda x: len(x[1]), reverse=True):
            print(f"    {sw:<50} ({len(files)} docs)")

        if self.gps:
            print(f"\n[+] UBICACIONES GPS ({len(self.gps)}):")
            for loc in self.gps:
                print(f"    {loc['file']}: {loc['google_maps']}")

        return report


# Ejecución del script
if __name__ == "__main__":
    import sys

    if len(sys.argv) < 2:
        print("Uso: python3 foca_lite.py <directorio_con_documentos>")
        sys.exit(1)

    docs_dir = sys.argv[1]
    foca = FOCALite(docs_dir)
    report = foca.run()

    # Guardar reporte JSON
    output = "foca_report.json"
    with open(output, "w", encoding="utf-8") as f:
        json.dump(report, f, indent=2, ensure_ascii=False, default=str)

    print(f"\n[+] Reporte completo guardado en: {output}")
```

---

## 5. Casos prácticos y hallazgos típicos

Para dar contexto a todo lo anterior, estos son los tipos de hallazgos más frecuentes en reconocimiento real con metadatos, y qué significa cada uno para el engagement:

### Hallazgo 1 — Username de Active Directory

```
Archivo: propuesta_comercial_2024.pdf
Author: jsmith
LastModifiedBy: John Smith
```

**Lo que significa**: `jsmith` es probablemente el formato de username en Active Directory de la empresa. Combinado con el dominio NetBIOS (que puede aparecer en otras rutas como `CORP\jsmith`), tienes las credenciales de formato para intentar password spraying contra OWA, Citrix o VPN.

### Hallazgo 2 — Servidor de ficheros interno

```
Archivo: contrato_servicio.docx
HyperlinkBase: \\fileserver-madrid-01\departamentos\comercial\
```

**Lo que significa**: `fileserver-madrid-01` es un servidor real de la red interna. Si desde internet resuelve una IP (quizás por un error de configuración DNS), puede ser un objetivo directo. Si no, saber su nombre es útil para enumeración lateral una vez dentro de la red.

### Hallazgo 3 — Versión de software vulnerable

```
Archivos: 8 documentos PDF
Creator: Adobe Acrobat 9.0
Producer: Adobe PDF Library 9.0
```

**Lo que significa**: Adobe Acrobat 9.0 es de 2008 y tiene docenas de CVEs críticos. El hecho de que ocho documentos recientes hayan sido creados con esta versión sugiere que hay equipos en la organización con un software gravemente desactualizado.

### Hallazgo 4 — Coordenadas GPS de instalaciones

```
Archivo: foto_inauguracion_oficina.jpg
GPSLatitude: 40.4296667
GPSLongitude: -3.6926333
```

**Lo que significa**: esta es la ubicación exacta de la oficina. Útil en reconocimiento físico (tailgating), para correlacionar con imágenes de satélite y planos de planta, o simplemente para confirmar qué instalación corresponde al objetivo.

---

## 6. Cheatsheet de referencia rápida

```bash
# ── EXIFTOOL — EXTRACCIÓN ─────────────────────────────────────────────────
exiftool archivo.pdf                                   # Todos los metadatos
exiftool -Author -Creator -Company archivo.pdf         # Solo campos clave
exiftool -csv directorio/ > metadatos.csv              # Batch a CSV
exiftool -r -if '$GPSLatitude' directorio/             # Solo con GPS
exiftool -r directorio/ | grep -i "path\|server\|\\\\" # Buscar rutas

# Campos más útiles en reconocimiento:
exiftool -Author -LastModifiedBy -Creator -Company \
         -Producer -CreatorTool -Template -Software \
         -GPSLatitude -GPSLongitude archivo

# ── EXIFTOOL — DEFENSA (eliminar metadatos) ───────────────────────────────
exiftool -all= archivo.pdf                             # Eliminar todo
exiftool -all= -o limpio.pdf original.pdf              # Mantener original
exiftool -GPS*= foto.jpg                               # Solo eliminar GPS
exiftool -all= *.pdf *.docx *.xlsx                     # Batch

# ── INSPECCIONAR DOCX MANUALMENTE ────────────────────────────────────────
unzip -p doc.docx docProps/core.xml | python3 -m json.tool
unzip -p doc.docx docProps/app.xml
unzip -p doc.docx word/_rels/document.xml.rels | grep Target

# ── GPS — CONVERTIR A GOOGLE MAPS ────────────────────────────────────────
exiftool -n -GPSLatitude -GPSLongitude foto.jpg
# → Usar valores en: https://maps.google.com/?q=LAT,LON

# ── GOOGLE DORKS PARA ENCONTRAR DOCUMENTOS ────────────────────────────────
site:ejemplo.com filetype:pdf
site:ejemplo.com filetype:docx OR filetype:doc
site:ejemplo.com filetype:xlsx OR filetype:xls
site:ejemplo.com filetype:pdf "internal" OR "confidential"
site:ejemplo.com filetype:pdf "draft" OR "not for distribution"

# ── DESCARGA BATCH ────────────────────────────────────────────────────────
wget -i urls.txt -P ./docs/ --wait=2 --random-wait
curl -o doc.pdf "https://ejemplo.com/informe.pdf"

# ── FOCA LITE — PYTHON ────────────────────────────────────────────────────
python3 foca_lite.py ./documentos_descargados/
# → Genera foca_report.json con usuarios, servidores, rutas, emails, GPS

# ── RECURSOS Y REFERENCIAS ───────────────────────────────────────────────
https://exiftool.org                    → Documentación oficial Exiftool
https://exiftool.org/TagNames/          → Todos los tags soportados
https://github.com/ElevenPaths/FOCA    → FOCA original (Windows)
https://www.sno.phy.queensu.ca/~phil/exiftool/forum/ → Foro Exiftool
https://github.com/danielmiessler/SecLists → Wordlists y recursos OSINT
```
