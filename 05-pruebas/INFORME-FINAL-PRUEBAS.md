# Informe final de pruebas — Sprint 5

## 1\. Resumen ejecutivo

Durante el Sprint 5 se ejecutaron los tres casos de prueba definidos para validar el laboratorio técnico y el procedimiento organizativo de respuesta ante incidentes.

Los casos ejecutados fueron:

* **TC-01 — Bloqueo de flujo directo Z2 → Z6.**
* **TC-02 — Detección FIM en archivo crítico mediante Wazuh.**
* **TC-03 — Simulacro tabletop NIS2.**

El objetivo global fue demostrar que la maqueta no solo implementa controles técnicos, sino que también permite registrar, detectar y documentar incidentes de seguridad de forma coherente con el diseño del Plan Director.

## 2\. Estado general del laboratorio

Antes de ejecutar los casos de prueba se verificó que las máquinas virtuales principales estaban operativas:

|VM|Función|Estado esperado|
|-|-|-|
|`pfsense`|Cortafuegos y segmentación de red|Activa|
|`srv-it`|Servidor en Z2 / entorno IT|Activa|
|`srv-ot`|Servidor en Z6 / entorno OT|Activa|
|`wazuh`|SIEM / monitorización|Activa|

También se verificó que Wazuh era accesible desde el equipo anfitrión y que los agentes de `srv-it` y `srv-ot` aparecían en estado `Active`.

## 3\. Caso TC-01 — Bloqueo de flujo directo Z2 → Z6

### 3.1 Objetivo

Validar que pfSense impide el tráfico directo desde la zona IT/Z2 hacia la zona OT/Z6, mitigando el riesgo de movimiento lateral entre zonas.

### 3.2 Trazabilidad

* Escenario mitigado: **A.4 — Movimiento lateral IT → OT**.
* Requisitos verificados:

  * **RF-04 — Segmentación de red**.
  * **RF-06 — Registro de eventos**.

### 3.3 Procedimiento resumido

Desde `srv-it` (`10.10.20.10`) se intentó comunicar con `srv-ot` (`10.10.60.10`) mediante:

```bash
ping -c 3 -W 2 10.10.60.10
sudo nmap -sS -p 22,80,502 -Pn 10.10.60.10
curl --max-time 5 http://10.10.60.10
```

Posteriormente se revisaron:

* Logs de firewall en pfSense.
* Alertas de seguridad en Wazuh.

### 3.4 Resultado esperado

El tráfico debía ser bloqueado por pfSense y registrado tanto en el propio cortafuegos como en Wazuh.

### 3.5 Resultado obtenido

Pendiente de completar con el resultado real.

Texto sugerido si la prueba fue correcta:

```text
El tráfico desde srv-it hacia srv-ot fue bloqueado correctamente. pfSense registró los intentos de conexión mediante la regla DENEGACIÓN Z2 → Z6 (TC-01) y Wazuh generó alertas específicas con Rule ID 100201.
```

### 3.6 Evidencias

* `06-evidencias/TC-01-terminal-srv-it.png`
* `06-evidencias/TC-01-pfsense-firewall-log.png`
* `06-evidencias/TC-01-wazuh-alerta-100201.png`

## 4\. Caso TC-02 — Detección FIM sobre `/etc/hosts`

### 4.1 Objetivo

Validar que Wazuh detecta modificaciones en un archivo crítico del sistema mediante File Integrity Monitoring.

### 4.2 Trazabilidad

* Escenarios mitigados:

  * **E.1 — Modificación de configuración**.
  * **A.5 — Escalada de privilegios**.
* Requisitos verificados:

  * **RF-06 — Registro de eventos**.
  * **RF-07 — Alertas centralizadas**.

### 4.3 Procedimiento resumido

En `srv-it` se realizó una modificación controlada sobre `/etc/hosts`:

```bash
cp /etc/hosts /tmp/hosts.backup
echo "192.0.2.42 evil.example.com" | sudo tee -a /etc/hosts
tail -1 /etc/hosts
date
```

Después se revisó Wazuh en:

```text
Modules → Integrity monitoring
```

y, de forma alternativa, en:

```text
Modules → Security events
```

### 4.4 Resultado esperado

Wazuh debía detectar la modificación de `/etc/hosts`, asociarla al agente `srv-it` y generar un evento FIM en un tiempo reducido.

### 4.5 Resultado obtenido

Pendiente de completar con el resultado real.

Texto sugerido si la prueba fue correcta:

```text
Wazuh detectó correctamente la modificación de /etc/hosts en srv-it. La alerta apareció asociada al agente srv-it en el módulo de Integrity Monitoring. El tiempo de detección fue de X segundos.
```

### 4.6 Restauración

Tras obtener la evidencia, se restauró el fichero original:

```bash
sudo cp /tmp/hosts.backup /etc/hosts
```

### 4.7 Evidencias

* `06-evidencias/TC-02-terminal-modificacion-hosts.png`
* `06-evidencias/TC-02-wazuh-integrity-monitoring.png`
* `06-evidencias/TC-02-wazuh-security-events.png`

## 5\. Caso TC-03 — Simulacro tabletop NIS2

### 5.1 Objetivo

Validar el procedimiento organizativo de respuesta a incidentes mediante un ejercicio de mesa basado en tres escenarios: ransomware, phishing y compromiso de proveedor.

### 5.2 Trazabilidad

* Requisito verificado: **RF-11 — Procedimientos de respuesta**.
* Referencia normativa simulada: artículo 23 de la Directiva NIS2.
* Plazos considerados:

  * 24 horas para notificación temprana.
  * 72 horas para notificación intermedia.
  * 1 mes para informe final.

### 5.3 Procedimiento resumido

Se ejecutaron tres escenarios:

1. Ransomware en el entorno corporativo.
2. Phishing exitoso con compromiso de cuenta.
3. Exfiltración por compromiso de proveedor.

Para cada escenario se realizó:

* Lectura del escenario.
* Clasificación de gravedad.
* Identificación de medidas de contención.
* Redacción de notificación.
* Registro de tiempos y decisiones.

### 5.4 Resultado esperado

El ejercicio debía producir:

* Tres borradores de notificación.
* Un acta del ejercicio.
* Una tabla de tiempos.
* Una lista de acciones de mejora.

### 5.5 Resultado obtenido

Pendiente de completar tras la ejecución.

Texto sugerido si la prueba fue correcta:

```text
El simulacro tabletop se ejecutó correctamente. Se redactaron tres borradores de notificación, se documentó el acta del ejercicio, se registraron los tiempos de respuesta y se identificaron acciones de mejora para el ciclo de mejora continua.
```

### 5.6 Evidencias

`06-evidencias/TC-03md



### 6\. Tabla resumen de resultados

|Caso|Estado|Resultado|
|-|-|-|
|TC-01|Pendiente de cerrar|Bloqueo Z2 → Z6 y registro en pfSense/Wazuh|
|TC-02|Pendiente de cerrar|Detección FIM sobre `/etc/hosts`|
|TC-03|Pendiente de cerrar|Tres notificaciones tabletop NIS2|

## 7\. Desviaciones y observaciones

Pendiente de completar.

Ejemplos de observaciones posibles:

* La alerta de TC-01 requirió una regla local específica en Wazuh para evitar ruido por logs genéricos de pfSense.
* La alerta de TC-02 puede tardar entre 5 y 60 segundos dependiendo de la configuración de FIM.
* El TC-03 puede ejecutarse como autoejercicio si no hay participantes adicionales.

## 8\. Conclusión final

Pendiente de completar.

Texto sugerido si los tres casos fueron satisfactorios:

```text
Los tres casos de prueba se ejecutaron satisfactoriamente. La maqueta demostró capacidad de segmentación, registro, detección y respuesta documental ante incidentes. Los resultados obtenidos son coherentes con los objetivos del Sprint 5 y con los requisitos definidos en la memoria.
```

## 9\. Recomendación para la defensa oral

Durante la defensa conviene destacar:

1. La evidencia visual de TC-01, porque demuestra segmentación real.
2. La evidencia de TC-02, porque demuestra detección sobre un archivo crítico.
3. El TC-03, porque completa la parte técnica con la dimensión organizativa y normativa.
4. La coherencia entre la maqueta, la memoria y la Tabla 5.3 de resultados.

