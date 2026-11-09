# Integración con VirusTotal

## Objetivo
Cuando FIM detecta un archivo nuevo o modificado, Wazuh consulta su hash en VirusTotal para conocer su reputación.

## Configuración
1. Crear cuenta en VirusTotal y obtener la API key.
2. Agregar el bloque `<integration>` en el `ossec.conf` del Manager (ver [`../configs/ossec-manager-snippets.xml`](../configs/ossec-manager-snippets.xml)).
3. Verificar que los directorios a vigilar estén en `<syscheck>` (ver [`../configs/agent-group-linux.conf`](../configs/agent-group-linux.conf)).
4. Reiniciar el Manager: `sudo systemctl restart wazuh-manager`.

> 🔐 La API key es personal. **No la subas al repositorio.** La cuenta gratuita tiene límite de consultas.

## Pruebas
| Prueba | Archivo | Resultado en VirusTotal | Alerta en Wazuh | Evidencia |
|---|---|---|---|---|
| _TODO_ | _(ej. archivo de prueba EICAR)_ | _TODO_ | _TODO_ | `docs/img/...` |

## Conclusiones
- _TODO_
