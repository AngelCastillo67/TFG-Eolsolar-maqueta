# Casos de prueba ejecutados

| Caso  | Estado | Fecha    | Resultado |
|-------|--------|----------|-----------|
| TC-01 | ✓ | DD/MM/AA | Bloqueo Z2 → Z6 y registro confirmado en pfSense/Wazuh |
| TC-02 | ✓ | DD/MM/AA | Alerta FIM detectada sobre `/etc/hosts` en Xs |
| TC-03 | ✓ | DD/MM/AA | 3 notificaciones tabletop NIS2 redactadas en plazo |

## Resumen

Durante el Sprint 5 se ejecutaron los tres casos de prueba definidos para validar la maqueta técnica y organizativa del proyecto.

- **TC-01** validó la segmentación entre la zona IT/Z2 y la zona OT/Z6 mediante pfSense, así como la recepción y correlación de eventos en Wazuh.
- **TC-02** validó la capacidad de Wazuh FIM para detectar modificaciones en un archivo crítico del sistema, concretamente `/etc/hosts` en `srv-it`.
- **TC-03** validó el procedimiento organizativo de respuesta a incidentes mediante un ejercicio tabletop alineado con los plazos de notificación de NIS2.

## Evidencias asociadas

Las evidencias de ejecución se encuentran en la carpeta:

```text
06-evidencias/
```

Archivos principales esperados:

```text
TC-01-terminal-srv-it.png
TC-01-pfsense-firewall-log.png
TC-01-wazuh-alerta-100201.png

TC-02-terminal-modificacion-hosts.png
TC-02-wazuh-integrity-monitoring.png
TC-02-wazuh-security-events.png

TC-03-notif-1-ransomware.md
TC-03-notif-2-phishing.md
TC-03-notif-3-proveedor.md
TC-03-acta.md
TC-03-mejoras.md
TC-03-tabla-tiempos.md
```

## Comprobaciones finales

Antes de cerrar el sprint, verificar:

- [ ] `TC-01.md` contiene procedimiento, resultado esperado, resultado obtenido y evidencias.
- [ ] `TC-02.md` contiene procedimiento, resultado esperado, resultado obtenido y evidencias.
- [ ] `TC-03.md` contiene procedimiento, resultado esperado, resultado obtenido y evidencias.
- [ ] Todas las capturas referenciadas existen en `06-evidencias/`.
- [ ] La Tabla 5.3 de la memoria coincide con los resultados reales obtenidos.
- [ ] Se han documentado las desviaciones, si las hubo.
