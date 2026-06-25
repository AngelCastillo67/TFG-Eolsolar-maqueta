# Sprint 3 — VMs Debian ligeras: `srv-it` y `srv-ot`

## Objetivo del Sprint 3

El objetivo de este sprint ha sido desplegar y configurar las dos máquinas Debian ligeras de la maqueta:

- `srv-it`, ubicada en la zona corporativa `Z2_USERS`.
- `srv-ot`, ubicada en la zona industrial/OT `Z6_OT`.

Estas dos máquinas sirven para validar la segmentación aplicada con pfSense y preparar los servicios necesarios para las pruebas posteriores del TFG.

---

## Arquitectura configurada

| Máquina | Zona | Bridge Proxmox | IP | Gateway | Función |
|---|---|---|---|---|---|
| `srv-it` | Z2_USERS | `vmbr1` | `10.10.20.10/24` | `10.10.20.1` | Servidor corporativo de pruebas |
| `srv-ot` | Z6_OT | `vmbr2` | `10.10.60.10/24` | `10.10.60.1` | Servidor OT de pruebas |
| `pfSense` | Firewall | `vmbr0`, `vmbr1`, `vmbr2` | `10.10.20.1` y `10.10.60.1` | — | Segmentación y control de tráfico |

---

## Configuración realizada en `srv-it`

### Clonado desde la plantilla Debian

Se creó la VM `srv-it` clonando la plantilla Debian preparada en el Sprint 1.

```text
VM ID: 120
Name: srv-it
Bridge: vmbr1
Zona: Z2_USERS
IP estática: 10.10.20.10
Gateway: 10.10.20.1
```

### Configuración de red de `srv-it`

Se editó el fichero:

```bash
sudo nano /etc/network/interfaces
```

Configuración aplicada:

```text
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

allow-hotplug ens18
iface ens18 inet static
    address 10.10.20.10
    netmask 255.255.255.0
    gateway 10.10.20.1
    dns-nameservers 8.8.8.8 1.1.1.1
```

Después se reinició la red:

```bash
sudo systemctl restart networking
```

o, en caso necesario:

```bash
sudo reboot
```

### Pruebas iniciales de conectividad en `srv-it`

Se realizaron las siguientes pruebas:

```bash
ping -c 3 10.10.20.1
ping -c 3 8.8.8.8
nslookup google.com
```

Resultados:

| Prueba | Resultado esperado | Resultado obtenido |
|---|---|---|
| `ping 10.10.20.1` | Correcto | Correcto |
| `ping 8.8.8.8` | Correcto | Correcto |
| `nslookup google.com` | Debía resolver DNS | Falló inicialmente |

---

## Problema DNS en `srv-it`

### Problema encontrado

Aunque `srv-it` tenía conexión con pfSense y salida a Internet por IP, no resolvía nombres DNS.

El comando:

```bash
nslookup google.com
```

devolvía errores de este tipo:

```text
communications error to ::1#53: connection refused
communications error to 127.0.0.1#53: connection refused
no servers could be reached
```

### Causa

La VM estaba intentando usar como DNS el propio localhost:

```text
127.0.0.1
::1
```

en lugar de utilizar los DNS externos configurados:

```text
8.8.8.8
1.1.1.1
```

### Solución aplicada

Se corrigió el fichero `/etc/resolv.conf` para usar servidores DNS externos.

```bash
sudo rm -f /etc/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
echo "nameserver 1.1.1.1" | sudo tee -a /etc/resolv.conf
```

Después se comprobó de nuevo:

```bash
nslookup google.com
```

Tras el cambio, la resolución DNS funcionó correctamente.

---

## Configuración realizada en `srv-ot`

### Clonado desde la plantilla Debian

Se creó la VM `srv-ot` clonando la plantilla Debian preparada previamente.

```text
VM ID: 130
Name: srv-ot
Bridge: vmbr2
Zona: Z6_OT
IP estática: 10.10.60.10
Gateway: 10.10.60.1
```

### Configuración de red de `srv-ot`

Se editó el fichero:

```bash
sudo nano /etc/network/interfaces
```

Configuración aplicada:

```text
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

allow-hotplug ens18
iface ens18 inet static
    address 10.10.60.10
    netmask 255.255.255.0
    gateway 10.10.60.1
    dns-nameservers 8.8.8.8 1.1.1.1
```

Después se reinició la red:

```bash
sudo systemctl restart networking
```

o, en caso necesario:

```bash
sudo reboot
```

---

## Problema de conectividad con el gateway Z6

### Problema encontrado

Al probar desde `srv-ot`:

```bash
ping -c 3 10.10.60.1
```

el resultado fue:

```text
100% packet loss
```

Esto indicaba que `srv-ot` no podía llegar a su gateway `10.10.60.1`.

### Causa

La regla de firewall de `Z6_OT` en pfSense estaba bloqueando todo el tráfico iniciado desde Z6, incluyendo el ping hacia el propio firewall.

La política final del escenario indicaba:

```text
Z6_OT:
1. Block Z6 → any + LOG
```

Por tanto, el bloqueo era coherente con una política estricta, pero impedía hacer una prueba básica de conectividad contra el gateway.

### Solución temporal aplicada

Se creó una regla temporal de diagnóstico en pfSense:

```text
Action: Pass
Interface: Z6_OT
Protocol: ICMP
Source: Z6_OT net
Destination: This Firewall / self
Description: Permitir ping a gateway Z6 para prueba
```

Con esta regla, `srv-ot` pudo hacer ping correctamente a:

```text
10.10.60.1
```

### Decisión final

Esta regla se considera temporal y solo se utilizó para validar que:

- `srv-ot` estaba correctamente conectado a `vmbr2`.
- La interfaz `Z6_OT` de pfSense estaba activa.
- El gateway `10.10.60.1` respondía correctamente.

Para el escenario final, la regla debe eliminarse o desactivarse si se quiere cumplir estrictamente la política:

```text
Z6_OT:
1. Block Z6_OT net → any + LOG
```

---

## Instalación de servicios en `srv-ot`

### Objetivo

`srv-ot` debe exponer servicios de prueba en los puertos:

- `22/tcp`, para SSH.
- `80/tcp`, para HTTP mediante `nginx`.

Estos servicios serán utilizados en las pruebas de conectividad y bloqueo desde `srv-it`.

### Problema encontrado

`srv-ot` está en la red `Z6_OT`, que tiene bloqueada la salida a Internet. Por tanto, no podía ejecutar:

```bash
sudo apt update
sudo apt install -y nginx openssh-server
```

directamente desde `vmbr2`.

### Solución aplicada: método 1

Se aplicó el método temporal indicado en el plan:

1. Apagar `srv-ot`.
2. Cambiar temporalmente el bridge de la VM desde `vmbr2` a `vmbr0`.
3. Cambiar temporalmente la red de Debian a DHCP.
4. Instalar los paquetes necesarios.
5. Volver a colocar la VM en `vmbr2`.
6. Restaurar la IP estática de Z6.

### Cambio temporal de red a DHCP

Al cambiar la VM a `vmbr0`, la configuración estática `10.10.60.10` ya no era válida. Por tanto, se modificó temporalmente `/etc/network/interfaces`:

```text
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

allow-hotplug ens18
iface ens18 inet dhcp
```

Después se reinició la red o la VM:

```bash
sudo systemctl restart networking
```

o:

```bash
sudo reboot
```

Con ello, `srv-ot` obtuvo una IP de la red externa y pudo acceder a Internet.

### Instalación de paquetes

Una vez con salida a Internet, se ejecutó:

```bash
sudo apt update
sudo apt install -y openssh-server nginx
```

Después se activaron los servicios:

```bash
sudo systemctl enable --now nginx ssh
```

---

## Problema con SSH en `srv-ot`

### Problema encontrado

Al intentar activar SSH, apareció el siguiente error:

```text
Job for ssh.service failed because the control process exited with error code.
```

### Causa

La causa fue que las claves SSH de host habían sido eliminadas en la preparación de la plantilla mediante:

```bash
sudo rm -f /etc/ssh/ssh_host_*
```

Por tanto, el servicio SSH no podía arrancar correctamente hasta regenerarlas.

### Solución aplicada

Se regeneraron las claves SSH con:

```bash
sudo ssh-keygen -A
```

Después se reinició SSH:

```bash
sudo systemctl restart ssh
```

Y se comprobó su estado:

```bash
sudo systemctl status ssh
```

Resultado esperado:

```text
active (running)
```

---

## Verificación de servicios en `srv-ot`

Se comprobó que `nginx` y `ssh` estaban activos:

```bash
sudo systemctl status nginx
sudo systemctl status ssh
```

También se verificaron los puertos abiertos:

```bash
ss -tlnp | grep -E ':(22|80) '
```

Resultado esperado:

```text
Puerto 22 escuchando
Puerto 80 escuchando
```

---

## Restauración de `srv-ot` a Z6

Una vez instalados los servicios, se apagó la VM:

```bash
sudo poweroff
```

En Proxmox se cambió de nuevo el bridge:

```text
srv-ot > Hardware > Network Device > Bridge: vmbr2
```

Después se restauró la configuración estática de Z6:

```text
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

allow-hotplug ens18
iface ens18 inet static
    address 10.10.60.10
    netmask 255.255.255.0
    gateway 10.10.60.1
    dns-nameservers 8.8.8.8 1.1.1.1
```

Y se reinició:

```bash
sudo reboot
```

---

## Pruebas cruzadas de conectividad

Desde `srv-it` se realizaron las siguientes pruebas:

```bash
ping -c 3 10.10.20.1
ping -c 3 8.8.8.8
ping -c 3 10.10.60.10
ping -c 3 10.10.60.1
```

Resultados esperados:

| Prueba desde `srv-it` | Resultado esperado |
|---|---|
| `ping 10.10.20.1` | OK |
| `ping 8.8.8.8` | OK |
| `ping 10.10.60.10` | Debe fallar |
| `ping 10.10.60.1` | Debe fallar |

También se probaron otros protocolos hacia `srv-ot`:

```bash
nc -zvw 5 10.10.60.10 22
nc -zvw 5 10.10.60.10 80
curl --max-time 5 http://10.10.60.10
nmap -sS -p 22,80 10.10.60.10
```

Todos estos intentos desde Z2 hacia Z6 deben fallar.

---

## Problema en regla de bloqueo Z2 → Z6

### Problema encontrado

Durante las pruebas, desde `srv-it` se podía hacer ping al gateway de Z6:

```bash
ping -c 3 10.10.60.1
```

Esto no era correcto según la política de segmentación.

### Causa

La regla de bloqueo `DENEGACIÓN Z2 → Z6 (TC-01)` no tenía configurado el destino como una red completa.

El campo `Destination` no estaba configurado como:

```text
Network 10.10.60.0/24
```

Por tanto, no estaba bloqueando correctamente toda la subred Z6.

### Solución aplicada

Se corrigió la regla en pfSense:

```text
Action: Block
Interface: Z2_USERS
Protocol: any
Source: Z2_USERS net
Destination: Network 10.10.60.0/24
Log: activado
Description: DENEGACIÓN Z2 → Z6 (TC-01)
```

La regla quedó colocada por encima de:

```text
Permitir salida desde Z2
```

Orden final recomendado en `Z2_USERS`:

```text
1. Anti-Lockout Rule
2. DENEGACIÓN Z2 → Z6 (TC-01)
3. Permitir salida desde Z2
```

Tras aplicar los cambios, el tráfico desde Z2 hacia `10.10.60.1` y `10.10.60.10` quedó bloqueado correctamente.

---

## Estado final de reglas relevantes

### Z2_USERS

```text
1. Pass HTTPS al firewall - Anti-Lockout Rule
2. Block Z2_USERS net → 10.10.60.0/24 + LOG
3. Pass Z2_USERS net → any
```

### Z6_OT

```text
1. Block Z6_OT net → any + LOG
```

Nota: durante la configuración se utilizó temporalmente una regla ICMP desde Z6 hacia el propio firewall para diagnóstico. Esta regla no forma parte del escenario final estricto.

---

## Problemas encontrados y soluciones

| Problema | Causa | Solución |
|---|---|---|
| `srv-it` no resolvía DNS | Usaba `127.0.0.1` y `::1` como DNS | Se corrigió `/etc/resolv.conf` con `8.8.8.8` y `1.1.1.1` |
| `srv-ot` no hacía ping a `10.10.60.1` | La regla `Block Z6 → any` bloqueaba también ICMP al firewall | Se creó una regla ICMP temporal para diagnóstico |
| `srv-ot` no podía instalar paquetes | Z6 no tiene salida a Internet | Se cambió temporalmente la VM a `vmbr0` y DHCP |
| Al cambiar `srv-ot` a `vmbr0`, seguía sin Internet | Debian mantenía IP estática `10.10.60.10` | Se cambió temporalmente `/etc/network/interfaces` a DHCP |
| SSH fallaba en `srv-ot` | Faltaban claves SSH de host | Se regeneraron con `sudo ssh-keygen -A` |
| Desde `srv-it` se podía llegar a `10.10.60.1` | La regla de bloqueo no tenía `Destination` como `Network` | Se corrigió a `Network 10.10.60.0/24` |

---

## Comandos principales utilizados

### Red

```bash
ip a
ip route
sudo nano /etc/network/interfaces
sudo systemctl restart networking
sudo reboot
```

### DNS

```bash
cat /etc/resolv.conf
sudo rm -f /etc/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
echo "nameserver 1.1.1.1" | sudo tee -a /etc/resolv.conf
nslookup google.com
```

### Servicios

```bash
sudo apt update
sudo apt install -y openssh-server nginx
sudo systemctl enable --now nginx ssh
sudo ssh-keygen -A
sudo systemctl restart ssh
sudo systemctl status ssh
sudo systemctl status nginx
ss -tlnp | grep -E ':(22|80) '
```

### Pruebas de conectividad

```bash
ping -c 3 10.10.20.1
ping -c 3 8.8.8.8
ping -c 3 10.10.60.10
ping -c 3 10.10.60.1
nc -zvw 5 10.10.60.10 22
nc -zvw 5 10.10.60.10 80
curl --max-time 5 http://10.10.60.10
nmap -sS -p 22,80 10.10.60.10
```

---

## Resultado final del Sprint 3

El Sprint 3 queda completado con los siguientes resultados:

- `srv-it` configurado en Z2 con IP `10.10.20.10`.
- `srv-it` llega a su gateway `10.10.20.1`.
- `srv-it` tiene salida a Internet.
- `srv-it` resuelve DNS correctamente tras corregir `/etc/resolv.conf`.
- `srv-ot` configurado en Z6 con IP `10.10.60.10`.
- `srv-ot` tiene instalados y activos `nginx` y `ssh`.
- `srv-ot` escucha en los puertos `22` y `80`.
- La política de firewall bloquea el tráfico desde Z2 hacia Z6.
- Los errores detectados han sido documentados y corregidos.

---

## Conclusión

Durante este sprint se desplegaron las dos VMs Debian principales de la maqueta y se validó la segmentación entre la zona corporativa y la zona OT. Los problemas encontrados estuvieron relacionados principalmente con DNS, reglas de firewall y conectividad temporal para instalar paquetes en la red OT.

La configuración final permite continuar con los siguientes casos de prueba, especialmente el TC-01, donde se demostrará que el tráfico desde Z2 hacia Z6 queda bloqueado y registrado correctamente por pfSense.
