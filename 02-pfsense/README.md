# Sprint 2 — pfSense: firewall de tres zonas

## Objetivo

El objetivo de este sprint ha sido desplegar y configurar una máquina virtual pfSense dentro de Proxmox para actuar como firewall central de la maqueta reducida del TFG EolSolar.

pfSense queda situado entre tres redes o zonas:

* `WAN`, conectada a la red externa a través de `vmbr0`.
* `Z2\_USERS`, red corporativa o de usuarios, conectada a `vmbr1`.
* `Z6\_OT`, red OT/industrial, conectada a `vmbr2`.

La finalidad principal es comprobar que el tráfico entre la zona corporativa y la zona OT queda controlado por reglas de firewall, especialmente la denegación del flujo directo desde Z2 hacia Z6.

\---

## Máquina virtual pfSense

Configuración principal de la VM:

|Parámetro|Valor|
|-|-|
|VM ID|`110`|
|Nombre|`pfsense`|
|Sistema|Other / pfSense CE|
|ISO utilizada|Netgate Installer / pfSense CE|
|CPU|2 cores|
|Memoria|2 GB|
|Disco|10–15 GB|
|Almacenamiento|`local-lvm`|
|Modelo de red|VirtIO|
|Firewall de Proxmox|Desmarcado en todas las interfaces|

\---

## Interfaces de red

La VM pfSense se ha configurado con tres interfaces de red virtuales.

|Interfaz pfSense|Interfaz Proxmox|Bridge|Zona|Función|
|-|-|-|-|-|
|`WAN`|`vtnet0`|`vmbr0`|Red externa|Salida hacia la red del anfitrión / Internet|
|`Z2\_USERS`|`vtnet1`|`vmbr1`|Z2|Red corporativa / usuarios|
|`Z6\_OT`|`vtnet2`|`vmbr2`|Z6|Red OT / industrial|

La correspondencia esperada es:

```text
vtnet0 -> vmbr0 -> WAN
vtnet1 -> vmbr1 -> Z2\_USERS
vtnet2 -> vmbr2 -> Z6\_OT
```

\---

## Subredes asignadas

|Zona|Interfaz|Dirección|DHCP|
|-|-|-|-|
|WAN|`vtnet0`|DHCP de la red externa|Sí, desde la red del anfitrión|
|Z2\_USERS|`vtnet1`|`10.10.20.1/24`|Sí, rango `10.10.20.100` a `10.10.20.200`|
|Z6\_OT|`vtnet2`|`10.10.60.1/24`|No|

La red `Z2\_USERS` se utiliza para las máquinas del entorno corporativo. La red `Z6\_OT` se utiliza para las máquinas del entorno OT. La comunicación entre ambas redes debe pasar siempre por pfSense.

\---

## Reglas de firewall configuradas

### WAN

Durante la configuración inicial fue necesario permitir temporalmente el acceso al panel web de pfSense desde la red externa, ya que todavía no existía una VM en Z2 desde la que acceder a `https://10.10.20.1`.

Reglas utilizadas temporalmente en WAN:

|Orden|Acción|Origen|Destino|Puerto|Descripción|
|-|-|-|-|-|-|
|1|Pass|IP del equipo Windows anfitrión|This Firewall|HTTPS 443|Acceso temporal al panel pfSense desde Windows|
|2|Pass|Any|Any|Any|Regla temporal creada mediante `pfSsh.php playback enableallowallwan`|

La segunda regla es demasiado permisiva y debe eliminarse cuando ya no sea necesaria. La regla de acceso temporal por HTTPS también debe eliminarse cuando se pueda acceder a pfSense desde una VM situada en la red Z2.

Estado final recomendado para WAN:

```text
WAN: reglas por defecto, bloqueando entrada no solicitada.
```

\---

### Z2\_USERS

Reglas recomendadas y aplicadas en la interfaz `Z2\_USERS`:

|Orden|Acción|Origen|Destino|Puerto|Log|Descripción|
|-|-|-|-|-|-|-|
|1|Pass|`Z2\_USERS net`|This Firewall|HTTPS 443|No necesario|Anti-Lockout Rule / acceso al panel desde Z2|
|2|Block|`Z2\_USERS net`|`10.10.60.0/24`|Any|Sí|DENEGACIÓN Z2 -> Z6 (TC-01)|
|3|Pass|`Z2\_USERS net`|Any|Any|No|Permitir salida desde Z2|

La regla crítica es la segunda, ya que bloquea cualquier intento de comunicación desde la red corporativa Z2 hacia la red OT Z6.

Es importante que esta regla esté colocada antes de la regla `Pass Z2 -> any`, porque pfSense evalúa las reglas de arriba hacia abajo y aplica la primera coincidencia.

\---

### Z6\_OT

Reglas recomendadas y aplicadas en la interfaz `Z6\_OT`:

|Orden|Acción|Origen|Destino|Puerto|Log|Descripción|
|-|-|-|-|-|-|-|
|1|Block|`Z6\_OT net`|Any|Any|Sí|Z6 no inicia conexiones|

Esta regla impide que la red OT inicie conexiones hacia otras redes o hacia Internet. El objetivo es simular una red industrial aislada y controlada.

\---

## Comprobación esperada de reglas

El conjunto final de reglas debe quedar así:

|Interfaz|Reglas esperadas|
|-|-|
|WAN|Reglas por defecto, bloqueando entrada no solicitada. Puede existir una regla temporal de administración mientras se configura el laboratorio.|
|Z2\_USERS|1. Pass HTTPS al firewall. 2. Block Z2 -> `10.10.60.0/24` + LOG. 3. Pass Z2 -> any.|
|Z6\_OT|1. Block Z6 -> any + LOG.|

\---

## Exportación y restauración de configuración

### Exportar configuración

Una vez configurado pfSense, se debe guardar una copia de seguridad del archivo de configuración.

Ruta en el panel web:

```text
Diagnostics -> Backup \& Restore -> Backup Configuration
```

Opciones recomendadas:

* Marcar `Do not backup RRD data`.
* Descargar el archivo `config.xml`.
* Guardarlo en el repositorio del TFG en:

```text
02-pfsense/config.xml
```

También conviene guardar capturas de pantalla de:

* Interfaces configuradas.
* Reglas de `WAN`.
* Reglas de `Z2\_USERS`.
* Reglas de `Z6\_OT`.

\---

### Restaurar configuración

Si pfSense se rompe o hay que reinstalar la VM, se puede restaurar la configuración desde el panel web.

Pasos:

1. Instalar pfSense de nuevo en una VM con las mismas interfaces.
2. Acceder al panel web.
3. Ir a:

```text
Diagnostics -> Backup \& Restore -> Restore Backup
```

4. Seleccionar el archivo:

```text
02-pfsense/config.xml
```

5. Restaurar la configuración.
6. Reiniciar pfSense si lo solicita.
7. Comprobar que las interfaces siguen en el orden correcto:

```text
vtnet0 -> WAN
vtnet1 -> Z2\_USERS
vtnet2 -> Z6\_OT
```

Si el orden de interfaces cambia, puede ser necesario reasignarlas desde la consola de pfSense mediante la opción:

```text
1) Assign Interfaces
```

\---

## Inconvenientes encontrados y soluciones aplicadas

### 1\. Dificultad para descargar la ISO clásica de pfSense

**Problema:**

No se encontraba fácilmente una ISO clásica offline de pfSense CE. La página oficial redirigía al Netgate Installer.

**Causa:**

Netgate distribuye actualmente pfSense mediante el Netgate Installer, que descarga componentes durante la instalación.

**Solución:**

Se decidió utilizar el Netgate Installer, manteniendo la configuración de la VM como `Other OS`, con 2 GB de RAM y la primera interfaz conectada a `vmbr0` para garantizar salida a Internet durante la instalación.

\---

### 2\. Instalación lenta de pfSense

**Problema:**

La instalación era muy lenta y en algunos momentos parecía bloqueada.

**Causa:**

El Netgate Installer necesita descargar paquetes desde Internet. La conexión pasa por varias capas:

```text
WiFi -> Windows -> VMware Workstation -> Proxmox -> VM pfSense
```

Esto aumenta la latencia y puede ralentizar mucho la descarga.

**Solución:**

Se mantuvo la VM con 2 GB de RAM y se comprobó que `net0` estuviera conectado a `vmbr0`, ya que esa interfaz actúa como WAN. Se continuó la instalación cuando el proceso avanzaba, aunque fuese lentamente.

\---

### 3\. Error de servidor durante la descarga del instalador

**Problema:**

Durante la instalación con Netgate Installer apareció un error de servidor.

**Causa:**

El problema no parecía estar relacionado con la memoria de la VM, sino con la dependencia del instalador respecto a los servidores externos de Netgate o con la conexión de red.

**Solución:**

Se verificó que 2 GB de RAM eran suficientes para pfSense y que el problema principal era la descarga online. Se recomendó repetir la instalación manteniendo `vtnet0/vmbr0` como interfaz WAN con DHCP.

\---

### 4\. Bloqueos frecuentes del acceso web a pfSense

**Problema:**

Durante la configuración inicial, la página web de pfSense se bloqueaba o dejaba de responder con frecuencia.

**Causa:**

Se estaba accediendo al panel web por la interfaz WAN. pfSense, por seguridad, bloquea por defecto el acceso administrativo desde WAN. Al aplicar cambios o recargar reglas, se perdía el acceso.

**Solución:**

Desde la consola de pfSense se entró en Shell y se ejecutó:

```sh
pfSsh.php playback enableallowallwan
```

Esto permitió crear una regla temporal de acceso por WAN. Después se creó una regla más controlada permitiendo únicamente HTTPS desde la IP del equipo Windows anfitrión hacia `This Firewall`.

\---

## Estado final del Sprint 2

El Sprint 2 queda completado cuando se cumplen estas condiciones:

* pfSense está instalado y arrancando correctamente.
* Tiene tres interfaces asignadas: `WAN`, `Z2\_USERS` y `Z6\_OT`.
* `Z2\_USERS` tiene IP `10.10.20.1/24`.
* `Z6\_OT` tiene IP `10.10.60.1/24`.
* La regla de bloqueo `Z2 -> Z6` está antes de la regla de salida general de Z2.
* Las reglas de bloqueo tienen activado el registro de logs.
* Se ha exportado `config.xml` y guardado en `02-pfsense/config.xml`.
* Se ha documentado la configuración en este `README.md`.

\---

## Conclusión

pfSense queda configurado como elemento central de segmentación de la maqueta. La red corporativa `Z2\_USERS` puede salir hacia Internet, pero no puede acceder directamente a la red OT `Z6\_OT`. La red `Z6\_OT`, por su parte, no inicia conexiones hacia otras redes.

Esta configuración permite validar posteriormente el caso de prueba TC-01, en el que se comprobará que el tráfico desde Z2 hacia Z6 queda bloqueado y registrado en los logs del firewall.

