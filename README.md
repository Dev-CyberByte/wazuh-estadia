# Wazuh SIEM/XDR — Proyecto de Estadía

> **Integración y configuración de la plataforma Wazuh en la infraestructura tecnológica de una empresa de gestión documental (ADEA MX).**
> Repositorio con la configuración, las pruebas y la documentación técnica desarrolladas durante mi **estadía profesional**.

![Wazuh](https://img.shields.io/badge/Wazuh-4.9.2-blue)
![OS](https://img.shields.io/badge/Server-Oracle%20Linux%208.10-red)
![Sysmon](https://img.shields.io/badge/Sysmon-v15.15-orange)
![Status](https://img.shields.io/badge/estado-en%20desarrollo-yellow)

---

## Contexto académico

| | |
|---|---|
| **Institución** | Universidad Politécnica del Valle de Toluca |
| **Programa** | Ingeniería en Tecnologías de la Información |
| **Asignatura** | Estadía |
| **Autor** | Juan Pablo Olivares Pérez |
| **Periodo** | Septiembre 2026 – en curso |

Este repositorio forma parte del proyecto de estadía y documenta, de forma reproducible, lo que se fue construyendo en un **entorno de laboratorio** antes de llevarlo (de forma controlada y autorizada) a la infraestructura real.

---

## Objetivo

Integrar y configurar Wazuh para fortalecer el **monitoreo, la detección y la gestión de eventos de seguridad**: centralizar logs de endpoints, generar alertas, detectar vulnerabilidades y validar las detecciones con pruebas controladas de adversarios (Blue Team / Purple Team).

---

## Arquitectura del laboratorio

```
┌────────────────────┐        1514/TCP (cifrado AES)        ┌──────────────────────────────┐
│  Endpoints         │ ───────────────────────────────────▶ │  Servidor Wazuh (todo en uno)│
│  • Windows + Sysmon│        1515/TCP (enrolamiento)       │  • Manager  (analysisd, ...) │
│  • Linux + auditd  │                                      │  • Indexer  (9200/TCP)       │
│  • Wazuh Agent     │                                      │  • Dashboard (443/TCP)       │
└────────────────────┘                                      └──────────────┬───────────────┘
                                                                           │
                                          Inteligencia de amenazas  ◀──────┤  VirusTotal API
                                          Vulnerabilidades (CVE)    ◀──────┘  Wazuh CTI
```

| Componente | Detalle |
|---|---|
| Servidor | Oracle Linux 8.10 · despliegue **todo en uno** (indexer + manager + dashboard) |
| Versión Wazuh | 4.9.2 |
| Agentes | Windows (con Sysmon) y Linux (con auditd) |
| Grupos de agentes | `Windows`, `Linux` |

**Puertos utilizados**

| Componente | Puerto | Uso |
|---|---|---|
| Manager | 1514/TCP | Conexión de agentes |
| Manager | 1515/TCP | Registro (enrolamiento) |
| Manager | 55000/TCP | API RESTful |
| Indexer | 9200/TCP | API del indexador |
| Dashboard | 443/TCP | Interfaz web |

---

## Estructura del repositorio

```
.
├── README.md
├── .gitignore
├── configs/
│   ├── agent-group-windows.conf     # agent.conf del grupo Windows (Sysmon)
│   ├── agent-group-linux.conf       # agent.conf del grupo Linux (FIM + auditd)
│   ├── audit.rules                  # reglas de auditd
│   ├── ossec-manager-snippets.xml   # fragmentos relevantes del ossec.conf del Manager
│   └── sysmon/                      # (opcional) config de Sysmon usada
├── pruebas/
│   ├── atomic-red-team.md           # pruebas de detección + mapeo MITRE ATT&CK
│   └── virustotal.md                # integración y pruebas con VirusTotal
└── docs/
    ├── instalacion.md               # (opcional) guía detallada
    └── img/                         # capturas (sin datos sensibles)
```

---

## Qué se ha implementado

- [x] Instalación asistida de Wazuh (todo en uno) en Oracle Linux 8.10
- [x] Despliegue de agentes (Linux RPM / Windows)
- [x] **Sysmon** en endpoints Windows con configuración de SwiftOnSecurity
- [x] **auditd** en endpoints Linux (monitoreo de ejecución de comandos y cambios de privilegios)
- [x] **FIM** (File Integrity Monitoring) en tiempo real
- [x] **Detección de vulnerabilidades** (inventario de software ↔ CVE)
- [x] **SCA** (Security Configuration Assessment, benchmarks CIS)
- [x] **MITRE ATT&CK**: mapeo de alertas a tácticas y técnicas
- [x] **Pruebas con Atomic Red Team** para validar detecciones
- [x] **Inteligencia de amenazas** con VirusTotal
- [x] **Threat Hunting** sobre las alertas generadas
- [ ] Respuesta activa (Active Response) — pendiente
- [ ] Integración de endpoints autorizados de producción — pendiente

---

## 1. Instalación del servidor

```bash
# Abrir puertos (familia RHEL / Oracle Linux)
sudo firewall-cmd --permanent --add-port=1514/tcp
sudo firewall-cmd --permanent --add-port=1515/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --reload

# Descargar y ejecutar el asistente (todo en uno)
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh   # ojo: "O" mayúscula, no cero
sudo bash ./wazuh-install.sh -a

# Verificar servicios
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard filebeat
```

> ⚠️ El asistente muestra una advertencia en Oracle Linux 8.10 (no figura en su lista de SO recomendados, aunque RHEL 8 sí). La instalación concluyó y los servicios iniciaron correctamente; debe validarse antes de pasar a producción.

> 🔐 Las credenciales las genera el instalador (`wazuh-install-files.tar`). **Nunca** las subas al repositorio.

## 2. Despliegue de agentes

Desde el panel: **Server management → Endpoints Summary → Deploy new agent**. Ejemplo para Linux RPM (el comando real lo genera el panel):

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

Los agentes se organizan por **grupos** (`Windows`, `Linux`) para aplicar configuración centralizada mediante `agent.conf`.

---

## 3. Recolección de eventos

### 3.1 Windows + Sysmon

1. Instalar Sysmon con la configuración de [SwiftOnSecurity/sysmon-config](https://github.com/SwiftOnSecurity/sysmon-config) (como administrador):
   ```cmd
   sysmon.exe -accepteula -i sysmonconfig-export.xml
   ```
2. Hacer que el agente lea el canal de Sysmon → [`configs/agent-group-windows.conf`](configs/agent-group-windows.conf)

### 3.2 Linux + auditd

1. Instalar y habilitar auditd:
   ```bash
   sudo apt install auditd audispd-plugins
   sudo systemctl enable --now auditd
   ```
2. Cargar las reglas → [`configs/audit.rules`](configs/audit.rules)
3. Hacer que el agente lea `audit.log` y vigile directorios en tiempo real → [`configs/agent-group-linux.conf`](configs/agent-group-linux.conf)

---

## 4. Pruebas de detección — Atomic Red Team

Para validar que las detecciones funcionan se ejecutaron técnicas de [Atomic Red Team](https://github.com/redcanaryco/atomic-red-team) **únicamente en equipos de laboratorio autorizados**, y se verificó qué alertas generó Wazuh y a qué técnica MITRE ATT&CK se asociaron.

Detalle y resultados → [`pruebas/atomic-red-team.md`](pruebas/atomic-red-team.md)

## 5. Inteligencia de amenazas — VirusTotal

Integración de Wazuh con la API de VirusTotal para consultar la reputación de archivos detectados por FIM.

Detalle → [`pruebas/virustotal.md`](pruebas/virustotal.md)

## 6. MITRE ATT&CK y Threat Hunting

Las reglas predeterminadas de Wazuh ya están mapeadas a identificadores MITRE. Desde el dashboard se usa:

- **MITRE ATT&CK** → visualizar tácticas y técnicas observadas
- **Threat Hunting** → explorar y filtrar alertas para buscar actividad sospechosa

## 7. Detección de vulnerabilidades

El agente envía su inventario de software al servidor, que lo correlaciona con las fuentes de vulnerabilidades (Wazuh CTI) y genera alertas cuando aparece o se resuelve una CVE.

Configuración en el `ossec.conf` del Manager → [`configs/ossec-manager-snippets.xml`](configs/ossec-manager-snippets.xml)

## 8. SCA y FIM

- **SCA:** evaluación periódica de configuraciones seguras con políticas basadas en CIS.
- **FIM:** `syscheck` compara sumas de verificación y atributos de archivos monitoreados; con `realtime="yes"` detecta cambios casi al instante.

---

## 📈 Cronograma y avance

| Etapa | Estado |
|---|---|
| Análisis del proyecto y requerimientos | ✅ |
| Investigación de Wazuh y antecedentes | ✅ |
| Análisis de infraestructura y endpoints | ✅ |
| Configuración inicial de Wazuh | ✅ |
| Integración y registro de endpoints | ✅ |
| Monitoreo y recolección de eventos | ✅ |
| Detección y vulnerabilidades | ✅ |
| Pruebas funcionales y análisis de resultados | 🔄 En curso |
| Ajustes y optimización | ⏳ |
| Documentación técnica y entrega | ⏳ |

---

## ⚠️ Aviso de confidencialidad y uso ético

- Este repositorio **no contiene** direcciones IP, nombres de equipos, usuarios, contraseñas, API keys ni información interna de la organización. Todo se sanitiza antes de publicarse.
- Las pruebas de Atomic Red Team se realizan **solo en equipos autorizados** y en laboratorio. No ejecutes técnicas ofensivas en sistemas que no te pertenezcan o sin permiso explícito.
- El contenido es con fines **educativos y de documentación académica**.

## 📚 Referencias

- [Documentación oficial de Wazuh](https://documentation.wazuh.com/current/index.html)
- [Wazuh Quickstart](https://documentation.wazuh.com/current/quickstart.html)
- [Sysmon — Microsoft Learn](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
- [SwiftOnSecurity/sysmon-config](https://github.com/SwiftOnSecurity/sysmon-config)
- [Atomic Red Team](https://github.com/redcanaryco/atomic-red-team)
- [MITRE ATT&CK](https://attack.mitre.org/)
- [Wazuh — Monitoring execution of malicious commands (auditd)](https://documentation.wazuh.com/current/learning-wazuh/audit-commands.html)

---

<sub>Proyecto de estadía · UPVT · ITI · 2026</sub>
