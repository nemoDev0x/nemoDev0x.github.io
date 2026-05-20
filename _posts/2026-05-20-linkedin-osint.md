---
layout: post
title: "LinkedIn OSINT — Enumeración de Empleados y Estructura Corporativa"
date: 2026-05-20
categories: [reconocimiento]
tags: [linkedin, osint, reconocimiento, recon, empleados, estructura-corporativa, spear-phishing, red-team]
description: "Guía profesional de LinkedIn OSINT: enumeración de empleados, inferencia de organigramas, identificación de tecnologías, generación de listas de usuarios y construcción de perfiles para Red Team y pentesting autorizado."
---

## LinkedIn como fuente de inteligencia corporativa

En reconocimiento profesional, LinkedIn es la fuente de inteligencia corporativa más rica que existe en fuentes abiertas. Cada perfil que un empleado completa voluntariamente contiene información que en un engagement de Red Team tiene un valor enorme: nombres reales, cargos exactos, departamentos, tecnologías con las que trabaja, proyectos en los que participa, formación, certificaciones, y la red de conexiones que revela la estructura organizativa de la empresa.

Lo que hace a LinkedIn especialmente valioso no es solo la información individual de cada empleado, sino la **capacidad de inferir estructuras y relaciones** a partir del conjunto. Si ves que diez personas en la misma empresa tienen en su perfil "Azure AD", "Palo Alto Networks" y "CrowdStrike", sabes exactamente qué stack de seguridad usa esa organización sin haber enviado un solo paquete. Si ves que el Director de IT lleva solo tres meses en el cargo, ese es un objetivo ideal para un ataque de ingeniería social porque aún está aprendiendo los procesos internos.

Todo el contenido de este post está orientado a **reconocimiento en contextos autorizados**: auditorías de seguridad, Red Team con contrato, y Bug Bounty. La recolección de información personal sin autorización puede ser ilegal según la jurisdicción.

---

## 1. Qué información revela LinkedIn y por qué importa

Antes de entrar en herramientas y técnicas, conviene entender exactamente qué tipos de información se pueden extraer de LinkedIn y qué valor tiene cada uno en un engagement.

### Información directa de perfiles

Cada perfil de LinkedIn puede contener:

```
Información de identidad:
├── Nombre completo real
├── Foto de perfil (para reconocimiento facial en ingeniería social)
├── Ubicación (ciudad, país)
└── Pronombres y otras preferencias personales

Información profesional:
├── Cargo actual y anteriores
├── Empresa actual y antiguas
├── Departamento (a veces explícito en el cargo)
├── Años de experiencia en cada posición
├── Descripción del puesto (puede revelar responsabilidades específicas)
└── Fecha de incorporación a la empresa actual

Habilidades y tecnologías:
├── Lista de skills validados por otros usuarios
├── Certificaciones (AWS, Azure, CISSP, CEH, OSCP...)
├── Tecnologías mencionadas en la descripción del perfil
└── Cursos y formación completados

Red de contactos:
├── Conexiones comunes (revelan estructura organizativa)
├── Recomendaciones (quién trabaja con quién)
├── Grupos a los que pertenece
└── Publicaciones y comentarios (revelan opiniones, proyectos, eventos)
```

### Por qué cada tipo de dato importa en seguridad

**Nombres y cargos** — permiten construir la lista de objetivos para spear phishing, identificar quién tiene acceso privilegiado (administradores de sistema, responsables de seguridad, directores financieros), e inferir el patrón de emails corporativos combinando nombre + dominio.

**Tecnologías en los perfiles** — revelan el stack técnico sin necesidad de escaneo activo. Un administrador de sistemas que lista "Windows Server 2019, VMware vSphere, NetApp" está describiendo la infraestructura de su empresa. Un ingeniero de seguridad que menciona "Splunk SIEM, CrowdStrike Falcon, Palo Alto NGFW" está describiendo las defensas.

**Fecha de incorporación** — los empleados recientes son objetivos preferidos para ingeniería social porque aún no conocen todos los procedimientos, son más propensos a seguir instrucciones sin cuestionarlas, y su presencia en la empresa puede no estar completamente verificada por sus compañeros.

**Certificaciones** — un administrador de AWS certificado probablemente gestiona infraestructura cloud. Un CISSP probablemente está en el equipo de seguridad. Un PMP probablemente gestiona proyectos con acceso a información estratégica.

**Historial laboral** — las empresas anteriores pueden tener acuerdos con la empresa objetivo (proveedores, clientes, socios), y conocer esas relaciones ayuda a construir pretextos más creíbles.

---

## 2. Enumeración manual sin herramientas

La forma más básica de recolección en LinkedIn no requiere ninguna herramienta especial — solo un navegador y metodología.

### Página de empleados de la empresa

Cada empresa en LinkedIn tiene una página pública que incluye la sección "Personas" mostrando todos los empleados que han declarado trabajar ahí:

```
# URL directa a la lista de empleados
https://www.linkedin.com/company/nombre-empresa/people/

# Filtros disponibles en la interfaz:
# - Por ubicación (Madrid, Barcelona, Remote...)
# - Por departamento (Engineering, Sales, IT, Security...)
# - Por cargo (Manager, Director, Engineer...)
# - Por universidad
# - Por habilidades

# Ejemplos de URLs con filtros (pueden variar)
https://www.linkedin.com/company/nombre-empresa/people/?facetCurrentFunction=13
# facetCurrentFunction=13 → Information Technology
# facetCurrentFunction=8  → Engineering
# facetCurrentFunction=25 → Finance
```

Sin estar logado en LinkedIn, la página de empleados muestra resultados limitados. Con una cuenta, muestra todos los empleados paginados. Con una cuenta de LinkedIn Premium o Sales Navigator, los filtros son más potentes y puedes exportar listas.

### Google Dorks para perfiles de LinkedIn

Google indexa los perfiles públicos de LinkedIn, lo que permite enumerarlos sin tener cuenta o sin estar limitado por las restricciones de LinkedIn:

```bash
# Buscar empleados de una empresa que mencionan tecnologías específicas
site:linkedin.com/in "Example Corp" "engineer"
site:linkedin.com/in "Example Corp" "security"
site:linkedin.com/in "Example Corp" "system administrator"
site:linkedin.com/in "Example Corp" "devops"
site:linkedin.com/in "Example Corp" "network"

# Buscar directivos y personas con acceso privilegiado
site:linkedin.com/in "Example Corp" "CISO" OR "Chief Information Security"
site:linkedin.com/in "Example Corp" "IT Director" OR "IT Manager"
site:linkedin.com/in "Example Corp" "CTO" OR "Chief Technology"
site:linkedin.com/in "Example Corp" "sysadmin" OR "system admin"
site:linkedin.com/in "Example Corp" "domain admin" OR "Active Directory"

# Buscar personas con tecnologías específicas que revelan la infraestructura
site:linkedin.com/in "Example Corp" "Azure AD" OR "Active Directory"
site:linkedin.com/in "Example Corp" "AWS" "EC2" OR "S3"
site:linkedin.com/in "Example Corp" "Palo Alto" OR "Fortinet" OR "Cisco ASA"
site:linkedin.com/in "Example Corp" "Splunk" OR "QRadar" OR "Sentinel"

# Buscar empleados recientes (mencionan "recently joined" o fecha reciente)
site:linkedin.com/in "Example Corp" "joined" "2024"

# Buscar perfiles que revelan proyectos internos
site:linkedin.com/in "Example Corp" "migration" OR "rollout" OR "implementation"
```

### Información de tecnologías desde las ofertas de trabajo

Las ofertas de empleo publicadas por la empresa son una fuente de inteligencia tecnológica sin parangón. Describen exactamente qué usa la empresa internamente:

```bash
# En LinkedIn Jobs
site:linkedin.com/jobs "Example Corp" "security engineer"
site:linkedin.com/jobs "Example Corp" "system administrator"
site:linkedin.com/jobs "Example Corp" "cloud" "AWS"

# En Indeed (indexado por Google)
site:indeed.com "Example Corp" developer
site:indeed.com "Example Corp" security

# En la web de la empresa directamente
site:example.com/careers
site:example.com/jobs
site:example.com "we're hiring"
```

**Qué extraer de una oferta de trabajo para el reporte de reconocimiento:**

```
Oferta: "Senior Security Engineer — Example Corp"

Requisitos:
"5+ años con Palo Alto NGFW"           → Firewall: Palo Alto
"Experiencia con CrowdStrike Falcon"   → EDR: CrowdStrike
"Conocimientos de Splunk ES"           → SIEM: Splunk Enterprise Security
"AWS Security Hub y GuardDuty"         → Cloud: AWS
"Active Directory y Azure AD"          → Directorio: AD híbrido
"Tenable Nessus o Qualys"              → Scanner: Tenable o Qualys

Deseado:
"CISSP o CISM valorado"                → Nivel de madurez del equipo de seguridad
"Experiencia con ISO 27001"            → Tienen certificación o la buscan
"Red Team / Penetration Testing"       → Equipo maduro, probable purple team

Conclusión: empresa con defensa madura, AD híbrido, AWS, EDR y SIEM bien implementados.
Atacar intentando evadir CrowdStrike. Phishing como vector más viable.
```

---

## 3. Herramientas automatizadas

### linkedin2username — Generación de listas de usuarios

linkedin2username es una herramienta Python que accede a LinkedIn con credenciales de una cuenta real, busca los empleados de una empresa, y genera automáticamente listas de posibles nombres de usuario en todos los formatos comunes.

La lista generada es el input ideal para ataques de **password spraying** contra servicios expuestos como OWA (Outlook Web Access), Citrix, VPN, o paneles de administración. El password spraying consiste en probar una sola contraseña común contra muchos usuarios, evitando el bloqueo de cuentas que ocurriría al hacer fuerza bruta de un solo usuario.

```bash
# Instalar desde GitHub
git clone https://github.com/initstring/linkedin2username
cd linkedin2username
pip3 install -r requirements.txt

# Uso básico
# -u: tu cuenta de LinkedIn (solo para autenticación)
# -c: nombre exacto de la empresa tal como aparece en LinkedIn
python3 linkedin2username.py -u tu_email@gmail.com -c "Example Corp"

# Especificar el número de páginas a recolectar
# Cada página tiene ~25 empleados
python3 linkedin2username.py -u tu_email@gmail.com -c "Example Corp" -p 10

# Filtrar por departamento o cargo (si LinkedIn lo permite)
python3 linkedin2username.py -u tu_email@gmail.com -c "Example Corp" -k "engineer"

# Output generado automáticamente:
# jsmith           → inicial + apellido
# john.smith       → nombre.apellido
# smith.john       → apellido.nombre
# johnsmith        → sin separador
# smithj           → apellido + inicial
# john_smith       → nombre_apellido
# j.smith          → inicial.apellido

# Los archivos de output se guardan en el directorio actual:
# jsmith.txt, john.smith.txt, etc. — uno por formato
```

Lo que hace linkedin2username especialmente útil es que genera todos los formatos a la vez y los guarda en archivos separados. Esto permite probar cada formato contra el servicio objetivo y descartar los que no existen sin mucho esfuerzo manual.

### CrossLinked — Alternativa sin cuenta

CrossLinked es similar a linkedin2username pero usa Google como intermediario en lugar de acceder directamente a LinkedIn. Esto tiene la ventaja de no requerir una cuenta de LinkedIn, pero da menos resultados:

```bash
# Instalar
pip3 install crosslinked

# Uso básico: el formato especifica el patrón de usuario deseado
# {f} = primera inicial del nombre
# {last} = apellido completo
# {first} = nombre completo
crosslinked -f '{f}{last}' 'Example Corp'
crosslinked -f '{first}.{last}' 'Example Corp'

# Guardar en archivo y con timeout más largo para empresas grandes
crosslinked -f '{f}{last}' 'Example Corp' -o usuarios.txt -t 30

# Múltiples formatos en un solo comando
crosslinked -f '{first}.{last}@example.com' 'Example Corp' -o emails.txt
```

### Scrapin — Extracción detallada de perfiles

Scrapin es una herramienta más reciente que permite extraer información detallada de perfiles de LinkedIn incluyendo historial laboral, habilidades y conexiones:

```bash
# Instalar
pip3 install scrapin

# Extraer información de un perfil específico
scrapin -u https://www.linkedin.com/in/john-smith-123456/

# Extraer empleados de una empresa
scrapin -c "Example Corp" -o empleados.json
```

---

## 4. Construir el organigrama inferido

Una de las técnicas más valiosas en LinkedIn OSINT no es recolectar perfiles individuales sino construir el organigrama de la empresa a partir de los datos de cargo y jerarquía que los propios empleados publican.

### Identificar niveles jerárquicos

Los cargos en LinkedIn siguen patrones muy predecibles que permiten inferir niveles de autoridad y acceso:

```
Nivel C-Suite (máximo acceso, mayor impacto si comprometido):
├── CEO, CTO, CISO, CFO, COO
├── Chief * Officer, Chief * Director
└── Presidente, Vicepresidente Ejecutivo

Directores (acceso amplio, decisiones estratégicas):
├── Director de IT, Director de Seguridad
├── IT Director, Security Director
└── Head of *, VP of *

Managers (acceso operacional, a menudo con privilegios elevados):
├── IT Manager, Security Manager
├── Team Lead, Tech Lead
└── Supervisor de *

Técnicos especializados (acceso a sistemas específicos):
├── Sysadmin, System Administrator
├── Network Engineer, Network Admin
├── DBA, Database Administrator
├── Cloud Engineer, DevOps Engineer
└── Security Analyst, SOC Analyst

Usuarios estándar (menor acceso individual pero útiles para acceso inicial):
├── Software Developer, Software Engineer
├── Business Analyst
└── Sales Representative, Account Manager
```

### Mapa de accesos inferido

A partir de los cargos podemos inferir qué sistemas puede acceder cada perfil, lo que guía la priorización de objetivos en spear phishing:

```
Administrador de Active Directory:
→ Control total del directorio: usuarios, grupos, GPOs, forest

DBA (Database Administrator):
→ Acceso a todas las bases de datos de producción

Cloud Engineer / AWS Administrator:
→ Acceso a infraestructura cloud: S3, EC2, RDS, IAM

Security Analyst / SOC Analyst:
→ Acceso a SIEM, logs, alertas — paradójicamente útil
  porque puede ver todo lo que detecta el sistema

Finance Director / CFO:
→ Sistemas financieros, ERP, nóminas — datos muy sensibles

HR Manager:
→ Datos de empleados, nóminas, contratos — muy sensible

IT Helpdesk:
→ Puede resetear contraseñas, acceso a ticketing, contacto con todos
  → Objetivo clásico de vishing (phishing por teléfono)
```

### Script para construir el organigrama

```python
#!/usr/bin/env python3
"""
org_mapper.py — Construye un organigrama inferido desde datos de LinkedIn
Entrada: archivo JSON o CSV con empleados y sus cargos
Salida: organigrama en texto y análisis de targets prioritarios
"""

import json
import sys
from collections import defaultdict

# Clasificación de cargos por nivel de acceso y valor para el engagement
ROLE_CLASSIFICATION = {
    "c_suite": {
        "keywords": ["ceo", "cto", "ciso", "cfo", "coo", "chief", "president", "vp ", "vice president"],
        "priority": 5,
        "access": "Acceso estratégico total — decisiones y datos sensibles"
    },
    "directors": {
        "keywords": ["director", "head of", "global head"],
        "priority": 4,
        "access": "Acceso departamental amplio — aprobaciones y datos del área"
    },
    "managers": {
        "keywords": ["manager", "team lead", "tech lead", "supervisor", "coordinator"],
        "priority": 3,
        "access": "Acceso operacional — gestión de equipos y sistemas del área"
    },
    "sysadmin": {
        "keywords": ["sysadmin", "system admin", "windows admin", "linux admin",
                     "active directory", "domain admin", "infrastructure"],
        "priority": 5,
        "access": "Acceso privilegiado a sistemas — administración de infraestructura"
    },
    "cloud": {
        "keywords": ["cloud", "aws", "azure", "gcp", "devops", "site reliability", "sre"],
        "priority": 4,
        "access": "Acceso a infraestructura cloud — posibles claves de API y roles IAM"
    },
    "security": {
        "keywords": ["security", "soc", "ciberseguridad", "infosec", "pentest",
                     "incident response", "threat"],
        "priority": 4,
        "access": "Acceso a herramientas de seguridad — SIEM, EDR, logs"
    },
    "database": {
        "keywords": ["dba", "database", "sql", "oracle", "data engineer"],
        "priority": 4,
        "access": "Acceso a bases de datos — datos de negocio y usuarios"
    },
    "network": {
        "keywords": ["network", "networking", "firewall", "routing", "cisco", "juniper"],
        "priority": 4,
        "access": "Acceso a infraestructura de red — configuraciones y segmentación"
    },
    "developer": {
        "keywords": ["developer", "engineer", "programmer", "software", "frontend",
                     "backend", "fullstack", "architect"],
        "priority": 2,
        "access": "Acceso a código fuente y posibles credenciales en repos"
    },
    "helpdesk": {
        "keywords": ["helpdesk", "help desk", "support", "it support", "soporte"],
        "priority": 3,
        "access": "Puede resetear contraseñas — objetivo ideal para vishing"
    },
    "finance": {
        "keywords": ["finance", "financial", "accounting", "payroll", "treasury", "cfo"],
        "priority": 3,
        "access": "Acceso a sistemas financieros — objetivo BEC (Business Email Compromise)"
    },
    "hr": {
        "keywords": ["human resources", "hr ", "people", "talent", "recruiting"],
        "priority": 3,
        "access": "Datos de empleados — nóminas, contratos, información personal"
    }
}

def classify_role(title):
    """
    Clasifica un cargo en una categoría y devuelve la prioridad y descripción de acceso.
    Comprueba si alguna keyword de cada categoría aparece en el título.
    """
    title_lower = title.lower()
    matches = []

    for role_type, config in ROLE_CLASSIFICATION.items():
        for keyword in config["keywords"]:
            if keyword in title_lower:
                matches.append((role_type, config["priority"], config["access"]))
                break  # Solo un match por categoría

    if matches:
        # Devolver la categoría con mayor prioridad
        return max(matches, key=lambda x: x[1])
    return ("other", 1, "Acceso estándar de usuario")

def analyze_employees(employees):
    """
    Analiza la lista de empleados y construye el mapa de la organización.
    employees: lista de dicts con al menos 'name' y 'title'
    """
    org_map = defaultdict(list)
    priority_targets = []
    tech_stack = defaultdict(int)

    # Tecnologías a buscar en los perfiles
    tech_keywords = [
        "aws", "azure", "gcp", "kubernetes", "docker",
        "active directory", "azure ad", "okta", "ldap",
        "palo alto", "fortinet", "cisco asa", "checkpoint",
        "splunk", "qradar", "sentinel", "elastic siem",
        "crowdstrike", "defender", "sentinelone", "carbon black",
        "vmware", "hyper-v", "esxi",
        "oracle", "sql server", "mysql", "postgresql",
        "jenkins", "gitlab", "github actions", "terraform", "ansible"
    ]

    for employee in employees:
        name = employee.get("name", "")
        title = employee.get("title", "")
        skills = employee.get("skills", "").lower()
        description = employee.get("description", "").lower()

        # Clasificar el cargo
        role_type, priority, access_desc = classify_role(title)
        org_map[role_type].append({
            "name": name,
            "title": title,
            "priority": priority,
            "access": access_desc
        })

        # Si es objetivo de alta prioridad, añadir a la lista especial
        if priority >= 4:
            priority_targets.append({
                "name": name,
                "title": title,
                "role_type": role_type,
                "priority": priority,
                "access": access_desc
            })

        # Buscar tecnologías en skills y descripción del perfil
        full_text = f"{skills} {description}"
        for tech in tech_keywords:
            if tech in full_text:
                tech_stack[tech] += 1

    return org_map, priority_targets, dict(tech_stack)

def print_report(org_map, priority_targets, tech_stack, company_name):
    """Imprime el reporte de reconocimiento de LinkedIn."""

    print(f"\n{'='*65}")
    print(f"  RECONOCIMIENTO LINKEDIN — {company_name.upper()}")
    print(f"{'='*65}")

    total = sum(len(v) for v in org_map.values())
    print(f"\n  Empleados totales analizados: {total}")

    # Objetivos prioritarios — ordenados por prioridad
    print(f"\n{'─'*65}")
    print(f"  OBJETIVOS DE ALTA PRIORIDAD ({len(priority_targets)})")
    print(f"{'─'*65}")

    for target in sorted(priority_targets, key=lambda x: x["priority"], reverse=True):
        stars = "★" * target["priority"]
        print(f"\n  {stars} {target['name']}")
        print(f"     Cargo:   {target['title']}")
        print(f"     Tipo:    {target['role_type'].replace('_', ' ').title()}")
        print(f"     Acceso:  {target['access']}")

    # Distribución por departamento
    print(f"\n{'─'*65}")
    print(f"  DISTRIBUCIÓN POR DEPARTAMENTO")
    print(f"{'─'*65}")

    for role_type, members in sorted(org_map.items(),
                                      key=lambda x: len(x[1]), reverse=True):
        print(f"\n  {role_type.replace('_', ' ').title()} ({len(members)}):")
        for member in members[:5]:  # Mostrar solo los primeros 5
            print(f"    - {member['name']} — {member['title']}")
        if len(members) > 5:
            print(f"    ... y {len(members)-5} más")

    # Stack tecnológico inferido de los perfiles
    if tech_stack:
        print(f"\n{'─'*65}")
        print(f"  STACK TECNOLÓGICO INFERIDO DE LOS PERFILES")
        print(f"{'─'*65}")

        sorted_tech = sorted(tech_stack.items(), key=lambda x: x[1], reverse=True)
        for tech, count in sorted_tech[:20]:
            bar = "▓" * min(count, 20)
            print(f"  {tech:<35} {bar} ({count} perfiles)")

# Ejemplo de uso
if __name__ == "__main__":
    # Ejemplo con datos ficticios de demostración
    sample_employees = [
        {"name": "John Smith", "title": "CISO", "skills": "splunk palo alto crowdstrike", "description": ""},
        {"name": "María García", "title": "IT Director", "skills": "active directory azure ad vmware", "description": ""},
        {"name": "Carlos López", "title": "System Administrator", "skills": "windows server active directory gpo", "description": ""},
        {"name": "Ana Martínez", "title": "Cloud Engineer", "skills": "aws ec2 terraform kubernetes", "description": ""},
        {"name": "Pedro Sánchez", "title": "IT Helpdesk", "skills": "windows office365 support", "description": ""},
        {"name": "Laura González", "title": "Finance Director", "skills": "sap oracle", "description": ""},
        {"name": "Roberto Fernández", "title": "Senior Developer", "skills": "java python github jenkins", "description": ""},
        {"name": "Isabel Ruiz", "title": "HR Manager", "skills": "workday recruitment", "description": ""},
    ]

    org_map, priority_targets, tech_stack = analyze_employees(sample_employees)
    print_report(org_map, priority_targets, tech_stack, "Example Corp")
```

---

## 5. Enumeración de tecnologías desde LinkedIn

Una de las aplicaciones más directas de LinkedIn OSINT para reconocimiento técnico es inferir el stack tecnológico completo de la empresa a partir de las habilidades y descripciones de los empleados. Este proceso es completamente pasivo y puede dar una imagen muy precisa de la infraestructura.

### Metodología de inferencia tecnológica

```bash
# 1. Buscar administradores de sistema y ver qué tecnologías listan
site:linkedin.com/in "Example Corp" "system administrator" OR "sysadmin"

# Lo que aparezca en sus skills y descripciones revela:
# - Sistema operativo predominante (Windows vs Linux)
# - Plataforma de virtualización (VMware, Hyper-V)
# - Directorio (Active Directory, OpenLDAP, Azure AD)
# - Herramientas de monitorización (Nagios, Zabbix, PRTG, SolarWinds)

# 2. Buscar ingenieros cloud para identificar el proveedor
site:linkedin.com/in "Example Corp" "cloud" OR "devops" OR "AWS" OR "Azure"
# AWS vs Azure vs GCP — cuál menciona más empleados

# 3. Buscar el equipo de seguridad para conocer las defensas
site:linkedin.com/in "Example Corp" "security engineer" OR "SOC analyst"
# Qué SIEM, qué EDR, qué firewall, qué herramientas de análisis

# 4. Buscar DBAs para identificar las bases de datos
site:linkedin.com/in "Example Corp" "DBA" OR "database administrator"
# Oracle, SQL Server, MySQL, PostgreSQL, MongoDB

# 5. Buscar el equipo de red
site:linkedin.com/in "Example Corp" "network engineer" OR "network administrator"
# Cisco, Juniper, Palo Alto, Fortinet, F5
```

### Plantilla de informe tecnológico inferido

Después de este proceso, el informe de reconocimiento incluye una sección como esta:

```markdown
## Stack Tecnológico Inferido — Example Corp

### Sistema Operativo y Virtualización
- Windows Server (mencionado por 15 sysadmins)
- VMware vSphere/ESXi (mencionado por 8 ingenieros)
- Hyper-V como secundario (3 menciones)

### Directorio y Gestión de Identidad
- Active Directory (mencionado por 12 perfiles)
- Azure AD / Entra ID (8 menciones — entorno híbrido probable)
- No se detecta Okta ni Ping Identity

### Cloud
- AWS primario (12 ingenieros cloud con certificaciones AWS)
- Azure secundario (4 menciones, probablemente Microsoft 365)
- GCP no detectado

### Seguridad
- SIEM: Splunk (4 analistas SOC lo mencionan)
- EDR: CrowdStrike Falcon (3 menciones en ofertas de trabajo)
- Firewall: Palo Alto NGFW (2 admins de red certificados PCNSE)
- Sin detección de herramientas de PAM (Privileged Access Management)

### Bases de Datos
- Oracle (2 DBAs certificados OCP)
- SQL Server (5 menciones)

### Implicaciones para el Engagement
- Entorno AD híbrido: ataques Kerberoasting y Pass-the-Hash son vectores relevantes
- CrowdStrike activo: payloads deben evadir Falcon
- Splunk como SIEM: las alertas basadas en reglas son el mecanismo de detección principal
- Sin PAM visible: probablemente no hay gestión centralizada de cuentas privilegiadas
```

---

## 6. Generación de listas de usuarios para password spraying

La combinación de nombres de empleados extraídos de LinkedIn con el patrón de emails inferido de otras fuentes (Hunter.io, theHarvester) da una lista de usuarios válidos para probar acceso en servicios expuestos.

### Proceso completo

```python
#!/usr/bin/env python3
"""
user_gen.py — Genera variantes de nombres de usuario desde una lista de empleados
Combina datos de LinkedIn con el patrón de emails de la empresa
"""

import re
import unicodedata

def normalize_name(name):
    """
    Normaliza un nombre para generar variantes de usuario:
    - Convierte a minúsculas
    - Elimina acentos y caracteres especiales
    - Elimina caracteres no válidos en usernames
    """
    # Convertir a minúsculas
    name = name.lower()
    # Eliminar acentos (ñ → n, á → a, etc.)
    normalized = unicodedata.normalize('NFD', name)
    ascii_name = ''.join(c for c in normalized
                        if unicodedata.category(c) != 'Mn')
    # Reemplazar ñ explícitamente (puede no normalizarse bien)
    ascii_name = ascii_name.replace('ñ', 'n')
    # Eliminar caracteres que no son letras, números o espacios
    ascii_name = re.sub(r"[^a-z0-9\s\-']", '', ascii_name)
    return ascii_name.strip()

def generate_username_variants(full_name, domain=None):
    """
    Genera todas las variantes comunes de nombre de usuario
    para un nombre completo dado.

    Si se proporciona domain, también genera variantes de email.
    """
    normalized = normalize_name(full_name)
    parts = normalized.split()

    if len(parts) < 2:
        return [normalized]  # Solo un nombre, devolver tal cual

    first = parts[0]
    last = parts[-1]  # Último apellido si hay varios
    first_initial = first[0]
    last_initial = last[0]

    variants = [
        # Formatos más comunes en entornos corporativos Windows/AD
        f"{first_initial}{last}",           # jsmith
        f"{first}.{last}",                  # john.smith
        f"{first}_{last}",                  # john_smith
        f"{first}{last}",                   # johnsmith
        f"{last}.{first}",                  # smith.john
        f"{last}{first_initial}",           # smithj
        f"{first_initial}.{last}",          # j.smith
        f"{last}_{first}",                  # smith_john
        f"{first}",                         # john (menos común)
        f"{last}",                          # smith (menos común)
        f"{first_initial}{last_initial}",   # js (poco común)
    ]

    # Si hay segundo nombre o segundo apellido, añadir variantes con ellos
    if len(parts) > 2:
        middle = parts[1] if len(parts) >= 3 else ""
        middle_initial = middle[0] if middle else ""
        if middle_initial:
            variants.extend([
                f"{first_initial}{middle_initial}{last}",  # jjsmith
                f"{first}.{middle_initial}.{last}",        # john.j.smith
            ])

    # Generar también variantes de email si se proporciona el dominio
    if domain:
        email_variants = [f"{v}@{domain}" for v in variants[:8]]  # Top 8 formatos
        return variants, email_variants

    return variants

def process_employee_list(employees_file, domain, output_prefix="users"):
    """
    Procesa una lista de empleados y genera archivos de usuarios por formato.
    employees_file: archivo con un nombre por línea
    domain: dominio de la empresa (example.com)
    output_prefix: prefijo para los archivos de salida
    """
    with open(employees_file) as f:
        names = [line.strip() for line in f if line.strip()]

    print(f"[*] Procesando {len(names)} empleados...")
    print(f"[*] Dominio: {domain}")

    # Generar todas las variantes para todos los empleados
    all_variants_by_format = {
        "firstlast": [],          # jsmith
        "first.last": [],         # john.smith
        "first_last": [],         # john_smith
        "firstlast_nospace": [],  # johnsmith
        "last.first": [],         # smith.john
        "lastfirst": [],          # smithj
        "first.last_email": [],   # john.smith@example.com
        "firstlast_email": [],    # jsmith@example.com
    }

    for name in names:
        normalized = normalize_name(name)
        parts = normalized.split()
        if len(parts) < 2:
            continue

        first = parts[0]
        last = parts[-1]
        fi = first[0]

        all_variants_by_format["firstlast"].append(f"{fi}{last}")
        all_variants_by_format["first.last"].append(f"{first}.{last}")
        all_variants_by_format["first_last"].append(f"{first}_{last}")
        all_variants_by_format["firstlast_nospace"].append(f"{first}{last}")
        all_variants_by_format["last.first"].append(f"{last}.{first}")
        all_variants_by_format["lastfirst"].append(f"{last}{fi}")
        all_variants_by_format["first.last_email"].append(f"{first}.{last}@{domain}")
        all_variants_by_format["firstlast_email"].append(f"{fi}{last}@{domain}")

    # Guardar cada formato en un archivo separado
    for format_name, users in all_variants_by_format.items():
        if not users:
            continue
        filename = f"{output_prefix}_{format_name}.txt"
        with open(filename, "w") as f:
            for user in users:
                f.write(user + "\n")
        print(f"  [+] {filename}: {len(users)} usuarios")

    print(f"\n[*] Archivos generados con prefijo: {output_prefix}_")
    print(f"[*] Para password spraying usar con CrackMapExec o Spray:")
    print(f"    crackmapexec smb target_ip -u {output_prefix}_firstlast.txt -p 'Winter2024!'")
    print(f"    crackmapexec smb target_ip -u {output_prefix}_first.last.txt -p 'Winter2024!'")

# Ejemplo de uso
if __name__ == "__main__":
    # Lista de empleados de ejemplo (extraídos de LinkedIn)
    test_names = [
        "John Smith",
        "María García López",
        "Carlos Martínez",
        "Ana Rodríguez",
        "Ángel Hernández",
        "Sofía Müller",
        "François Dupont",
    ]

    # Guardar en archivo temporal para la demo
    with open("/tmp/test_employees.txt", "w") as f:
        for name in test_names:
            f.write(name + "\n")

    process_employee_list("/tmp/test_employees.txt", "example.com", "ejemplo")
```

---

## 7. Cheatsheet de referencia rápida

```bash
# ── GOOGLE DORKS PARA LINKEDIN ────────────────────────────────────────────
site:linkedin.com/in "Empresa" "cargo"              # Empleados por cargo
site:linkedin.com/in "Empresa" "CISO" OR "CTO"      # Directivos
site:linkedin.com/in "Empresa" "sysadmin"           # Administradores
site:linkedin.com/in "Empresa" "AWS" OR "Azure"     # Stack cloud
site:linkedin.com/in "Empresa" "Splunk" "security"  # Equipo de seguridad
site:linkedin.com/jobs "Empresa" "security"         # Ofertas de trabajo
site:linkedin.com/company/empresa/people/           # Página de empleados

# ── HERRAMIENTAS ──────────────────────────────────────────────────────────
# linkedin2username
git clone https://github.com/initstring/linkedin2username
python3 linkedin2username.py -u EMAIL -c "Empresa" -p 10

# CrossLinked (sin cuenta de LinkedIn)
pip3 install crosslinked
crosslinked -f '{f}{last}' 'Empresa' -o usuarios.txt
crosslinked -f '{first}.{last}@empresa.com' 'Empresa' -o emails.txt

# ── FORMATOS DE USERNAME MÁS COMUNES ─────────────────────────────────────
# jsmith        → {f}{last}           (muy común en AD)
# john.smith    → {first}.{last}      (muy común en email)
# john_smith    → {first}_{last}
# johnsmith     → {first}{last}
# smith.john    → {last}.{first}
# smithj        → {last}{f}
# j.smith       → {f}.{last}

# ── INFERENCIA TECNOLÓGICA ────────────────────────────────────────────────
# Buscar en Google por tecnología + empresa:
site:linkedin.com/in "Empresa" "Active Directory"  → Directorio
site:linkedin.com/in "Empresa" "CrowdStrike"       → EDR
site:linkedin.com/in "Empresa" "Splunk"            → SIEM
site:linkedin.com/in "Empresa" "Palo Alto"         → Firewall
site:linkedin.com/in "Empresa" "VMware" OR "ESXi"  → Virtualización
site:linkedin.com/in "Empresa" "AWS" certif*       → Cloud provider

# ── PASSWORD SPRAYING CON LISTA GENERADA ─────────────────────────────────
# Con CrackMapExec contra SMB
crackmapexec smb IP -u users_firstlast.txt -p 'Password1' --continue-on-success

# Con CrackMapExec contra OWA/Exchange
crackmapexec http IP -u users_first.last.txt -p 'Password1' --path /owa/

# Con Spray para Office 365
spray -u users.txt -p 'Password1!' -d example.com -c

# ── BUENAS PRÁCTICAS EN RECONOCIMIENTO DE LINKEDIN ───────────────────────
# 1. Usar una cuenta de LinkedIn creada específicamente para reconocimiento
#    (no tu cuenta personal)
# 2. No conectarte con empleados del objetivo
# 3. Activar el modo de navegación privada en LinkedIn antes de ver perfiles
# 4. No superar las ~100 visitas de perfil por día para no activar alertas
# 5. Documentar toda la información con capturas de pantalla como evidencia
# 6. En el reporte, anonimizar nombres reales de empleados si el cliente no
#    es la propia empresa (por ejemplo, en bug bounty con scope amplio)

# ── RECURSOS ──────────────────────────────────────────────────────────────
https://github.com/initstring/linkedin2username    → linkedin2username
https://github.com/m8sec/crosslinked               → CrossLinked
https://www.linkedin.com/company/                  → Páginas de empresa
https://hunter.io                                  → Verificar patrón de emails
```
