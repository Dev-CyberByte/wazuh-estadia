# 🛠️ Guía de instalación y configuración de Wazuh

Guía paso a paso basada en la documentación oficial de Wazuh y en lo realizado en el laboratorio (Oracle Linux 8.10, Wazuh 4.9.2, despliegue todo en uno).

---

## Índice
1. [Requisitos previos](#1-requisitos-previos)
2. [Instalación del servidor (todo en uno)](#2-instalación-del-servidor-todo-en-uno)
3. [Instalación por componentes (opcional)](#3-instalación-por-componentes-opcional)
4. [Despliegue de agentes](#4-despliegue-de-agentes)
5. [Sysmon en Windows](#5-sysmon-en-windows)
6. [auditd en Linux](#6-auditd-en-linux)
7. [Grupos y agent.conf](#7-grupos-y-agentconf)
8. [Configuración del servidor (ossec.conf)](#8-configuración-del-servidor-ossecconf)
9. [Verificación](#9-verificación)
10. [Problemas comunes](#10-problemas-comunes)

---

## 1. Requisitos previos

| Requisito | Detalle |
|---|---|
| SO del servidor | Linux 64 bits. Laboratorio: Oracle Linux 8.10 |
| Hardware | Depende del número de endpoints; consultar la tabla de dimensionamiento del [Quickstart](https://documentation.wazuh.com/current/quickstart.html) |
| Permisos | Usuario con `sudo` |
| Herramientas | `curl` |
| Firewall | Permitir 1514/tcp, 1515/tcp y 443/tcp |

### Puertos de Wazuh

| Componente | Puerto | Protocolo | Propósito |
|---|---|---|---|
| Servidor | 1514 | TCP | Conexión de agentes |
| Servidor | 1515 | TCP | Registro (enrolamiento) de agentes |
| Servidor | 1516 | TCP | Clúster |
| Servidor | 514 | UDP/TCP | Syslog (deshabilitado por defecto) |
| Servidor | 55000 | TCP | API RESTful |
| Indexador | 9200 | TCP | API del indexador |
| Indexador | 9300–9400 | TCP | Clúster del indexador |
| Dashboard | 443 | TCP | Interfaz web |

---

## 2. Instalación del servidor (todo en uno)

**Paso 1. Abrir puertos (familia RHEL / Oracle Linux)**
```bash
sudo firewall-cmd --permanent --add-port=1514/tcp
sudo firewall-cmd --permanent --add-port=1515/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --reload
```

**Paso 2. Descargar y ejecutar el asistente**
```bash
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```
> ⚠️ En `-sO` la última letra es una **O mayúscula**. Con un cero (`-s0`) el script se imprime en pantalla en lugar de guardarse.

**Paso 3. Seguir la instalación.** El asistente genera certificados e instala, en orden: indexador → servidor (con Filebeat) → dashboard. Termina con el mensaje `Installation finished`.

**Paso 4. Guardar credenciales.** El asistente muestra el usuario `admin` y su contraseña. Todas quedan en `wazuh-install-files.tar`:
```bash
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

**Paso 5. Verificar servicios**
```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
sudo systemctl status filebeat
```

**Paso 6. Entrar al panel:** `https://<IP-del-servidor>` (puerto 443). El navegador advertirá del certificado autofirmado.

---

## 3. Instalación por componentes (opcional)

Útil para nodo único o multinodo (cada componente en un servidor distinto).

```bash
sudo curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
sudo bash ./wazuh-install.sh --generate-config-files
```
Antes de ejecutar el segundo comando se definen los nodos y direcciones en `config.yml`. Después se instala cada componente en su servidor copiando los archivos generados (ver la documentación oficial de instalación paso a paso).

---

## 4. Despliegue de agentes

### 4.1 Desde el panel
1. **Server management → Endpoints Summary → Deploy new agent**.
2. Elegir el paquete (ej. *Linux RPM amd64* para Oracle Linux; *Windows MSI* para Windows).
3. Escribir la dirección del servidor Wazuh.
4. (Opcional) Nombre único del agente y grupo. **El nombre no se puede cambiar** después de enrolar.
5. Copiar y ejecutar el comando generado en el endpoint, con privilegios de administrador.
6. Iniciar el agente (Linux):
```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```
7. Volver a *Endpoints Summary* y confirmar que el agente aparece **Active**.

### 4.2 Verificar en el endpoint
```bash
sudo systemctl status wazuh-agent
sudo tail -f /var/ossec/logs/ossec.log      # Linux
```
En Windows: servicio **Wazuh** en `services.msc` y log en `C:\Program Files (x86)\ossec-agent\ossec.log`.

---

## 5. Sysmon en Windows

1. Descargar [Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) y la configuración de [SwiftOnSecurity](https://github.com/SwiftOnSecurity/sysmon-config) (`sysmonconfig-export.xml`).
2. En una consola **como administrador**, dentro de la carpeta con ambos:
```cmd
sysmon.exe -accepteula -i sysmonconfig-export.xml
```
3. Verificar: debe indicar que el controlador `SysmonDrv` y el servicio `Sysmon` se instalaron e iniciaron. Los eventos quedan en **Visor de eventos → Microsoft → Windows → Sysmon → Operational**.
4. Hacer que el agente Wazuh lo lea → ver [sección 7](#7-grupos-y-agentconf).

---

## 6. auditd en Linux

```bash
sudo apt install auditd audispd-plugins       # Debian/Ubuntu
sudo systemctl enable auditd
sudo systemctl start auditd
```
En RHEL/Oracle Linux, `auditd` normalmente ya viene instalado (`sudo dnf install audit` si no).

Parámetros revisados en `/etc/audit/auditd.conf`: log en `/var/log/audit/audit.log`, formato `ENRICHED`, `max_log_file = 8`, `num_logs = 5`, acción `ROTATE`.

Reglas en `/etc/audit/rules.d/audit.rules` → ver [`../configs/audit.rules`](../configs/audit.rules). Para cargarlas:
```bash
sudo augenrules --load
sudo auditctl -l        # lista las reglas activas
```

---

## 7. Grupos y agent.conf

Los grupos permiten aplicar configuración centralizada a varios agentes.

1. **Server management → Groups** → crear/seleccionar el grupo (`Windows`, `Linux`).
2. Asignar los agentes al grupo.
3. Editar `agent.conf` del grupo:
   - Windows → [`../configs/agent-group-windows.conf`](../configs/agent-group-windows.conf)
   - Linux → [`../configs/agent-group-linux.conf`](../configs/agent-group-linux.conf)
4. Guardar. Los agentes recibirán la configuración al sincronizar (la suma de verificación del grupo cambia).

---

## 8. Configuración del servidor (ossec.conf)

**Server management → Settings → Edit configuration** (o directamente `/var/ossec/etc/ossec.conf`).

Elementos revisados:

| Bloque | Valor en laboratorio |
|---|---|
| `<jsonout_output>` / `<alerts_log>` | habilitados |
| `<email_notification>` | `no` |
| `<alerts>` | log nivel 3, correo nivel 12 |
| `<sca>` | habilitado, al iniciar, cada 12 h |
| `<vulnerability-detection>` | `enabled=yes`, `index-status=yes`, feed cada 60 m |
| `<indexer>` | `https://127.0.0.1:9200` con certificados de Filebeat |
| `<cluster>` | deshabilitado (puerto 1516) |

Fragmentos listos → [`../configs/ossec-manager-snippets.xml`](../configs/ossec-manager-snippets.xml)

Después de editar: **Save → Restart Manager**, o por consola:
```bash
sudo systemctl restart wazuh-manager
```
> 💡 Antes de reiniciar, el panel valida la configuración. Haz respaldo: `sudo cp /var/ossec/etc/ossec.conf /var/ossec/etc/ossec.conf.bak`.

---

## 9. Verificación

| Qué | Cómo |
|---|---|
| Servicios del servidor activos | `systemctl status wazuh-manager wazuh-indexer wazuh-dashboard` |
| Agente conectado | Panel → Endpoints Summary → **Active** |
| Sysmon llegando | Panel → Threat Hunting → filtrar `agent.name` y `data.win.system.providerName: Microsoft-Windows-Sysmon` |
| auditd llegando | Threat Hunting → buscar `rule.groups: audit` |
| FIM | Crear/modificar un archivo en un directorio vigilado y revisar el módulo File Integrity Monitoring |
| Vulnerabilidades | Panel → Vulnerability Detection (puede tardar tras el primer escaneo) |
| SCA | Panel → Configuration Assessment por agente |

---

## 10. Problemas comunes

| Síntoma | Causa probable | Solución |
|---|---|---|
| El script `wazuh-install.sh` se imprime en pantalla | Se escribió `-s0` (cero) | Usar `-sO` (letra O) |
| Advertencia de SO no recomendado | Oracle Linux no está en la lista oficial | Continuar en laboratorio; validar antes de producción |
| Advertencia de firewall | `firewalld` activo | Abrir 1514, 1515 y 443 |
| Agente en *Disconnected* / *Never connected* | Puerto cerrado o IP del manager incorrecta | Revisar firewall, `WAZUH_MANAGER` y `ossec.log` del agente |
| No llegan eventos de Sysmon | `localfile` mal escrito o agente fuera del grupo | Revisar `agent.conf`, grupo y que el canal sea `Microsoft-Windows-Sysmon/Operational` |
| No llegan eventos de auditd | Reglas no cargadas o ruta incorrecta | `sudo auditctl -l`, revisar `/var/log/audit/audit.log` |
| Contraseña perdida | No se guardó | Recuperarla de `wazuh-install-files.tar` |
