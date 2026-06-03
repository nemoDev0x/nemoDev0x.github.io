---
layout: post
title: "MITRE ATT&CK — Framework de Tácticas y Técnicas Reales"
date: 2025-05-30
categories: [vulnerabilidades]
tags: [mitre, attack, framework, tácticas, técnicas, red-team, blue-team, threat-intelligence, ttps]
description: "Guía profesional de MITRE ATT&CK: estructura del framework, tácticas, técnicas, sub-técnicas, grupos APT, navegación de la matriz y aplicación en Red Team, Blue Team y Threat Intelligence."
---

## MITRE ATT&CK — el lenguaje común de la ciberseguridad ofensiva y defensiva

Antes de que existiera MITRE ATT&CK, documentar un ataque era un proceso sin estándares. Un equipo de Red Team describía lo que hacía en su propio lenguaje, el equipo de Blue Team usaba otra terminología, y los informes de Threat Intelligence de distintos proveedores eran imposibles de comparar porque cada uno categorizaba las técnicas de forma diferente.

MITRE ATT&CK (Adversarial Tactics, Techniques & Common Knowledge) cambió esto. Es un framework de conocimiento sobre el comportamiento de los adversarios — una base de datos estructurada y pública que documenta exactamente cómo los atacantes reales operan una vez han ganado acceso a un sistema. No es teoría — cada técnica del framework está documentada con ejemplos reales de grupos APT que la han usado, con referencias a informes de amenazas, con procedimientos de detección y con mitigaciones.

Lo que hace a ATT&CK tan valioso no es una técnica concreta sino el **vocabulario común** que proporciona. Cuando un red teamer dice "usé T1055.001" (Process Injection via DLL Injection), el blue teamer que lee el informe sabe exactamente qué ocurrió, cómo detectarlo, y qué controles deberían haberlo prevenido. Este lenguaje compartido es la base de la colaboración entre equipos ofensivos y defensivos.

---

## 1. Estructura del framework

ATT&CK está organizado en tres matrices principales según el entorno:

```
MITRE ATT&CK
├── Enterprise  → Sistemas Windows, Linux, macOS, cloud, red, contenedores
├── Mobile      → Android e iOS
└── ICS         → Sistemas de control industrial (SCADA, PLC)
```

La mayoría del trabajo en pentesting corporativo usa la matriz **Enterprise**, que es la más completa y la que cubriremos en profundidad.

### La jerarquía: Tácticas → Técnicas → Sub-técnicas

La estructura es jerárquica y responde a preguntas diferentes:

```
TÁCTICA (¿Por qué?)
│   Define el objetivo del atacante — qué quiere conseguir
│   Ejemplo: Persistence (mantener acceso)
│
└── TÉCNICA (¿Cómo en general?)
    │   Define el método general para lograr ese objetivo
    │   Ejemplo: T1078 — Valid Accounts
    │
    └── SUB-TÉCNICA (¿Cómo específicamente?)
        Define la implementación concreta de la técnica
        Ejemplo: T1078.002 — Domain Accounts
```

Esta jerarquía permite hablar a distintos niveles de granularidad según el contexto. En un resumen ejecutivo hablas de tácticas. En un informe técnico hablas de técnicas. En el análisis forense hablas de sub-técnicas con los procedimientos exactos.

---

## 2. Las 14 tácticas de ATT&CK Enterprise

Las tácticas representan el "porqué" de cada acción del atacante. Están ordenadas aproximadamente en el orden en que ocurren durante un ataque, desde el acceso inicial hasta el impacto final:

```
ID      TÁCTICA                   DESCRIPCIÓN
────────────────────────────────────────────────────────────────
TA0043  Reconnaissance            Recopilar información sobre el objetivo
TA0042  Resource Development      Preparar infraestructura para el ataque
TA0001  Initial Access            Ganar el primer punto de entrada a la red
TA0002  Execution                 Ejecutar código malicioso en el sistema
TA0003  Persistence               Mantener el acceso tras reinicios o cambios
TA0004  Privilege Escalation      Obtener permisos más elevados
TA0005  Defense Evasion           Evitar la detección por los sistemas de seguridad
TA0006  Credential Access         Robar credenciales (hashes, contraseñas, tokens)
TA0007  Discovery                 Aprender sobre el entorno comprometido
TA0008  Lateral Movement          Moverse a otros sistemas de la red
TA0009  Collection                Recopilar datos de interés para el objetivo
TA0010  Exfiltration              Extraer datos hacia el exterior
TA0011  Command and Control       Comunicarse con los sistemas comprometidos
TA0040  Impact                    Manipular, interrumpir o destruir datos/sistemas
```

Entender estas tácticas permite estructurar un engagement de Red Team de forma que cubra el ciclo completo de ataque, desde el reconocimiento hasta el impacto.

---

## 3. Técnicas y sub-técnicas clave por táctica

No hay espacio para cubrir las más de 400 técnicas del framework, pero estas son las más relevantes en engagements de pentesting corporativo:

### TA0001 — Initial Access

```
T1566      Phishing
  T1566.001   Spearphishing Attachment    → Email con adjunto malicioso
  T1566.002   Spearphishing Link          → Email con enlace malicioso
  T1566.003   Spearphishing via Service   → Phishing via LinkedIn, Teams, etc.

T1190      Exploit Public-Facing Application
           → Explotar servicio expuesto (web app, VPN, firewall)

T1078      Valid Accounts
  T1078.001   Default Accounts           → Credenciales por defecto
  T1078.002   Domain Accounts            → Cuentas de dominio comprometidas
  T1078.003   Local Accounts             → Cuentas locales comprometidas
  T1078.004   Cloud Accounts             → Cuentas cloud comprometidas

T1133      External Remote Services
           → Acceso vía VPN, RDP, Citrix expuestos
```

### TA0002 — Execution

```
T1059      Command and Scripting Interpreter
  T1059.001   PowerShell                 → El más común en Windows
  T1059.003   Windows Command Shell      → cmd.exe
  T1059.004   Unix Shell                 → bash, sh, zsh
  T1059.007   JavaScript                 → Node.js, JScript en HTA

T1053      Scheduled Task/Job
  T1053.005   Scheduled Task             → Windows Task Scheduler
  T1053.003   Cron                       → Linux cron jobs

T1047      Windows Management Instrumentation
           → WMI para ejecutar código remoto

T1204      User Execution
  T1204.001   Malicious Link             → Usuario hace clic en enlace
  T1204.002   Malicious File             → Usuario abre fichero malicioso
```

### TA0003 — Persistence

```
T1547      Boot or Logon Autostart Execution
  T1547.001   Registry Run Keys          → HKCU/HKLM\...\Run
  T1547.004   Winlogon Helper DLL        → Persistence via Winlogon

T1543      Create or Modify System Process
  T1543.003   Windows Service            → Crear servicio malicioso
  T1543.002   Systemd Service            → Linux systemd unit

T1136      Create Account
  T1136.001   Local Account              → Crear cuenta local
  T1136.002   Domain Account             → Crear cuenta de dominio

T1078      Valid Accounts               → (ver Initial Access)

T1505      Server Software Component
  T1505.003   Web Shell                  → Webshell en servidor web
```

### TA0004 — Privilege Escalation

```
T1548      Abuse Elevation Control Mechanism
  T1548.002   Bypass User Account Control → UAC bypass en Windows

T1134      Access Token Manipulation
  T1134.001   Token Impersonation/Theft  → Impersonar tokens de otros procesos
  T1134.002   Create Process with Token  → Crear proceso con token robado

T1068      Exploitation for Privilege Escalation
           → Explotar vulnerabilidad local para elevar privilegios

T1055      Process Injection
  T1055.001   Dynamic-link Library Injection → DLL injection
  T1055.002   Portable Executable Injection → PE injection
  T1055.012   Process Hollowing          → Vaciar proceso legítimo

T1611      Escape to Host
           → Escape de contenedor Docker/K8s al host
```

### TA0005 — Defense Evasion

```
T1027      Obfuscated Files or Information
  T1027.001   Binary Padding             → Añadir bytes para cambiar hash
  T1027.002   Software Packing           → UPX, custom packers
  T1027.010   Command Obfuscation        → PowerShell -enc, char concat

T1562      Impair Defenses
  T1562.001   Disable or Modify Tools    → Deshabilitar AV/EDR
  T1562.004   Disable or Modify Firewall → Modificar reglas de firewall

T1036      Masquerading
  T1036.003   Rename System Utilities    → Renombrar cmd.exe a svchost.exe
  T1036.005   Match Legitimate Name or Location → Copiar a %System%

T1070      Indicator Removal
  T1070.001   Clear Windows Event Logs   → Limpiar logs de Windows
  T1070.002   Clear Linux or Mac System Logs → Limpiar /var/log/
  T1070.004   File Deletion             → Eliminar herramientas usadas

T1218      System Binary Proxy Execution
           → Usar binarios legítimos de Windows para ejecutar código
           → LOLBAS: rundll32, regsvr32, mshta, certutil
```

### TA0006 — Credential Access

```
T1003      OS Credential Dumping
  T1003.001   LSASS Memory              → Mimikatz sekurlsa::logonpasswords
  T1003.002   Security Account Manager  → SAM database dump
  T1003.003   NTDS                      → ntds.dit del controlador de dominio
  T1003.004   LSA Secrets               → Secretos en el registro

T1558      Steal or Forge Kerberos Tickets
  T1558.003   Kerberoasting              → Solicitar TGS y crackear offline
  T1558.004   AS-REP Roasting            → Usuarios sin preautenticación Kerberos

T1552      Unsecured Credentials
  T1552.001   Credentials In Files       → Contraseñas en ficheros de config
  T1552.002   Credentials in Registry    → Contraseñas en el registro
  T1552.004   Private Keys               → Claves SSH/SSL sin proteger

T1110      Brute Force
  T1110.001   Password Guessing          → Probar contraseñas comunes
  T1110.003   Password Spraying          → Una contraseña, muchos usuarios
  T1110.004   Credential Stuffing        → Credenciales de brechas previas
```

### TA0007 — Discovery

```
T1087      Account Discovery
  T1087.001   Local Account             → Enumerar cuentas locales
  T1087.002   Domain Account            → Enumerar cuentas de dominio

T1069      Permission Groups Discovery
  T1069.001   Local Groups              → Grupos locales
  T1069.002   Domain Groups             → Grupos de dominio (Domain Admins, etc.)

T1082      System Information Discovery
           → systeminfo, uname -a, hostname

T1083      File and Directory Discovery
           → dir, ls, find

T1046      Network Service Discovery
           → Nmap, netstat, arp

T1135      Network Share Discovery
           → net share, smbclient -L

T1018      Remote System Discovery
           → net view, ping sweep, nmap
```

### TA0008 — Lateral Movement

```
T1021      Remote Services
  T1021.001   Remote Desktop Protocol   → RDP lateral movement
  T1021.002   SMB/Windows Admin Shares  → PsExec, net use
  T1021.004   SSH                       → SSH con claves robadas
  T1021.006   Windows Remote Management → WinRM / Evil-WinRM

T1550      Use Alternate Authentication Material
  T1550.002   Pass the Hash             → NTLM hash sin crackear
  T1550.003   Pass the Ticket           → Ticket Kerberos robado

T1563      Remote Service Session Hijacking
  T1563.001   SSH Hijacking             → Secuestrar sesión SSH activa
  T1563.002   RDP Hijacking             → Secuestrar sesión RDP activa
```

---

## 4. ATT&CK Navigator — visualizar la cobertura

ATT&CK Navigator es la herramienta web oficial de MITRE para visualizar y trabajar con la matriz. Permite colorear técnicas según si han sido usadas, detectadas, mitigadas, o cualquier otro criterio personalizado.

```
Acceso: https://mitre-attack.github.io/attack-navigator/

Usos principales:
├── Red Team:  marcar las técnicas usadas en el engagement
├── Blue Team: marcar las técnicas que el SIEM detecta (coverage)
├── Threat Intel: visualizar las TTPs de un grupo APT específico
└── Gap Analysis: comparar cobertura defensiva vs técnicas de amenazas reales
```

### Crear un layer personalizado con Python

La API de ATT&CK permite crear layers del Navigator programáticamente:

```python
#!/usr/bin/env python3
"""
attack_navigator_layer.py
Genera un layer de ATT&CK Navigator con las técnicas usadas en un engagement.
El JSON generado se importa directamente en el Navigator.
"""

import json
from datetime import datetime

def create_layer(name, description, techniques_used):
    """
    Crea un layer para ATT&CK Navigator.

    techniques_used: lista de dicts con:
        - technique_id: ej. "T1059.001"
        - tactic: ej. "execution"
        - comment: descripción de cómo se usó
        - score: 1-100 (intensidad de uso, afecta el color)
    """
    layer = {
        "name": name,
        "versions": {
            "attack": "14",
            "navigator": "4.9",
            "layer": "4.5"
        },
        "domain": "enterprise-attack",
        "description": description,
        "filters": {
            "platforms": ["Windows", "Linux", "macOS"]
        },
        "sorting": 0,
        "layout": {
            "layout": "side",
            "showID": True,
            "showName": True
        },
        "hideDisabled": False,
        "techniques": [],
        "gradient": {
            "colors": ["#ff6666", "#ff0000"],
            "minValue": 0,
            "maxValue": 100
        },
        "legendItems": [
            {"label": "Técnica usada en engagement", "color": "#ff6666"},
            {"label": "Técnica crítica/alto impacto", "color": "#ff0000"}
        ],
        "metadata": [],
        "showTacticRowBackground": True,
        "tacticRowBackground": "#dddddd",
        "selectTechniquesAcrossTactics": True,
        "selectSubtechniquesWithParent": False
    }

    for tech in techniques_used:
        entry = {
            "techniqueID": tech["technique_id"],
            "tactic": tech.get("tactic", ""),
            "score": tech.get("score", 50),
            "color": "",
            "comment": tech.get("comment", ""),
            "enabled": True,
            "metadata": [],
            "links": [],
            "showSubtechniques": False
        }
        layer["techniques"].append(entry)

    return layer

# Ejemplo: documentar las técnicas usadas en un engagement de Red Team
techniques_engagement = [
    {
        "technique_id": "T1566.001",
        "tactic": "initial-access",
        "comment": "Spearphishing con adjunto Word malicioso (.docm)",
        "score": 80
    },
    {
        "technique_id": "T1059.001",
        "tactic": "execution",
        "comment": "PowerShell para descargar y ejecutar payload en memoria",
        "score": 90
    },
    {
        "technique_id": "T1055.001",
        "tactic": "defense-evasion",
        "comment": "DLL injection en proceso svchost.exe para evasión de EDR",
        "score": 85
    },
    {
        "technique_id": "T1003.001",
        "tactic": "credential-access",
        "comment": "Mimikatz sekurlsa::logonpasswords en servidor comprometido",
        "score": 100
    },
    {
        "technique_id": "T1558.003",
        "tactic": "credential-access",
        "comment": "Kerberoasting — 3 cuentas de servicio crackeadas",
        "score": 95
    },
    {
        "technique_id": "T1021.002",
        "tactic": "lateral-movement",
        "comment": "PsExec con hashes NTLM para movimiento lateral",
        "score": 85
    },
    {
        "technique_id": "T1550.002",
        "tactic": "lateral-movement",
        "comment": "Pass-the-Hash para acceder a 12 sistemas adicionales",
        "score": 90
    },
    {
        "technique_id": "T1003.003",
        "tactic": "credential-access",
        "comment": "DCSync attack — volcado del ntds.dit del DC",
        "score": 100
    },
]

layer = create_layer(
    name="Red Team Engagement — Example Corp — Mayo 2025",
    description="Técnicas usadas durante el engagement de Red Team autorizado.",
    techniques_used=techniques_engagement
)

output_file = "engagement_layer.json"
with open(output_file, "w") as f:
    json.dump(layer, f, indent=2)

print(f"[+] Layer generado: {output_file}")
print(f"[+] Importar en: https://mitre-attack.github.io/attack-navigator/")
print(f"[+] File → Open Existing Layer → Upload from local")
print(f"\n[*] Técnicas documentadas: {len(techniques_engagement)}")
```

---

## 5. Grupos APT en ATT&CK

ATT&CK documenta más de 130 grupos de actores de amenaza (APT) con sus TTPs conocidas. Esta información es valiosa para Threat Intelligence y para simular ataques reales en ejercicios de Red Team.

```python
#!/usr/bin/env python3
"""
Consulta la API de ATT&CK para obtener las técnicas de un grupo APT.
Requiere: pip3 install mitreattack-python
"""

from mitreattack.stix20 import MitreAttackData

def get_apt_techniques(group_name):
    """
    Obtiene las técnicas asociadas a un grupo APT desde ATT&CK STIX.
    """
    # Cargar los datos de ATT&CK (descarga automáticamente si no existe)
    mitre_data = MitreAttackData("enterprise-attack.json")

    # Buscar el grupo
    groups = mitre_data.get_groups()
    target_group = None

    for group in groups:
        names = [group.get("name", "")]
        aliases = group.get("aliases", [])
        all_names = names + aliases

        if any(group_name.lower() in n.lower() for n in all_names):
            target_group = group
            break

    if not target_group:
        print(f"[-] Grupo '{group_name}' no encontrado")
        return

    group_id = target_group["external_references"][0]["external_id"]
    print(f"[+] Grupo: {target_group['name']} ({group_id})")
    print(f"    Aliases: {', '.join(target_group.get('aliases', []))}")
    print(f"    Descripción: {target_group.get('description', '')[:200]}...")

    # Obtener técnicas del grupo
    techniques = mitre_data.get_techniques_used_by_group(target_group["id"])

    print(f"\n[+] Técnicas conocidas ({len(techniques)}):")
    print(f"{'─'*60}")

    # Agrupar por táctica
    by_tactic = {}
    for tech in techniques:
        obj = tech["object"]
        tactic_phases = obj.get("kill_chain_phases", [])
        for phase in tactic_phases:
            tactic = phase["phase_name"]
            if tactic not in by_tactic:
                by_tactic[tactic] = []
            tech_id = obj["external_references"][0]["external_id"]
            tech_name = obj["name"]
            by_tactic[tactic].append(f"{tech_id}: {tech_name}")

    for tactic, techs in sorted(by_tactic.items()):
        print(f"\n  [{tactic.upper()}]")
        for t in sorted(techs):
            print(f"    {t}")

# Ejemplo: ver técnicas conocidas de APT29 (Cozy Bear / SVR)
# get_apt_techniques("APT29")

# Grupos APT relevantes en ATT&CK:
APT_GROUPS = {
    "APT28":  "Fancy Bear — GRU ruso, muy activo en phishing y lateral movement",
    "APT29":  "Cozy Bear — SVR ruso, ataques a cadena de suministro (SolarWinds)",
    "APT41":  "Winnti Group — China, espionaje y crimen financiero",
    "Lazarus": "Corea del Norte, ataques a bancos y criptomonedas",
    "FIN7":   "Grupo criminal, ataques a sector retail y hostelería",
    "Carbanak":"Grupo criminal, ataques a entidades bancarias",
    "APT1":   "Comment Crew — China, robo de propiedad intelectual",
    "Sandworm":"GRU ruso, ataques a infraestructura crítica (NotPetya, Ukraine)",
}

for apt, desc in APT_GROUPS.items():
    print(f"  {apt:<12} → {desc}")
```

---

## 6. ATT&CK en el ciclo de un engagement

### Cómo usar ATT&CK en Red Team

```
FASE DE PLANIFICACIÓN:
→ Seleccionar el perfil de amenaza a simular (ej: APT29, FIN7)
→ Identificar las técnicas que ese grupo usa típicamente
→ Planificar las TTPs que se van a ejecutar en el engagement
→ Crear el threat model basado en ATT&CK

FASE DE EJECUCIÓN:
→ Documentar cada técnica usada con su ID de ATT&CK
→ Registrar el procedimiento exacto (qué herramienta, qué comando)
→ Anotar si fue detectada o no por las defensas del cliente

FASE DE REPORTE:
→ Generar un layer de ATT&CK Navigator con las técnicas usadas
→ Comparar con la cobertura defensiva del cliente
→ Identificar gaps — técnicas usadas sin detección
→ Priorizar recomendaciones según el impacto de cada técnica
```

### Cómo usar ATT&CK en Blue Team

```
DETECCIÓN:
→ Mapear las reglas de detección del SIEM a técnicas ATT&CK
→ Identificar qué porcentaje de la matriz tiene cobertura
→ Priorizar implementar detecciones para técnicas de alto riesgo

RESPUESTA A INCIDENTES:
→ Usar ATT&CK para categorizar las técnicas vistas durante el incidente
→ Buscar otras técnicas del mismo grupo APT que puedan haberse usado
→ Extender la búsqueda de indicadores basándose en el perfil del grupo

THREAT HUNTING:
→ Seleccionar técnicas de la matriz sin reglas de detección
→ Crear hipótesis de caza basadas en esas técnicas
→ Buscar evidencias de uso en logs históricos
```

---

## 7. Cheatsheet de referencia rápida

```
# ── ESTRUCTURA DEL FRAMEWORK ──────────────────────────────────────────────
Tácticas (14): Reconnaissance → Resource Development → Initial Access
               → Execution → Persistence → Privilege Escalation
               → Defense Evasion → Credential Access → Discovery
               → Lateral Movement → Collection → Exfiltration
               → Command and Control → Impact

Técnicas: ~200 técnicas principales
Sub-técnicas: ~400 sub-técnicas

# ── TÉCNICAS MÁS COMUNES EN PENTESTING ───────────────────────────────────
T1566.001  Spearphishing Attachment       → Initial Access
T1190      Exploit Public-Facing App      → Initial Access
T1059.001  PowerShell                     → Execution
T1055.001  DLL Injection                  → Defense Evasion / Priv Esc
T1003.001  LSASS Memory (Mimikatz)        → Credential Access
T1558.003  Kerberoasting                  → Credential Access
T1550.002  Pass the Hash                  → Lateral Movement
T1021.002  SMB/Admin Shares (PsExec)      → Lateral Movement
T1003.003  NTDS (DCSync)                  → Credential Access
T1078.002  Domain Accounts               → Multiple tactics
T1547.001  Registry Run Keys             → Persistence
T1505.003  Web Shell                      → Persistence
T1070.001  Clear Windows Event Logs       → Defense Evasion

# ── RECURSOS WEB ──────────────────────────────────────────────────────────
https://attack.mitre.org                          → ATT&CK principal
https://mitre-attack.github.io/attack-navigator/  → Navigator (visualización)
https://attack.mitre.org/groups/                  → Grupos APT
https://attack.mitre.org/software/               → Herramientas/malware
https://attack.mitre.org/mitigations/            → Mitigaciones
https://car.mitre.org                             → Analytics (detecciones)
https://d3fend.mitre.org                         → D3FEND (contramedidas)

# ── LIBRERÍAS PYTHON ──────────────────────────────────────────────────────
pip3 install mitreattack-python   → API oficial de ATT&CK
pip3 install attackcti            → Librería alternativa ATT&CK CTI

# ── HERRAMIENTAS ──────────────────────────────────────────────────────────
ATT&CK Navigator  → Visualización de la matriz
VECTR             → Tracking de ejercicios Red Team con ATT&CK
Atomic Red Team   → Tests atómicos para validar detecciones
Caldera           → Framework de emulación de adversarios (MITRE)
```
