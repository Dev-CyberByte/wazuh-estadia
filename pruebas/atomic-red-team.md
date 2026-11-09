# Pruebas con Atomic Red Team

> ⚠️ Ejecutar **solo** en equipos de laboratorio autorizados.

## Objetivo
Validar que Wazuh (con Sysmon / auditd) detecta técnicas de adversarios y las asocia a MITRE ATT&CK.

## Preparación
- Endpoint Windows de laboratorio con agente Wazuh + Sysmon
- [Invoke-AtomicRedTeam](https://github.com/redcanaryco/invoke-atomicredteam) instalado
- Excepciones de antivirus **solo** en la carpeta de pruebas del laboratorio

## Metodología
1. Elegir una técnica (ID MITRE).
2. Ver qué hace: `Invoke-AtomicTest T#### -ShowDetails`
3. Ejecutarla: `Invoke-AtomicTest T#### -TestNumbers N`
4. Buscar la alerta en Wazuh (Threat Hunting / MITRE ATT&CK).
5. Registrar resultado y limpiar: `Invoke-AtomicTest T#### -Cleanup`

## Resultados

| # | Técnica MITRE | ID | Prueba ejecutada | ¿Detectada? | Regla / Alerta Wazuh | Evidencia |
|---|---|---|---|---|---|---|
| 1 | _(ej. PowerShell)_ | T1059.001 | _TODO_ | ✅ / ❌ | _TODO_ | `docs/img/...` |
| 2 | _(ej. Creación de cuenta local)_ | T1136.001 | _TODO_ | ✅ / ❌ | _TODO_ | `docs/img/...` |
| 3 | _(ej. Credential dumping)_ | T1003 | _TODO_ | ✅ / ❌ | _TODO_ | `docs/img/...` |

## Hallazgos y ajustes
- _TODO: qué no se detectó y qué se ajustó (reglas, Sysmon config, etc.)_

## Conclusiones
- _TODO_
