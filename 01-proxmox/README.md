01-proxmox
## Problemas y soluciones encontrados - Sprint 0
===

# 

# | Problema encontrado | Causa identificada | Cómo se ha resuelto |

# |---|---|---|

# | Error al arrancar la VM: “VMware Workstation no admite contadores de rendimiento virtualizados en este host. Error al encender el módulo VPMC.” | La VM de Proxmox tenía activada la opción “Virtualize CPU performance counters”, pero el host Windows/VMware no podía exponer esos contadores virtualizados. | Se desmarcó en VMware Workstation la opción `Virtualize CPU performance counters` dentro de `Settings > Processors`. Con ello desapareció el error del módulo VPMC. |

# | Error al arrancar la VM: “La función 'hv.capable' era 0, pero debería ser al menos 0x1. Error al encender el módulo FeatureCompatLate.” | VMware no podía exponer correctamente la virtualización anidada requerida por Proxmox. La causa probable era interferencia de Hyper-V/VBS o configuración incompleta de virtualización. | Se revisó la configuración de la VM y se mantuvo activada la opción `Virtualize Intel VT-x/EPT or AMD-V/RVI`, necesaria para Proxmox. También se revisaron las funciones de virtualización de Windows para evitar conflictos con VMware. |

# | Proxmox arrancaba correctamente, pero no era posible acceder al panel web desde el navegador. | Aunque Proxmox mostraba una URL de acceso, Windows no podía llegar a esa IP. El acceso a `https://IP:8006` fallaba. | Se comprobó la conectividad desde Windows con `ping` y `Test-NetConnection IP -Port 8006`, confirmando que el problema era de red y no del navegador. |

# | Proxmox no respondía al ping desde Windows. | Proxmox estaba configurado en una red distinta a la red real del equipo anfitrión. Windows tenía IP `192.168.1.141/24` con gateway `192.168.1.1`, mientras que Proxmox tenía IP `192.168.100.2/24` con gateway `192.168.100.1`. | Se identificó la diferencia de red comparando `ipconfig` en Windows con `ip a` e `ip route` en Proxmox. |

# | VMware mostraba redes VMnet1 y VMnet8, pero inicialmente no estaba claro qué red debía usarse. | VMnet1 es una red solo-host y VMnet8 es NAT. Para este despliegue se necesitaba que Proxmox estuviera en la misma red que Windows mediante bridge. | Se accedió a `Editar > Editor de red virtual > Cambiar configuración` y se comprobó que `VMnet0` estuviera configurado como `Puente` hacia la tarjeta WiFi real `Intel(R) Wireless-AC 9560 160MHz`. |

# | Proxmox seguía inaccesible aunque VMnet0 estaba en modo puente. | La configuración de puente de VMware era correcta, pero la IP fija de Proxmox seguía perteneciendo a otra subred distinta. | Se corrigió la configuración de red de Proxmox editando `/etc/network/interfaces`. |

# | IP incorrecta en Proxmox: `192.168.100.2/24`. | Durante la instalación se había asignado una IP estática que no pertenecía a la red actual del equipo anfitrión. | Se cambió la IP de Proxmox a una dirección libre dentro de la red WiFi real, por ejemplo `192.168.1.200/24`, y se configuró como gateway `192.168.1.1`. |

# | Acceso web a Proxmox no disponible antes de corregir la IP. | Windows y Proxmox no estaban en la misma subred, por lo que el navegador no podía alcanzar el puerto `8006`. | Tras corregir la IP y reiniciar Proxmox, se comprobó la conectividad con `ping 192.168.1.200` y `Test-NetConnection 192.168.1.200 -Port 8006`. Finalmente se accedió correctamente desde el navegador mediante `https://192.168.1.200:8006`. |

# 

# \## Configuración final corregida

# 

# Windows:

# 

# ```text

# IP WiFi: 192.168.1.141

# Máscara: 255.255.255.0

Gateway: 192.168.1.1

Proxmox:

IP vmbr0: 192.168.1.200/24
Gateway: 192.168.1.1
Interfaz puente: ens33
Bridge: vmbr0

VMware Workstation:

VMnet0: Puente
Adaptador físico: Intel(R) Wireless-AC 9560 160MHz
Adaptador de red de la VM Proxmox: Bridged
Virtualize Intel VT-x/EPT or AMD-V/RVI: activado
Virtualize CPU performance counters: desactivado

===

# Sprint 1 — Bridges de red y plantilla Debian 12

## Objetivo del Sprint 1

El objetivo del Sprint 1 ha sido preparar la base técnica sobre la que se desplegarán posteriormente las máquinas virtuales de la maqueta. Según el plan de despliegue, este sprint incluye la creación de los bridges de red `vmbr1` y `vmbr2`, la subida de las ISOs necesarias a Proxmox y la creación de una plantilla Debian 12 reutilizable para clonar las futuras VMs del laboratorio. :contentReference[oaicite:0]{index=0}

---

## Tareas realizadas

### 1. Creación y revisión de redes virtuales

Se ha trabajado con la red principal `vmbr0`, que conecta Proxmox con la red externa a través de VMware Workstation.

También se ha avanzado en la preparación de las redes internas previstas para la maqueta:

- `vmbr0`: red principal/WAN, conectada hacia la red del anfitrión.
- `vmbr1`: red corporativa Z2.
- `vmbr2`: red OT/Z6.

Durante esta fase se ha comprobado que la red de Proxmox dependía directamente de la red WiFi utilizada por el equipo anfitrión. Al cambiar de WiFi, la IP fija configurada anteriormente dejó de ser válida.

---

### 2. Subida de ISOs a Proxmox

Se ha revisado el procedimiento para subir ISOs a Proxmox desde el almacenamiento `local`.

La ruta correcta utilizada ha sido:

```text
Datacenter > nodo Proxmox > local > ISO Images > Upload


Sprint 1 — Bridges de red y plantilla Debian 12
Objetivo del Sprint 1
El objetivo del Sprint 1 ha sido preparar la base técnica sobre la que se desplegarán posteriormente las máquinas virtuales de la maqueta. En este sprint se ha trabajado en la preparación de las redes virtuales de Proxmox, la subida de ISOs y la creación de una plantilla Debian 12 reutilizable para clonar las futuras máquinas virtuales del laboratorio.
---
Tareas realizadas
1. Revisión de la red principal de Proxmox
Se ha trabajado con la red principal `vmbr0`, que conecta Proxmox con la red externa a través de VMware Workstation.
La red principal se ha utilizado como red de salida temporal para instalar y preparar la máquina Debian base.
También se ha revisado la función de los bridges previstos para la maqueta:
`vmbr0`: red principal/WAN, conectada hacia la red del anfitrión.
`vmbr1`: red corporativa Z2.
`vmbr2`: red OT/Z6.
Durante esta fase se ha comprobado que la red de Proxmox depende directamente de la red WiFi utilizada por el equipo anfitrión. Al cambiar de WiFi, la IP fija configurada anteriormente dejó de ser válida.
---
2. Subida de ISOs a Proxmox
Se ha revisado el procedimiento para subir imágenes ISO a Proxmox desde el almacenamiento `local`.
La ruta correcta utilizada ha sido:
```text
Datacenter > nodo Proxmox > local > ISO Images > Upload
```
Se aclaró que no debe utilizarse `local-lvm`, ya que este almacenamiento está destinado principalmente a discos de máquinas virtuales, no a imágenes ISO.
También se identificó que descargar las ISOs directamente desde Proxmox era muy lento. Por ello, se decidió como mejor opción descargar las ISOs desde Windows y después subirlas a Proxmox mediante la opción `Upload`.
---
3. Creación de la VM base Debian 12
Se creó una máquina virtual Debian 12 para utilizarla como base de la plantilla.
La máquina se instaló correctamente y se accedió a ella desde la consola de Proxmox. Se comprobó su dirección IP mediante:
```bash
ip a
```
La interfaz detectada dentro de Debian fue:
```text
ens18
```
Inicialmente, la VM obtuvo una IP de la red externa, lo que permitió continuar con la instalación de paquetes y la configuración básica.
---
4. Preparación básica de la plantilla Debian
Se ejecutaron los comandos de actualización del sistema:
```bash
sudo apt update
sudo apt upgrade -y
```
Después se instalaron paquetes útiles para administración, diagnóstico y funcionamiento de la maqueta:
```bash
sudo apt install -y curl wget vim net-tools htop git
sudo apt install -y qemu-guest-agent ca-certificates gnupg
sudo apt install -y nmap netcat-openbsd openssh-server
```
También se activó el servicio SSH:
```bash
sudo systemctl enable ssh
sudo systemctl start ssh
```
Se intentó trabajar con `qemu-guest-agent`, aunque aparecieron incidencias al intentar habilitarlo y arrancarlo.
Finalmente, se limpiaron los identificadores y claves propias de la máquina antes de convertirla en plantilla:
```bash
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id
sudo rm -f /etc/ssh/ssh_host_*
```
Esta limpieza es importante para evitar que todos los clones tengan el mismo identificador interno y las mismas claves SSH.
---
5. Prueba de clonado
Se creó una VM de prueba a partir de la plantilla para verificar que el clonado funcionaba correctamente.
Durante esta prueba se comprobó que:
El clon arrancaba correctamente.
La VM clonada obtenía una IP diferente.
El servicio SSH podía funcionar tras regenerar las claves.
La plantilla era utilizable como base para futuras máquinas virtuales.
Posteriormente, se decidió utilizar un clon donde SSH ya funcionaba correctamente como nueva base de plantilla, pero limpiándolo de nuevo antes de convertirlo definitivamente en template.
---
Problemas encontrados y soluciones aplicadas
Problema encontrado	Causa identificada	Solución aplicada
No se encontraba claramente el botón `Upload` para subir ISOs.	Se estaba buscando en una zona incorrecta de la interfaz de Proxmox o podía confundirse `local` con `local-lvm`.	Se identificó la ruta correcta: `Datacenter > nodo > local > ISO Images > Upload`. Se aclaró que las ISOs deben subirse a `local`, no a `local-lvm`.
La descarga de ISOs desde Proxmox era muy lenta.	Proxmox se ejecuta dentro de VMware Workstation, usando red virtualizada sobre WiFi, lo que ralentizaba la descarga.	Se decidió descargar las ISOs desde Windows y después subirlas a Proxmox mediante `Upload` o, alternativamente, mediante `scp`.
No era posible conectarse por SSH desde Windows a la VM Debian plantilla.	Aunque la VM tenía IP, Windows no podía alcanzar directamente las VMs internas creadas dentro de Proxmox, probablemente por la virtualización anidada VMware → Proxmox → Debian y por limitaciones del modo bridge sobre WiFi.	Se decidió continuar desde la consola de Proxmox, ya que para preparar la plantilla no era obligatorio acceder por SSH desde Windows.
Al instalar paquetes apareció un error indicando que no se encontraba el paquete `gnupg`.	El comando se había escrito incorrectamente, usando `\-y` en lugar de `-y`. Esto hacía que Debian interpretara mal la opción de instalación.	Se corrigió el comando usando `sudo apt install -y qemu-guest-agent ca-certificates gnupg`.
`qemu-guest-agent` mostró errores al intentar habilitarse con `systemctl enable`.	El servicio no estaba pensado para habilitarse de esa forma o faltaba la configuración del agente desde Proxmox.	Se comprobó que no era un error bloqueante. Se recomendó activar `QEMU Guest Agent` desde las opciones de la VM en Proxmox y continuar con el resto de la preparación.
`qemu-guest-agent` falló al arrancar con error de dependencias.	Probablemente la opción QEMU Guest Agent no estaba habilitada correctamente en la configuración de la VM dentro de Proxmox.	Se decidió continuar sin bloquear el sprint, ya que el agente no es imprescindible para que la plantilla funcione. Se dejó anotado como incidencia.
El clon de prueba obtuvo una IP `169.254.x.x`.	La VM clonada estaba conectada a una red sin DHCP funcional, probablemente `vmbr1`, que todavía no tenía pfSense configurado.	Se cambió temporalmente la red del clon a `vmbr0`, donde sí podía obtener una IP válida por DHCP.
El servicio SSH del clon aparecía como `failed`.	Al limpiar la plantilla se habían eliminado las claves SSH del host con `rm -f /etc/ssh/ssh_host_*`, y en el clon no se habían regenerado todavía.	Se regeneraron las claves SSH con `sudo ssh-keygen -A` y se reinició el servicio con `sudo systemctl restart ssh`. Tras ello, SSH quedó activo.
Windows seguía sin poder hacer ping ni SSH al clon, aunque SSH estaba activo dentro de la VM.	Limitación de conectividad desde Windows hacia VMs internas dentro de Proxmox, por la virtualización anidada y el uso de WiFi.	Se verificó desde la propia consola de la VM que el clon tenía IP diferente y SSH activo. Se dio por válida la prueba de clonado sin exigir conexión directa desde Windows.
Se convirtió un clon funcional en plantilla.	Se utilizó un clon donde SSH ya funcionaba como nueva base de template. Esto podía provocar duplicidad de `machine-id` y claves SSH si no se limpiaba antes.	Se recomendó limpiar de nuevo `/etc/machine-id`, `/var/lib/dbus/machine-id` y `/etc/ssh/ssh_host_*` antes de convertir esa VM en plantilla definitiva.
---
Comandos relevantes utilizados
Comprobación de red en Debian
```bash
ip a
ip route
```
Actualización del sistema
```bash
sudo apt update
sudo apt upgrade -y
```
Instalación de paquetes
```bash
sudo apt install -y curl wget vim net-tools htop git
sudo apt install -y qemu-guest-agent ca-certificates gnupg
sudo apt install -y nmap netcat-openbsd openssh-server
```
Activación de SSH
```bash
sudo systemctl enable ssh
sudo systemctl start ssh
sudo systemctl status ssh
```
Regeneración de claves SSH en clones
```bash
sudo ssh-keygen -A
sudo systemctl restart ssh
sudo systemctl status ssh
```
Limpieza de plantilla
```bash
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id
sudo rm -f /etc/ssh/ssh_host_*
sudo poweroff
```
---
Resultado final del Sprint 1
El Sprint 1 queda completado con los siguientes resultados:
Proxmox operativo y accesible.
Almacenamiento `local` identificado correctamente para subir ISOs.
ISOs preparadas para su uso en la creación de VMs.
VM Debian 12 instalada correctamente.
Paquetes básicos instalados.
SSH instalado y funcional.
Plantilla Debian preparada para clonado.
Clon de prueba creado correctamente.
Problemas de claves SSH solucionados mediante `ssh-keygen -A`.
Se ha identificado que la conectividad directa desde Windows hacia las VMs internas puede no funcionar por la virtualización anidada sobre WiFi, pero no bloquea el avance del laboratorio.

