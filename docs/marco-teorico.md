# 📚 Marco teórico

Resumen del marco teórico del proyecto de estadía. Las citas siguen formato APA; las referencias completas están al final.

## Índice
1. [Linux y Oracle Linux](#1-linux-y-oracle-linux)
2. [Conceptos de seguridad](#2-conceptos-de-seguridad)
3. [Wazuh](#3-wazuh)
4. [Módulos y capacidades](#4-módulos-y-capacidades)
5. [Sysmon](#5-sysmon)
6. [Linux Audit y auditd](#6-linux-audit-y-auditd)
7. [Referencias](#7-referencias)

---

## 1. Linux y Oracle Linux

### Linux
Sistema operativo de código abierto creado por Linus Torvalds en 1991 (Red Hat, s. f.). En sentido estricto, Linux es el **kernel**, el componente clave del sistema operativo (Oracle, s. f.-c). Sobre él se construyen las **distribuciones**: recopilaciones de componentes que forman un sistema completo. Se agrupan en **familias** según su sistema de paquetes y manejo de actualizaciones, por ejemplo la familia **RHEL** (paquetes RPM) y la familia **Debian** (paquetes DEB) (Oracle, s. f.-c). Esto determina el instalador del agente de Wazuh.

### Oracle Linux
Distribución empresarial compatible a nivel binario con RHEL y CentOS (Oracle, s. f.-b), por lo que usa los mismos comandos (`rpm`, `systemctl`, `firewall-cmd`). Ofrece dos kernels: **UEK** (Unbreakable Enterprise Kernel, por defecto) y **RHCK** (compatible con RHEL) (Oracle, s. f.-a). Incluye **Ksplice** para actualizar el kernel sin reiniciar (Oracle, s. f.-a).

En el proyecto se usó **Oracle Linux 8.10** para el servidor.

---

## 2. Conceptos de seguridad

| Concepto | Definición |
|---|---|
| **SOC** | Equipo que monitorea la infraestructura 24/7 para detectar, analizar y responder a incidentes (IBM, 2025d). Busca reducir MTTD y MTTR (Marín de la Fuente, 2026). |
| **SIEM** | Solución que recopila y correlaciona eventos de usuarios, endpoints, aplicaciones, nube y dispositivos de seguridad para identificar amenazas (IBM, 2025c). Nació de unir SIM y SEM. |
| **XDR** | Amplía el SIEM correlacionando endpoints, red y nube, con respuesta automatizada (TecnetOne, 2026). |
| **Blue Team** | Equipo de defensa; se contrapone al Red Team, que simula ataques (S2GRUPO, 2024; Hixec, s. f.). Usa SIEM, EDR y SOAR (Jiratek, 2025). |
| **Monitoreo de seguridad** | Vigilancia continua de logs, tráfico y comportamiento de usuarios (Marín de la Fuente, 2026). |
| **Evento (log)** | Suceso significativo reconocido por un sistema, registrado cronológicamente (CrowdStrike, 2026). Según ISO 27002, indica que una política pudo ser violada o una protección fallar (Norma ISO 27001, s. f.). |
| **Incidente** | Uno o varios eventos no deseados con probabilidad significativa de comprometer las operaciones (Norma ISO 27001, s. f.; IBM, 2025b). |
| **Endpoint** | Dispositivo conectado a la red que puede ser punto de entrada de amenazas (Kaspersky, 2025). |
| **Vulnerabilidad** | Debilidad en un activo que puede ser explotada (IBM, 2025a). Se distingue de la **amenaza**, que es la acción que la aprovecha (INCIBE, 2017). Su gestión es un ciclo continuo: descubrir, priorizar, resolver, reevaluar y reportar (IBM, 2025a). |

### Niveles de analistas SOC
| Nivel | Rol |
|---|---|
| **Tier 1** | Primera línea: clasifica alertas y decide si son amenaza real, falso positivo o requieren escalamiento, apoyándose en playbooks (Cloudskope, s. f.). |
| **Tier 2** | Recibe incidentes escalados, investiga a fondo y coordina la respuesta (nFlo, 2026). |
| **Tier 3** | Máxima experiencia: threat hunting, forense avanzado y desarrollo de contenido de detección (Riskitera, 2026). |

---

## 3. Wazuh

Plataforma gratuita y de código abierto que unifica **XDR y SIEM** para proteger endpoints y cargas de trabajo en la nube (Wazuh, s. f.-b). Se compone de un **agente** universal y tres componentes centrales: **servidor**, **indexador** y **dashboard**.

### Origen
Nació como un *fork* de **OSSEC** en 2015, que llevaba mucho tiempo en modo de mantenimiento. Los agentes de OSSEC son compatibles con el servidor Wazuh (Wazuh, s. f.-g).

### Arquitectura

```
Agente ──(1514/TCP, AES)──▶ Servidor ──(Filebeat, TLS, 9200)──▶ Indexador ◀── Dashboard (443)
                              │                                        ▲
                              └──────── API 55000 ─────────────────────┘
```

| Componente | Función |
|---|---|
| **Agente** | Se instala en laptops, servidores, VMs o nube. Recolecta logs, hace FIM y envía eventos al servidor de forma cifrada (Wazuh, s. f.-a, -b). |
| **Servidor** | Analiza los datos con decodificadores y reglas, usa inteligencia de amenazas para buscar indicadores de compromiso y escala como clúster (Wazuh, s. f.-b). Procesos: `wazuh-remoted`, `wazuh-analysisd`, `wazuh-syscheckd`, `wazuh-logcollector`, `wazuh-authd`, `wazuh-modulesd`, `wazuh-db`, `wazuh-execd`, `wazuh-monitord` y la API. |
| **Indexador** | Motor de búsqueda y análisis de texto completo que almacena las alertas (Wazuh, s. f.-b). Recibe datos vía Filebeat con TLS (Wazuh, s. f.-a). |
| **Dashboard** | Interfaz web para visualización, administración de configuración y monitoreo de estado (Wazuh, s. f.-b). |

También admite monitoreo **sin agente** (agentless) para firewalls, switches y routers mediante Syslog y SSH (Wazuh, s. f.-a).

### Modalidades de despliegue
| Modalidad | Uso |
|---|---|
| **Todo en uno** | Los tres componentes en un servidor. Laboratorios y entornos pequeños. *(usada en este proyecto)* |
| **Nodo único** | Cada componente en un servidor distinto. Entornos medianos. |
| **Multinodo** | Clústeres de servidor e indexador. Entornos grandes / alta disponibilidad. |

---

## 4. Módulos y capacidades

| Módulo | Función |
|---|---|
| **Log data analysis** | Los agentes recolectan registros; el servidor los decodifica y compara contra reglas para generar alertas (Wazuh, s. f.-b). |
| **FIM** | Compara sumas de verificación y atributos de archivos; `syscheck` escanea cada 12 h por defecto y admite tiempo casi real (Wazuh, s. f.-d). |
| **SCA** | Verifica configuraciones seguras con políticas YAML basadas en benchmarks CIS (Wazuh, s. f.-c). |
| **Vulnerability Detection** | Correlaciona el inventario de software con la información de Wazuh CTI (CVE, severidad, mitigación) y alerta cuando aparece o se resuelve una vulnerabilidad (Wazuh, s. f.-f). |
| **MITRE ATT&CK** | Asigna alertas a tácticas y técnicas de adversarios; las reglas por defecto ya están mapeadas (Wazuh, s. f.-h). |
| **VirusTotal** | Analiza archivos sospechosos mediante su API. |
| **Threat Hunting** | Exploración de alertas para identificar amenazas. |
| **Detección de malware** | Acceso del panel para revisar archivos sospechosos. |

---

## 5. Sysmon

**System Monitor** es un servicio y controlador de Windows que persiste tras reinicios y registra actividad del sistema en el registro de eventos: creación de procesos, conexiones de red, cambios en fechas de creación de archivos, etc. (Microsoft, s. f.). Sus eventos están en el canal `Microsoft-Windows-Sysmon/Operational`.

Como punto de partida existen configuraciones de la comunidad, como la de SwiftOnSecurity.

**Integración con Wazuh:** el agente lee el canal con un bloque `localfile` de formato `eventchannel` (Wazuh, s. f.-j). Wazuh incluye reglas para canales de Windows, entre ellos Sysmon (Wazuh, s. f.-k).

---

## 6. Linux Audit y auditd

**Linux Audit** permite rastrear información relevante para la seguridad; no añade seguridad por sí mismo, pero ayuda a descubrir violaciones a las políticas (Red Hat, s. f.-a). Se compone de utilidades en espacio de usuario y del procesamiento de llamadas al sistema en el kernel. Su demonio es **auditd**.

**Integración con Wazuh:** el agente lee `/var/log/audit/audit.log` con un `localfile` de formato `audit` (Wazuh, s. f.-l). Wazuh usa claves predefinidas: `audit-wazuh-w` (escritura), `-r` (lectura), `-a` (atributo), `-x` (ejecución) y `-c` (comando).

---

## 7. Referencias

- Cloudskope. (s. f.). *Tier 1 vs 2 vs 3 SOC analyst roles*. https://www.cloudskope.com/cybersecurity-101/tier-1-vs-tier-2-vs-tier-3-soc-analyst
- CrowdStrike. (2026, 30 de marzo). *¿Qué es un log de evento? Contenido y uso*. https://www.crowdstrike.com/es-es/cybersecurity-101/next-gen-siem/event-logs/
- Hixec. (s. f.). *Red Team, Blue Team, Purple Team y más: roles en la ciberseguridad ofensiva y defensiva*. https://hixec.com/blog/red-team-blue-team-purple-team-y-mas-roles-en-la-ciberseguridad-ofensiva-y-defensiva/
- IBM. (2025a). *¿Qué es la gestión de vulnerabilidades?* https://www.ibm.com/mx-es/think/topics/vulnerability-management
- IBM. (2025b). *¿Qué es la respuesta a incidentes?* https://www.ibm.com/es-es/think/topics/incident-response
- IBM. (2025c). *¿Qué es SIEM?* https://www.ibm.com/mx-es/think/topics/siem
- IBM. (2025d). *¿Qué es un centro de operaciones de seguridad (SOC)?* https://www.ibm.com/mx-es/think/topics/security-operations-center
- INCIBE. (2017). *Amenaza vs vulnerabilidad, ¿sabes en qué se diferencian?* https://www.incibe.es/empresas/blog/amenaza-vs-vulnerabilidad-sabes-se-diferencian
- Jiratek. (2025). *Blue Team en ciberseguridad: estrategias de defensa y beneficios*. https://jiratek.com/blue-team-ciberseguridad/
- Kaspersky. (2025). *¿Qué es la seguridad de los endpoints y cómo funciona?* https://latam.kaspersky.com/resource-center/definitions/what-is-endpoint-security
- Marín de la Fuente. (2026). *¿Qué es un SOC? Guía completa*. https://www.marindelafuente.com.ar/que-es-un-soc-security-operations-center/
- Microsoft. (s. f.). *Sysmon*. https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon
- nFlo. (2026). *SOC tier 1, 2, 3 — analyst roles, skills and career path*. https://nflo.tech/knowledge-base/soc-tier-1-2-3-analyst-roles-responsibilities/
- Norma ISO 27001. (s. f.). *ISO 27002 A16 Gestión de incidentes de la seguridad de la información*. https://www.normaiso27001.es/a16-gestion-de-incidentes-de-la-seguridad-de-la-informacion/
- Oracle. (s. f.-a). *About Oracle Linux and the Unbreakable Enterprise Kernel*. https://docs.oracle.com/en/database/oracle/oracle-database/26/ladbi/about-oracle-linux-and-the-unbreakable-enterprise-kernel.html
- Oracle. (s. f.-b). *Certifying third-party software on Oracle Linux*. https://www.oracle.com/a/ocom/docs/certifying-third-party-software-on-oracle-linux.pdf
- Oracle. (s. f.-c). *¿Qué es Linux?* https://www.oracle.com/es/linux/what-is-linux/
- Red Hat. (s. f.). *El sistema operativo Linux*. https://www.redhat.com/es/topics/linux
- Red Hat. (s. f.-a). *Auditing the system*. https://access.redhat.com/documentation/en/red_hat_enterprise_linux/8/html/security_hardening/auditing-the-system_security-hardening
- Riskitera. (2026). *Analista SOC: roles N1, N2 y N3 explicados con detalle*. https://blog.riskitera.com/es/posts/2026/04/analista-soc-roles-n1-n2-n3/
- S2GRUPO. (2024). *Blue team en ciberseguridad: definición, funciones y herramientas*. https://s2grupo.es/blue-team-en-ciberseguridad-definicion-funciones-y-herramientas/
- SwiftOnSecurity. (s. f.). *sysmon-config*. https://github.com/SwiftOnSecurity/sysmon-config
- TecnetOne. (2026). *Wazuh: qué es, para qué sirve y cómo funciona en 2026*. https://blog.tecnetone.com/que-es-wazuh
- Wazuh. (s. f.-a). *Architecture*. https://documentation.wazuh.com/current/getting-started/architecture.html
- Wazuh. (s. f.-b). *Components*. https://documentation.wazuh.com/current/user-manual/overview.html
- Wazuh. (s. f.-c). *Configuration assessment*. https://documentation.wazuh.com/4.14/getting-started/use-cases/configuration-assessment.html
- Wazuh. (s. f.-d). *File integrity monitoring*. https://documentation.wazuh.com/current/compliance/pci-dss/file-integrity-monitoring.html
- Wazuh. (s. f.-f). *Vulnerability detection: How it works*. https://documentation.wazuh.com/current/user-manual/capabilities/vulnerability-detection/how-it-works.html
- Wazuh. (s. f.-g). *Migrating from OSSEC*. https://github.com/wazuh/wazuh-documentation/blob/4.2/source/migrating-from-ossec/index.rst
- Wazuh. (s. f.-h). *MITRE ATT&CK framework*. https://documentation.wazuh.com/current/user-manual/ruleset/mitre.html
- Wazuh. (s. f.-i). *Quickstart*. https://documentation.wazuh.com/current/quickstart.html
- Wazuh. (s. f.-j). *localfile*. https://documentation.wazuh.com/current/user-manual/reference/ossec-conf/localfile.html
- Wazuh. (s. f.-k). *Configuring log collection for different operating systems*. https://documentation.wazuh.com/current/user-manual/capabilities/log-data-collection/configuration.html
- Wazuh. (s. f.-l). *Monitoring execution of malicious commands*. https://documentation.wazuh.com/current/learning-wazuh/audit-commands.html
