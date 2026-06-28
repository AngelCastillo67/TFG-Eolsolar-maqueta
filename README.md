# TFG-Eolosolar-maqueta

Repositorio técnico de la maqueta desarrollada para el Trabajo Fin de Grado.  
El proyecto tiene como objetivo documentar, desplegar y validar un laboratorio de ciberseguridad segmentado por zonas, utilizando Proxmox, pfSense, máquinas Debian y Wazuh como plataforma SIEM.

La maqueta reproduce un escenario simplificado de red corporativa e industrial, donde se valida la separación entre zonas, la monitorización de eventos de seguridad y la detección de modificaciones en archivos críticos.

## Objetivo del proyecto

El objetivo principal es construir una maqueta funcional que permita demostrar:

- Segmentación de red entre una zona IT y una zona OT.
- Control de tráfico mediante pfSense.
- Registro de eventos de firewall.
- Envío de logs hacia Wazuh.
- Detección de eventos de seguridad.
- Monitorización de integridad de archivos mediante FIM.
- Documentación de casos de prueba y evidencias para la defensa del TFG.

## Estructura del repositorio

```text
TFG-Eolosolar-maqueta/
│
├── 01-proxmox/
├── 02-pfsense/
├── 03-debian-vms/
├── 04-wazuh/
├── 05-pruebas/
└── 06-evidencias/
```

## 01-proxmox

Contiene la documentación relacionada con la preparación del entorno de virtualización.

En esta carpeta se recoge la información relativa a:

- Instalación y configuración inicial de Proxmox.
- Creación de bridges de red.
- Preparación de la infraestructura base.
- Organización de las máquinas virtuales utilizadas en el laboratorio.

Proxmox actúa como plataforma de virtualización sobre la que se ejecutan el firewall, los servidores Debian y la máquina Wazuh.

## 02-pfsense

Contiene la documentación de configuración del cortafuegos pfSense.

En esta carpeta se documenta:

- Configuración de interfaces.
- Definición de zonas de red.
- Reglas de firewall.
- NAT y acceso a servicios internos.
- Envío de logs por syslog hacia Wazuh.
- Reglas de bloqueo entre zonas.

pfSense es el elemento encargado de aplicar la segmentación de red y controlar el tráfico entre las distintas zonas del laboratorio.

## 03-debian-vms

Contiene la documentación de las máquinas Debian utilizadas como servidores de prueba.

Incluye información sobre:

- `srv-it`, servidor ubicado en la zona IT/Z2.
- `srv-ot`, servidor ubicado en la zona OT/Z6.
- Configuración de red de cada máquina.
- Servicios levantados para las pruebas, como SSH o nginx.
- Comandos de validación de conectividad.

Estas máquinas permiten simular activos de diferentes zonas y ejecutar pruebas de conectividad, bloqueo y monitorización.

## 04-wazuh

Contiene la documentación relativa al despliegue y configuración de Wazuh.

En esta carpeta se documenta:

- Instalación de Wazuh mediante Docker.
- Configuración del manager, indexer y dashboard.
- Alta de agentes Wazuh.
- Recepción de logs desde pfSense.
- Reglas locales de detección.
- Configuración de File Integrity Monitoring.

Wazuh actúa como plataforma SIEM del laboratorio, centralizando eventos y generando alertas de seguridad.

## 05-pruebas

Contiene los casos de prueba ejecutados durante la validación final de la maqueta.

Cada caso se documenta en un fichero independiente:

```text
TC-01.md
TC-02.md
TC-03.md
README.md
```

Los casos principales son:

| Caso | Descripción | Objetivo |
|---|---|---|
| TC-01 | Bloqueo de flujo Z2 → Z6 | Validar la segmentación de red y el registro de eventos |
| TC-02 | Detección FIM en `/etc/hosts` | Validar la detección de cambios en archivos críticos |
| TC-03 | Simulacro documental de respuesta | Validar la capacidad de comunicación y respuesta ante incidentes |

Esta carpeta resume la ejecución de las pruebas, los resultados esperados, los resultados obtenidos y las evidencias asociadas.

## 06-evidencias

Contiene las capturas, salidas de comandos y documentos generados como evidencia durante la ejecución de los casos de prueba.

Puede incluir:

- Capturas de terminal.
- Capturas de reglas y logs de pfSense.
- Capturas de alertas en Wazuh.
- Salidas de comandos.
- Documentos de notificación simulada.
- Actas o evidencias documentales del ejercicio tabletop.

Esta carpeta es fundamental para justificar que las pruebas se ejecutaron realmente y que los resultados obtenidos coinciden con lo documentado.

## Topología lógica resumida

La maqueta se basa en varias zonas de red separadas mediante pfSense:

| Zona | Red | Elementos principales |
|---|---|---|
| Z2 / IT | `10.10.20.0/24` | `srv-it`, `wazuh` |
| Z6 / OT | `10.10.60.0/24` | `srv-ot` |
| WAN | Red externa del laboratorio | Acceso desde equipo anfitrión |

Elementos principales:

```text
Equipo anfitrión
      │
   Proxmox
      │
   pfSense
   ├── Z2_USERS: srv-it + Wazuh
   └── Z6_OT: srv-ot
```

## Casos de prueba principales

### TC-01 — Bloqueo Z2 → Z6

Este caso valida que un equipo de la zona IT no puede comunicarse directamente con un activo de la zona OT.

Se comprueba mediante:

- `ping`
- `nmap`
- `curl`
- logs de pfSense
- alertas de Wazuh

El resultado esperado es que el tráfico sea bloqueado y registrado.

### TC-02 — Detección FIM

Este caso valida que Wazuh detecta una modificación en un archivo crítico del sistema.

La prueba consiste en modificar `/etc/hosts` en `srv-it` y comprobar que Wazuh genera una alerta de File Integrity Monitoring.

### TC-03 — Simulacro documental

Este caso valida la parte organizativa de respuesta ante incidentes.

Se realiza una simulación documental simplificada, redactando comunicaciones internas y externas ante un incidente de seguridad. Su objetivo es demostrar que existe un procedimiento básico de análisis, escalado y comunicación.

## Uso del repositorio

Este repositorio está organizado para poder seguir el proyecto por fases:

1. Revisar primero `01-proxmox/` para entender la infraestructura base.
2. Continuar con `02-pfsense/` para comprender la segmentación.
3. Revisar `03-debian-vms/` para conocer los servidores de prueba.
4. Revisar `04-wazuh/` para entender la monitorización.
5. Consultar `05-pruebas/` para ver la validación final.
6. Consultar `06-evidencias/` para comprobar los resultados obtenidos.

## Estado del proyecto

La fase técnica queda finalizada cuando:

- Las máquinas virtuales están desplegadas.
- pfSense aplica correctamente las reglas de segmentación.
- Wazuh recibe eventos de pfSense.
- Los agentes Wazuh están activos.
- Los tres casos de prueba están documentados.
- Las evidencias están guardadas en `06-evidencias/`.
- La memoria del TFG refleja los mismos resultados que la maqueta.

## Nota final

Este repositorio sirve como soporte técnico de la maqueta del TFG.  
Su finalidad no es funcionar como una guía genérica de instalación, sino documentar de forma ordenada el proceso seguido, las configuraciones aplicadas y las evidencias obtenidas durante la validación del laboratorio.
