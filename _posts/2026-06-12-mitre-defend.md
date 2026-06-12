---
layout: post
title: "MITRE D3FEND — Framework de Defensa y Contramedidas"
date: 2026-06-12
categories: [vulnerabilidades]
tags: [d3fend, mitre, defensa, blue-team, contramedidas, ataque-defensa, hardening, detección]
description: "Guía profesional de MITRE D3FEND: estructura del framework, mapeo de técnicas defensivas frente a ATT&CK, navegación del grafo de conocimiento y aplicación práctica en hardening y Blue Team."
---

## D3FEND como contrapartida defensiva de ATT&CK

MITRE ATT&CK describe con enorme detalle cómo atacan los adversarios — sus tácticas, técnicas y procedimientos. Pero durante años faltaba el equivalente desde el lado defensivo: un vocabulario igualmente riguroso para describir las **contramedidas**. Cuando un equipo de seguridad decía "tenemos EDR" o "monitorizamos los logs", esas afirmaciones eran demasiado vagas para evaluar si realmente cubrían las técnicas de ataque relevantes.

D3FEND (Detection, Denial, and Disruption Framework Empowering Network Defense) es el proyecto de MITRE, financiado por la NSA, que resuelve este problema. Es un framework de conocimiento que cataloga las técnicas defensivas con el mismo nivel de detalle que ATT&CK cataloga las ofensivas, y lo más importante: las **conecta entre sí**. D3FEND no es solo una lista de buenas prácticas — es un grafo de conocimiento que muestra qué técnica defensiva concreta contrarresta qué técnica ofensiva concreta de ATT&CK.

Para un pentester esto tiene un valor directo: cuando documentas que usaste T1003.001 (LSASS Memory dumping con Mimikatz) en un engagement, D3FEND te dice exactamente qué contramedidas técnicas deberían haber estado presentes — no en términos genéricos como "mejorar la seguridad de credenciales", sino con técnicas específicas como Credential Hardening (D3-CH) o Process Self-Verification. Esto transforma el hallazgo de "encontramos una vulnerabilidad" a "esta es la contramedida exacta que falta y por qué".

---

## 1. Estructura del framework

D3FEND organiza las contramedidas en categorías tácticas que reflejan el ciclo de vida de la defensa, desde el endurecimiento previo hasta la respuesta tras el compromiso:

```
D3FEND — Tácticas Defensivas

Model      → Modelado del sistema (línea base para detectar anomalías)
Harden     → Reducir la superficie de ataque antes de que ocurra nada
Detect     → Identificar actividad maliciosa en curso
Isolate    → Contener el alcance de un ataque en progreso
Deceive    → Engañar al atacante con señuelos e información falsa
Evict      → Eliminar la presencia del atacante del entorno
Restore    → Recuperar sistemas y datos al estado anterior al ataque
```

Cada táctica se divide en técnicas, y cada técnica tiene un identificador con el prefijo `D3-` seguido de un acrónimo. A diferencia de ATT&CK (que usa IDs numéricos como T1003), D3FEND usa acrónimos descriptivos: `D3-CH` es Credential Hardening, `D3-FAPA` es File Access Pattern Analysis.

### Las digital artifacts — el corazón del modelo

Lo que hace único a D3FEND es que no describe las contramedidas de forma aislada, sino a través de **digital artifacts** — los objetos técnicos sobre los que operan tanto atacantes como defensores: procesos, ficheros, credenciales, conexiones de red, claves de registro. Cada técnica ofensiva de ATT&CK manipula ciertos artifacts, y cada técnica defensiva de D3FEND protege o monitoriza esos mismos artifacts. Esa es la conexión que permite el mapeo bidireccional.

```
Ejemplo de conexión via Digital Artifact:

ATT&CK T1003.001 (LSASS Memory)
   │
   │  manipula el artifact: "Process" (proceso lsass.exe)
   │
   ▼
D3FEND D3-PSA (Process Spawn Analysis)
D3FEND D3-PA  (Process Analysis)
D3FEND D3-OSM (Operating System Monitoring)
   │
   │  estas técnicas D3FEND monitorizan el mismo artifact "Process"
   │  y pueden detectar el acceso anómalo a lsass.exe
```

---

## 2. Las técnicas D3FEND por táctica

### Harden — Endurecimiento

El endurecimiento reduce la superficie de ataque antes de que el adversario actúe. Es la primera línea de defensa y la más rentable en términos de coste/beneficio:

```
D3-CH    Credential Hardening
         → Políticas de contraseñas, MFA, rotación de credenciales
         → Contrarresta: T1110 (Brute Force), T1003 (Credential Dumping)

D3-AH    Application Hardening
         → Sandboxing, control de ejecución, validación de entrada
         → Contrarresta: T1059 (Command and Scripting Interpreter)

D3-MH    Message Hardening
         → DMARC, SPF, DKIM, filtrado de adjuntos
         → Contrarresta: T1566 (Phishing)

D3-PH    Platform Hardening
         → Deshabilitar servicios innecesarios, hardening de OS
         → Contrarresta: múltiples técnicas de Initial Access

D3-EHPV  Exclusive Hardware Architecture Protection
         → TPM, Secure Boot, virtualización segura
         → Contrarresta: T1542 (Pre-OS Boot)

D3-NTA   Network Traffic Analysis (parte de Harden en segmentación)
         → Segmentación de red, listas de control de acceso
         → Contrarresta: T1021 (Remote Services lateral movement)
```

### Detect — Detección

Las técnicas de detección identifican actividad maliciosa mientras ocurre. Esta es la categoría más extensa de D3FEND porque cubre todos los tipos de telemetría:

```
D3-PA    Process Analysis
         → Analizar comportamiento de procesos en ejecución
         → Detecta: T1055 (Process Injection), T1059 (Scripting)

D3-FCA   File Content Analysis
         → Análisis de contenido de ficheros (firmas, hashes, heurística)
         → Detecta: T1027 (Obfuscated Files), malware en disco

D3-NTA   Network Traffic Analysis
         → Inspección profunda de paquetes, detección de anomalías
         → Detecta: T1071 (Application Layer Protocol — C2)

D3-UBA   User Behavior Analysis
         → UEBA — detectar comportamiento anómalo de usuarios
         → Detecta: T1078 (Valid Accounts usado maliciosamente)

D3-SCA   System Call Analysis
         → Monitorización a nivel de syscall (eBPF, auditd)
         → Detecta: T1055, T1014 (Rootkit)

D3-ANET  Authentication Event Thresholding
         → Umbrales en eventos de autenticación
         → Detecta: T1110.003 (Password Spraying)

D3-DNSTA DNS Traffic Analysis
         → Análisis de tráfico DNS para detectar exfiltración/C2
         → Detecta: T1071.004 (DNS), T1568 (Dynamic Resolution)
```

### Isolate — Aislamiento

Las técnicas de aislamiento contienen el alcance de un ataque en progreso, limitando el movimiento lateral y la propagación:

```
D3-NI    Network Isolation
         → Aislar segmentos de red comprometidos
         → Contrarresta: T1021 (Lateral Movement)

D3-EI    Execution Isolation
         → Sandboxing, contenedores, microsegmentación de procesos
         → Contrarresta: T1055 (Process Injection)

D3-PSEP  Process Segment Execution Prevention
         → Prevenir ejecución de segmentos de memoria modificados
         → Contrarresta: T1055.012 (Process Hollowing)
```

### Deceive — Engaño

El engaño es una de las categorías más interesantes para Red Team porque define cómo se construyen los entornos de honeypot y deception que un Red Team puede encontrarse:

```
D3-DE    Decoy Environment
         → Entornos completos de honeypot
         → Diseñado para: detectar T1018 (Remote System Discovery)

D3-DO    Decoy Object
         → Ficheros, credenciales o registros señuelo
         → Diseñado para: detectar T1552 (Unsecured Credentials)
         → Si un Red Team encuentra "credenciales" en un fichero compartido
           que son demasiado fáciles de encontrar, puede ser un D3-DO

D3-DNR   Decoy Network Resource
         → Servicios de red falsos (honeypots de SMB, RDP, etc.)
         → Diseñado para: detectar T1046 (Network Service Discovery)
```

### Evict — Expulsión

Las técnicas de expulsión eliminan la presencia del atacante una vez detectado:

```
D3-AL    Access Token Manipulation Defenses
         → Revocar tokens y sesiones comprometidas
         → Contrarresta: T1134 (Access Token Manipulation)

D3-CR    Credential Rotation
         → Forzar rotación de credenciales comprometidas
         → Contrarresta: T1078, T1003 (cualquier robo de credenciales)

D3-PT    Process Termination
         → Terminar procesos maliciosos identificados
         → Contrarresta: persistencia activa
```

### Restore — Recuperación

```
D3-FR    File Restoration
         → Restaurar ficheros desde backups limpios
         → Contrarresta: T1485 (Data Destruction), ransomware

D3-SI    System Image Recovery
         → Restaurar sistemas completos desde imágenes limpias
         → Contrarresta: T1486 (Data Encrypted for Impact)
```

---

## 3. Mapeo ATT&CK ↔ D3FEND — la herramienta más útil

La verdadera potencia de D3FEND está en su capacidad de mapeo bidireccional con ATT&CK. Esto permite responder a dos preguntas fundamentales:

```
PREGUNTA 1 (perspectiva Red Team / pentester):
"Usé la técnica T1003.001 (LSASS Memory Dumping).
 ¿Qué contramedidas D3FEND debería haber encontrado?"

PREGUNTA 2 (perspectiva Blue Team):
"Tenemos implementado D3-PA (Process Analysis).
 ¿Qué técnicas de ATT&CK estamos cubriendo con esto?"
```

### Consultar el mapeo con Python

```python
#!/usr/bin/env python3
"""
d3fend_mapper.py — Consulta el mapeo ATT&CK <-> D3FEND
La API de D3FEND es un endpoint SPARQL sobre un grafo RDF.
Documentación: https://d3fend.mitre.org/api/
"""

import requests
import json

D3FEND_API = "https://d3fend.mitre.org/api/graphql"

def get_d3fend_countermeasures_for_attack(attack_technique_id):
    """
    Dado un ID de técnica ATT&CK (ej: T1003.001),
    devuelve las técnicas D3FEND que la contrarrestan.
    """
    query = """
    query GetCountermeasures($attackId: String!) {
      attackToDefend(attackId: $attackId) {
        attack {
          id
          name
        }
        defend {
          id
          name
          description
          tactic
        }
      }
    }
    """

    response = requests.post(
        D3FEND_API,
        json={
            "query": query,
            "variables": {"attackId": attack_technique_id}
        },
        timeout=20
    )

    if response.status_code == 200:
        return response.json()
    return None


# Mapeo de referencia construido manualmente para las técnicas
# más comunes en pentesting (la API oficial de D3FEND requiere
# consultas SPARQL más complejas, esto es una referencia rápida)

ATTACK_TO_D3FEND_MAP = {
    "T1003.001": {  # LSASS Memory
        "attack_name": "OS Credential Dumping: LSASS Memory",
        "countermeasures": [
            ("D3-PA",   "Process Analysis",
             "Detectar acceso anómalo al proceso lsass.exe"),
            ("D3-CH",   "Credential Hardening",
             "Credential Guard de Windows protege la memoria de LSASS"),
            ("D3-SCA",  "System Call Analysis",
             "Detectar llamadas a OpenProcess sobre lsass.exe"),
        ]
    },
    "T1566.001": {  # Spearphishing Attachment
        "attack_name": "Phishing: Spearphishing Attachment",
        "countermeasures": [
            ("D3-FCA",  "File Content Analysis",
             "Sandboxing y análisis de adjuntos antes de entrega"),
            ("D3-MH",   "Message Hardening",
             "SPF/DKIM/DMARC, filtrado de adjuntos ejecutables"),
            ("D3-UA",   "User Account Permissions",
             "Limitar permisos de macros en Office"),
        ]
    },
    "T1558.003": {  # Kerberoasting
        "attack_name": "Steal or Forge Kerberos Tickets: Kerberoasting",
        "countermeasures": [
            ("D3-CH",   "Credential Hardening",
             "Contraseñas largas y aleatorias en cuentas de servicio"),
            ("D3-ANET", "Authentication Event Thresholding",
             "Detectar solicitudes anómalas de tickets TGS"),
            ("D3-NTA",  "Network Traffic Analysis",
             "Detectar tráfico Kerberos anómalo (RC4 vs AES)"),
        ]
    },
    "T1055.001": {  # DLL Injection
        "attack_name": "Process Injection: DLL Injection",
        "countermeasures": [
            ("D3-PA",   "Process Analysis",
             "Detectar inyección de DLL en otros procesos"),
            ("D3-EI",   "Execution Isolation",
             "Sandboxing de procesos críticos"),
            ("D3-SCF",  "System Call Filtering",
             "Restringir CreateRemoteThread, WriteProcessMemory"),
        ]
    },
    "T1021.002": {  # SMB/Admin Shares
        "attack_name": "Remote Services: SMB/Windows Admin Shares",
        "countermeasures": [
            ("D3-NI",   "Network Isolation",
             "Segmentación de red — limitar SMB entre segmentos"),
            ("D3-ANET", "Authentication Event Thresholding",
             "Detectar autenticaciones SMB anómalas"),
            ("D3-NTA",  "Network Traffic Analysis",
             "Monitorizar tráfico SMB lateral inusual"),
        ]
    },
    "T1550.002": {  # Pass the Hash
        "attack_name": "Use Alternate Authentication Material: Pass the Hash",
        "countermeasures": [
            ("D3-CH",   "Credential Hardening",
             "Credential Guard previene extracción de hashes"),
            ("D3-LAM",  "Local Account Monitoring",
             "Restricción de cuentas administrativas locales (LAPS)"),
            ("D3-ANET", "Authentication Event Thresholding",
             "Detectar autenticaciones NTLM anómalas"),
        ]
    },
    "T1003.003": {  # NTDS (DCSync)
        "attack_name": "OS Credential Dumping: NTDS",
        "countermeasures": [
            ("D3-PA",   "Process Analysis",
             "Detectar accesos al proceso ntdsutil o lsass en el DC"),
            ("D3-ANET", "Authentication Event Thresholding",
             "Detectar solicitudes de replicación AD anómalas (DCSync)"),
            ("D3-UAP",  "User Account Permissions",
             "Limitar permisos de replicación de directorio (DS-Replication-Get-Changes)"),
        ]
    },
}

def show_countermeasures(attack_id):
    """Muestra las contramedidas D3FEND para una técnica ATT&CK."""
    if attack_id not in ATTACK_TO_D3FEND_MAP:
        print(f"[-] No hay mapeo de referencia para {attack_id}")
        print(f"    Consultar directamente: https://d3fend.mitre.org/offensive-technique/attack/{attack_id}/")
        return

    info = ATTACK_TO_D3FEND_MAP[attack_id]
    print(f"\n{'='*65}")
    print(f"  {attack_id} — {info['attack_name']}")
    print(f"{'='*65}")
    print(f"\n  Contramedidas D3FEND recomendadas:\n")

    for d3id, name, desc in info["countermeasures"]:
        print(f"  [{d3id}] {name}")
        print(f"      → {desc}")
        print(f"      → https://d3fend.mitre.org/technique/d3f:{name.replace(' ', '')}/")
        print()


# Ejemplo: revisar contramedidas para las técnicas usadas en un engagement
techniques_used_in_engagement = [
    "T1566.001",  # Phishing inicial
    "T1003.001",  # Mimikatz LSASS dump
    "T1558.003",  # Kerberoasting
    "T1550.002",  # Pass the Hash
    "T1003.003",  # DCSync
]

print("ANÁLISIS DE CONTRAMEDIDAS D3FEND PARA TÉCNICAS DEL ENGAGEMENT")

for tech_id in techniques_used_in_engagement:
    show_countermeasures(tech_id)
```

---

## 4. Aplicación práctica — informe de gap analysis

Una de las aplicaciones más valiosas de D3FEND es generar análisis de brechas defensivas a partir de los hallazgos de un Red Team. En lugar de recomendaciones genéricas, el informe especifica exactamente qué controles D3FEND faltan:

```python
#!/usr/bin/env python3
"""
gap_analysis.py — Genera un informe de gap analysis usando D3FEND
A partir de las técnicas ATT&CK usadas y los controles existentes del cliente
"""

def generate_gap_report(techniques_used, existing_controls):
    """
    techniques_used: lista de técnicas ATT&CK que tuvieron éxito en el engagement
    existing_controls: lista de IDs D3FEND que el cliente ya tiene implementados
    """

    print("=" * 65)
    print("  INFORME DE GAP ANALYSIS — D3FEND")
    print("=" * 65)

    total_gaps = 0
    total_covered = 0

    for tech_id in techniques_used:
        if tech_id not in ATTACK_TO_D3FEND_MAP:
            continue

        info = ATTACK_TO_D3FEND_MAP[tech_id]
        print(f"\n[TÉCNICA EXITOSA] {tech_id} — {info['attack_name']}")

        for d3id, name, desc in info["countermeasures"]:
            if d3id in existing_controls:
                print(f"  [✓ IMPLEMENTADO PERO INEFECTIVO] {d3id} {name}")
                print(f"    → La técnica tuvo éxito a pesar de este control.")
                print(f"    → Revisar configuración: {desc}")
                total_covered += 1
            else:
                print(f"  [✗ GAP] {d3id} {name}")
                print(f"    → Control ausente. {desc}")
                total_gaps += 1

    print(f"\n{'='*65}")
    print(f"  RESUMEN")
    print(f"{'='*65}")
    print(f"  Controles ausentes (gaps):          {total_gaps}")
    print(f"  Controles presentes pero ineficaces: {total_covered}")
    print(f"\n  Priorización recomendada:")
    print(f"  1. Implementar controles marcados como GAP")
    print(f"  2. Auditar configuración de controles marcados como")
    print(f"     'implementado pero inefectivo' — pueden estar mal")
    print(f"     configurados, sin alertas activas, o con excepciones")
    print(f"     que el atacante explotó")


# Ejemplo de uso
existing_controls = ["D3-PA", "D3-NTA"]  # El cliente dice tener esto

techniques_used = [
    "T1003.001",  # LSASS dump — tuvo éxito a pesar de D3-PA
    "T1558.003",  # Kerberoasting — sin contramedida
    "T1550.002",  # Pass the Hash — sin contramedida
]

generate_gap_report(techniques_used, existing_controls)
```

---

## 5. D3FEND Navigator y recursos visuales

D3FEND tiene su propia herramienta de navegación, similar al ATT&CK Navigator, que permite explorar el grafo de conocimiento visualmente:

```
D3FEND Matrix: https://d3fend.mitre.org/matrix/
→ Vista de matriz similar a ATT&CK, organizada por las 7 tácticas

D3FEND Technique pages:
https://d3fend.mitre.org/technique/d3f:CredentialHardening/
→ Cada técnica tiene su propia página con:
   - Definición formal
   - Técnicas ATT&CK relacionadas (ofensivas que contrarresta)
   - Productos/tecnologías que implementan esta técnica
   - Artifacts digitales involucrados

D3FEND Digital Artifact Ontology:
https://d3fend.mitre.org/dao/
→ El modelo de datos completo — útil para entender las relaciones
   entre técnicas ofensivas y defensivas a nivel de objetos del sistema
```

---

## 6. D3FEND en el ciclo de un engagement

### Para Red Team

```
DURANTE EL ENGAGEMENT:
→ Documentar cada técnica ATT&CK exitosa
→ Anotar si el control D3FEND esperado existía pero no funcionó,
  o si simplemente no existía

EN EL INFORME:
→ Para cada hallazgo, mapear a D3FEND y especificar:
  - Qué contramedida específica falta o falló
  - Por qué esa contramedida habría detenido/detectado el ataque
  - Cómo verificar que la contramedida está correctamente implementada
```

### Para Blue Team

```
EVALUACIÓN DE MADUREZ:
→ Mapear los controles de seguridad existentes (SIEM rules, EDR
  policies, hardening guides) a técnicas D3FEND
→ Cruzar con ATT&CK para ver qué técnicas ofensivas quedan sin cobertura
→ El "hueco" entre lo que D3FEND dice que deberías tener
  y lo que realmente tienes es el roadmap de mejora

PRIORIZACIÓN DE INVERSIÓN:
→ D3FEND ayuda a justificar inversiones en seguridad con lenguaje
  técnico preciso: "necesitamos D3-CH (Credential Hardening) porque
  contrarresta T1003 y T1558, que son las técnicas #1 y #3 más
  usadas según nuestros últimos 5 pentests"
```

---

## 7. Cheatsheet de referencia rápida

```
# ── LAS 7 TÁCTICAS D3FEND ─────────────────────────────────────────────────
Model    → Modelado de línea base del sistema
Harden   → Reducir superficie de ataque (proactivo)
Detect   → Identificar actividad maliciosa (reactivo)
Isolate  → Contener el alcance del ataque
Deceive  → Engañar al atacante (honeypots, decoys)
Evict    → Eliminar presencia del atacante
Restore  → Recuperar al estado pre-compromiso

# ── TÉCNICAS D3FEND MÁS RELEVANTES ────────────────────────────────────────
D3-CH    Credential Hardening        → contra T1003, T1110, T1550
D3-PA    Process Analysis            → contra T1055, T1059
D3-NTA   Network Traffic Analysis    → contra T1071, T1021
D3-ANET  Auth Event Thresholding     → contra T1110.003, T1558
D3-MH    Message Hardening           → contra T1566 (phishing)
D3-NI    Network Isolation           → contra T1021 (lateral movement)
D3-FCA   File Content Analysis       → contra T1027, malware
D3-UBA   User Behavior Analysis      → contra T1078 (valid accounts)
D3-DO    Decoy Object                → detecta T1552 (creds en ficheros)

# ── MAPEO RÁPIDO ATT&CK → D3FEND (técnicas frecuentes) ───────────────────
T1003.001 (LSASS)         → D3-CH, D3-PA, D3-SCA
T1566.001 (Phishing)       → D3-FCA, D3-MH, D3-UA
T1558.003 (Kerberoasting)  → D3-CH, D3-ANET, D3-NTA
T1055.001 (DLL Injection)  → D3-PA, D3-EI, D3-SCF
T1021.002 (SMB Admin)      → D3-NI, D3-ANET, D3-NTA
T1550.002 (Pass the Hash)  → D3-CH, D3-LAM, D3-ANET
T1003.003 (DCSync)         → D3-PA, D3-ANET, D3-UAP

# ── RECURSOS WEB ──────────────────────────────────────────────────────────
https://d3fend.mitre.org                  → D3FEND principal
https://d3fend.mitre.org/matrix/          → Matriz de tácticas/técnicas
https://d3fend.mitre.org/dao/             → Ontología de digital artifacts
https://d3fend.mitre.org/api/             → API GraphQL/SPARQL
https://attack.mitre.org                  → ATT&CK (complementario)
https://mitre-attack.github.io/attack-navigator/ → ATT&CK Navigator
```
