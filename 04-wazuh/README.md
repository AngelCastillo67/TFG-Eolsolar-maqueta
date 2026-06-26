# Sprint 4 — Despliegue de Wazuh, agentes y recepción de eventos de pfSense

## 4.1 Objetivo del sprint

El objetivo de este sprint es desplegar una plataforma Wazuh en la red Z2, integrar agentes en las máquinas `srv-it` y `srv-ot`, habilitar la monitorización de integridad de ficheros para detectar cambios en `/etc/hosts`, y configurar pfSense para enviar eventos de firewall a Wazuh mediante syslog.

Con ello se pretende validar los siguientes puntos:

- Despliegue funcional de Wazuh mediante Docker.
- Acceso al panel web de Wazuh.
- Registro de agentes Linux.
- Detección de modificaciones en ficheros críticos.
- Recepción de logs de pfSense.
- Generación de alertas asociadas a bloqueos de tráfico entre zonas.

---

## 4.2 Ampliación del disco de la VM Wazuh

Se amplió el disco de la VM Wazuh desde Proxmox hasta 40 GB.

Inicialmente, dentro de Debian se comprobó que el disco físico ya aparecía ampliado:

```bash
lsblk
```

Resultado observado:

```text
sda      40G
├─sda1   14.2G /
├─sda2   1K
└─sda5   842M [SWAP]
```

Aunque el disco `/dev/sda` ya tenía 40 GB, la partición raíz `/dev/sda1` seguía teniendo aproximadamente 14 GB. El comando `growpart` no podía ampliar la partición porque existía una partición swap `/dev/sda5` ubicada después de `/dev/sda1`.

El mensaje observado fue:

```text
NOCHANGE: partition 1 could only be grown by 2046 [fudge=2048]
```

Esto indicaba que la partición raíz no podía crecer porque la partición swap estaba bloqueando el espacio libre.

### Solución aplicada

Se desactivó la swap:

```bash
sudo swapoff -a
```

Se eliminaron las particiones de swap y extendida mediante `fdisk`:

```bash
sudo fdisk /dev/sda
```

Dentro de `fdisk`:

```text
p
d
5
d
2
w
```

Después se reinició la máquina:

```bash
sudo reboot
```

Una vez eliminada la partición swap, se amplió la partición raíz:

```bash
sudo growpart /dev/sda 1
sudo resize2fs /dev/sda1
df -h /
```

Finalmente, se creó una nueva swap como archivo:

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

Se añadió la swap permanente en `/etc/fstab`:

```bash
sudo nano /etc/fstab
```

Añadiendo la línea:

```text
/swapfile none swap sw 0 0
```

Se verificó con:

```bash
free -h
swapon --show
```

También se eliminó el aviso de `suspend/resume device` configurando:

```bash
echo "RESUME=none" | sudo tee /etc/initramfs-tools/conf.d/resume
sudo update-initramfs -u
```

---

## 4.3 Ajuste de memoria de la VM Wazuh

Durante el arranque de Wazuh se detectó que el contenedor `wazuh.indexer` era detenido por falta de memoria.

El síntoma principal fue que el panel web devolvía:

```text
HTTP/1.1 503 Service Unavailable
```

En los logs del indexer apareció:

```text
Out of memory: Killed process ... java
```

También se verificó con:

```bash
sudo dmesg -T | grep -iE "out of memory|killed process" | tail -20
```

Resultado observado:

```text
Memory cgroup out of memory: Killed process ... java
```

### Solución aplicada

Se aumentó la memoria RAM de la VM Wazuh desde Proxmox.

Configuración recomendada aplicada:

```text
RAM: 6 GB
CPU: 2 cores
Disco: 40 GB
```

Además, se ajustó el perfil reducido del indexer en `docker-compose.yml`.

En el servicio `wazuh.indexer` se dejó la configuración:

```yaml
environment:
  - "OPENSEARCH_JAVA_OPTS=-Xms1g -Xmx1g"
mem_limit: 3g
```

Después se reinició el entorno Docker:

```bash
cd ~/wazuh-docker/single-node
docker compose down
docker compose up -d
```

Se verificó que el indexer respondía correctamente:

```bash
curl -k -u admin:SecretPassword https://127.0.0.1:9200
```

Resultado esperado:

```json
{
  "name" : "wazuh.indexer",
  "cluster_name" : "opensearch",
  "version" : {
    "number" : "7.10.2"
  }
}
```

También se comprobó el dashboard:

```bash
curl -k -I https://localhost
```

Resultado correcto:

```text
HTTP/1.1 302 Found
location: /app/login?
```

Con esto quedó resuelto el error `503 Service Unavailable`.

---

## 4.4 Instalación de Docker y Docker Compose

Se instaló Docker desde el repositorio oficial.

Comandos utilizados:

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
```

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

```bash
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

```bash
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Se activó Docker al arranque:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Se añadió el usuario al grupo Docker:

```bash
sudo usermod -aG docker $USER
```

Posteriormente se cerró sesión y se volvió a entrar.

Verificación:

```bash
docker --version
docker compose version
```

---

## 4.5 Despliegue de Wazuh con Docker

Se clonó el repositorio oficial de Wazuh Docker:

```bash
cd
git clone https://github.com/wazuh/wazuh-docker.git -b v4.7.5
cd wazuh-docker/single-node
```

Se modificó el archivo:

```bash
nano docker-compose.yml
```

En el servicio `wazuh.indexer` se aplicó el perfil reducido:

```yaml
wazuh.indexer:
  image: wazuh/wazuh-indexer:4.7.5
  hostname: wazuh.indexer
  restart: always
  ports:
    - "9200:9200"
  environment:
    - "OPENSEARCH_JAVA_OPTS=-Xms1g -Xmx1g"
  mem_limit: 3g
```

Se levantaron los contenedores:

```bash
docker compose up -d
```

Se comprobó el estado:

```bash
docker compose ps
```

Resultado esperado:

```text
wazuh.manager     Up
wazuh.indexer     Up
wazuh.dashboard   Up
```

---

## 4.6 Acceso al panel web de Wazuh

El panel web de Wazuh quedó disponible en:

```text
https://10.10.20.20
```

Desde el equipo anfitrión se publicó temporalmente mediante pfSense usando NAT:

```text
WAN:8443  →  10.10.20.20:443
```

Acceso desde navegador:

```text
https://IP_WAN_PFSENSE:8443
```

Se aceptó el certificado autofirmado y se accedió con las credenciales iniciales:

```text
Usuario: admin
Contraseña: SecretPassword
```

Tras validar el acceso, se recomienda cambiar la contraseña por defecto.

---

## 4.7 Instalación del agente Wazuh en srv-it

En `srv-it` se descargó el paquete del agente:

```bash
cd
rm -f wazuh-agent*.deb
wget -O wazuh-agent.deb https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.7.5-1_amd64.deb
```

Se instaló indicando la IP del manager y el nombre del agente:

```bash
sudo WAZUH_MANAGER='10.10.20.20' WAZUH_AGENT_NAME='srv-it' dpkg -i ./wazuh-agent.deb
```

Se activó y arrancó el servicio:

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

Verificación:

```bash
sudo systemctl status wazuh-agent --no-pager
```

Resultado esperado:

```text
active (running)
```

También se verificó la configuración del manager:

```bash
sudo grep -A5 "<server>" /var/ossec/etc/ossec.conf
```

Debe aparecer:

```xml
<address>10.10.20.20</address>
```

En el panel de Wazuh, el agente `srv-it` apareció como activo.

---

## 4.8 Instalación del agente Wazuh en srv-ot

La máquina `srv-ot` se encuentra en Z6 y no tiene salida a Internet. Por tanto, no puede descargar directamente el agente desde `packages.wazuh.com`.

Se optó por el método de cambio temporal de bridge en Proxmox.

### Procedimiento

1. Apagar `srv-ot`:

```bash
sudo shutdown -h now
```

2. En Proxmox, cambiar temporalmente el bridge de red de `srv-ot`:

```text
vmbr2 → vmbr1
```

3. Arrancar `srv-ot`.

4. Configurar IP temporal de Z2, ya que no estaba disponible `dhclient`:

```bash
sudo ip addr add 10.10.20.50/24 dev ens18
sudo ip link set ens18 up
sudo ip route add default via 10.10.20.1
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf
```

5. Verificar conectividad:

```bash
ping -c 3 8.8.8.8
ping -c 3 packages.wazuh.com
```

6. Descargar e instalar el agente:

```bash
cd
rm -f wazuh-agent*.deb
wget -O wazuh-agent.deb https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.7.5-1_amd64.deb
```

```bash
sudo WAZUH_MANAGER='10.10.20.20' WAZUH_AGENT_NAME='srv-ot' dpkg -i ./wazuh-agent.deb
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
sudo systemctl status wazuh-agent --no-pager
```

7. Apagar `srv-ot`:

```bash
sudo shutdown -h now
```

8. En Proxmox, devolver el bridge a Z6:

```text
vmbr1 → vmbr2
```

9. Verificar IP de Z6:

```bash
ip -4 addr show ens18
ip route
```

Debe aparecer:

```text
10.10.60.10
```

---

## 4.9 Regla pfSense para permitir comunicación srv-ot → Wazuh

Para que el agente de `srv-ot` pueda comunicarse con el manager Wazuh, se creó una regla en pfSense en la interfaz `Z6_OT`.

Configuración de la regla:

```text
Action: Pass
Interface: Z6_OT
Address Family: IPv4
Protocol: TCP

Source:
  Type: Address or Alias
  Address: 10.10.60.10
  Mask: /32

Destination:
  Type: Address or Alias
  Address: 10.10.20.20
  Mask: /32

Destination Port Range:
  From: 1514
  To: 1515

Description:
  Allow srv-ot to Wazuh Manager 1514-1515
```

La regla debe estar situada por encima de la regla general de bloqueo de Z6.

Prueba desde `srv-ot`:

```bash
nc -zv -w 5 10.10.20.20 1514
nc -zv -w 5 10.10.20.20 1515
```

---

## 4.10 Configuración centralizada para monitorizar /etc/hosts

Para el caso TC-02 se configuró Wazuh para detectar modificaciones en `/etc/hosts` de los agentes.

Es importante no confundir:

```text
Manager → Configuration → Edit configuration
```

con la configuración centralizada.

La configuración centralizada se encuentra en:

```text
Wazuh → Management → Groups → default → agent.conf
```

En `agent.conf` se añadió:

```xml
<agent_config>
  <syscheck>
    <disabled>no</disabled>
    <frequency>300</frequency>
    <scan_on_start>yes</scan_on_start>
    <alert_new_files>yes</alert_new_files>

    <directories realtime="yes" check_all="yes">/etc/hosts</directories>
    <directories realtime="yes" check_all="yes">/etc</directories>
    <directories realtime="yes" check_all="yes">/usr/bin</directories>
    <directories realtime="yes" check_all="yes">/usr/sbin</directories>

    <ignore>/etc/random-seed</ignore>
    <ignore>/etc/random.seed</ignore>
    <ignore>/etc/adjtime</ignore>
    <ignore>/etc/mtab</ignore>

    <skip_nfs>yes</skip_nfs>
    <skip_dev>yes</skip_dev>
    <skip_proc>yes</skip_proc>
    <skip_sys>yes</skip_sys>
  </syscheck>
</agent_config>
```

Después se reinició el agente en `srv-it`:

```bash
sudo systemctl restart wazuh-agent
```

Prueba de modificación:

```bash
echo "# prueba TC-02 Wazuh $(date)" | sudo tee -a /etc/hosts
```

La alerta debe aparecer en:

```text
Wazuh → File Integrity Monitoring
```

o en:

```text
Modules → Security events
```

---

## 4.11 Configuración de pfSense para enviar syslog a Wazuh

Para que los bloqueos del TC-01 queden registrados en Wazuh, se configuró pfSense para enviar logs remotos por syslog.

Ruta en pfSense:

```text
Status → System Logs → Settings
```

Configuración aplicada:

```text
Enable Remote Logging: marcado
Source Address: Z2_USERS
IP Protocol: IPv4
Remote log servers: 10.10.20.20:514
Remote Syslog Contents:
  System Events: marcado
  Firewall Events: marcado
```

Importante: en `Remote log servers` se debe introducir únicamente:

```text
10.10.20.20:514
```

en el primer campo. Los otros campos deben quedar vacíos.

---

## 4.12 Configuración de Wazuh Manager para recibir syslog

En el panel de Wazuh se editó la configuración del manager:

```text
Wazuh → Management → Configuration → Edit configuration
```

Dentro de `<ossec_config>`, junto al bloque `<remote>` existente para agentes, se añadió un nuevo bloque para syslog:

```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>10.10.20.1/32</allowed-ips>
</remote>
```

Quedando junto al bloque seguro de agentes:

```xml
<remote>
  <connection>secure</connection>
  <port>1514</port>
  <protocol>tcp</protocol>
  <queue_size>131072</queue_size>
</remote>

<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>10.10.20.1/32</allowed-ips>
</remote>
```

Se guardó la configuración y se reinició el manager:

```bash
cd ~/wazuh-docker/single-node
docker compose restart wazuh.manager
```

Verificación de logs del manager:

```bash
docker compose logs --tail=200 wazuh.manager | grep -iE "remote|syslog|514"
```

Resultado correcto observado:

```text
Remote syslog allowed from: '10.10.20.1/32'
Listening on port 514/UDP (syslog)
```

También se verificó que Docker publicaba el puerto UDP 514:

```bash
docker compose ps
```

Resultado esperado:

```text
0.0.0.0:514->514/udp
```

---

## 4.13 Verificación de recepción de logs pfSense en Wazuh

Se activó temporalmente el log completo en Wazuh para comprobar la recepción de los eventos.

En `<global>` se cambió:

```xml
<logall>no</logall>
<logall_json>no</logall_json>
```

por:

```xml
<logall>yes</logall>
<logall_json>yes</logall_json>
```

Se reinició el manager:

```bash
docker compose restart wazuh.manager
```

Se visualizaron los logs crudos:

```bash
docker compose exec wazuh.manager tail -f /var/ossec/logs/archives/archives.json
```

Desde `srv-it` se generó tráfico bloqueado hacia `srv-ot`:

```bash
ping -c 5 10.10.60.10
```

En `archives.json` se observaron entradas de pfSense:

```text
filterlog
match,block
10.10.20.10
10.10.60.10
```

Esto confirmó que:

```text
pfSense genera logs: SÍ
pfSense envía syslog a Wazuh: SÍ
Wazuh recibe los logs: SÍ
```

---

## 4.14 Creación de regla local para alertas de pfSense

Para que los logs recibidos por syslog aparezcan como eventos en `Security events`, se creó una regla local en Wazuh.

Ruta:

```text
Wazuh → Management → Rules → Custom rules → local_rules.xml
```

Regla añadida:

```xml
<group name="local,pfsense,firewall,tc01,">
  <rule id="100200" level="8">
    <match>filterlog</match>
    <description>pfSense firewall log recibido por syslog</description>
  </rule>
</group>
```

Después se reinició el manager:

```bash
cd ~/wazuh-docker/single-node
docker compose restart wazuh.manager
```

Se generó de nuevo tráfico bloqueado desde `srv-it` hacia `srv-ot`:

```bash
ping -c 5 10.10.60.10
```

En Wazuh se comprobó la aparición de alertas en:

```text
Modules → Security events
```

Con los siguientes datos:

```text
Description: pfSense firewall log recibido por syslog
Rule ID: 100200
Level: 8
Agent name: wazuh.manager
```

Esto confirma que Wazuh recibe y procesa correctamente los logs de pfSense.

---

## 4.15 Evidencias recomendadas

Para documentar el sprint se recomienda guardar capturas de:

1. Estado de contenedores Docker:

```bash
docker compose ps
```

2. Respuesta del indexer:

```bash
curl -k -u admin:SecretPassword https://127.0.0.1:9200
```

3. Respuesta del dashboard:

```bash
curl -k -I https://localhost
```

4. Agente `srv-it` activo en Wazuh.

5. Agente `srv-ot` activo en Wazuh.

6. pfSense mostrando logs de firewall:

```text
Status → System Logs → Firewall
```

7. Regla de bloqueo:

```text
DENEGACIÓN Z2 → Z6 (TC-01)
```

8. Wazuh mostrando eventos de pfSense:

```text
Modules → Security events
Rule ID: 100200
Description: pfSense firewall log recibido por syslog
```

9. Archivo `archives.json` mostrando logs crudos con:

```text
filterlog
match,block
10.10.20.10
10.10.60.10
```

---

## 4.16 Problemas encontrados y solución aplicada

### Problema 1 — `growpart` no ampliaba `/dev/sda1`

Causa:

```text
La partición swap /dev/sda5 estaba situada después de /dev/sda1.
```

Solución:

```text
Eliminar la swap antigua, ampliar /dev/sda1 y crear una nueva swapfile.
```

---

### Problema 2 — Wazuh Dashboard devolvía 503

Causa:

```text
El contenedor wazuh.indexer era detenido por falta de memoria.
```

Evidencia:

```text
Out of memory: Killed process ... java
```

Solución:

```text
Aumentar RAM de la VM y configurar OPENSEARCH_JAVA_OPTS=-Xms1g -Xmx1g con mem_limit: 3g.
```

---

### Problema 3 — Firefox no abría en srv-it

Error:

```text
Error: no DISPLAY environment variable specified
```

Causa:

```text
srv-it no tenía entorno gráfico.
```

Solución:

```text
Acceder desde el equipo anfitrión mediante NAT/port forward en pfSense.
```

---

### Problema 4 — SSH en srv-it no arrancaba

Error:

```text
sshd: no hostkeys available -- exiting
```

Solución:

```bash
sudo ssh-keygen -A
sudo systemctl restart ssh
sudo systemctl status ssh --no-pager
```

---

### Problema 5 — Error al instalar agente Wazuh por URL incorrecta

Error:

```text
403 Forbidden
```

Causa:

```text
Ruta incorrecta del paquete.
```

URL correcta:

```text
https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.7.5-1_amd64.deb
```

---

### Problema 6 — Error con variable WAZUH_MANAGER

Error:

```text
sudo: 10.10.20.20: command not found
```

Causa:

```text
Se dejó un espacio después de WAZUH_MANAGER=
```

Incorrecto:

```bash
sudo WAZUH_MANAGER= '10.10.20.20'
```

Correcto:

```bash
sudo WAZUH_MANAGER='10.10.20.20' WAZUH_AGENT_NAME='srv-it' dpkg -i ./wazuh-agent.deb
```

---

### Problema 7 — srv-ot no tenía Internet

Causa:

```text
srv-ot estaba en Z6, una red bloqueada sin salida a Internet.
```

Solución:

```text
Cambiar temporalmente el bridge a Z2, instalar el agente y devolverlo después a Z6.
```

---

### Problema 8 — Wazuh no recibía syslog al principio

Causa:

```text
Wazuh Manager no estaba configurado para escuchar en UDP 514.
```

Solución:

```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>10.10.20.1/32</allowed-ips>
</remote>
```

---

### Problema 9 — Los logs llegaban a archives.json pero no a Security events

Causa:

```text
Wazuh recibía logs crudos, pero no existía una regla local que los elevara a alerta.
```

Solución:

```xml
<group name="local,pfsense,firewall,tc01,">
  <rule id="100200" level="8">
    <match>filterlog</match>
    <description>pfSense firewall log recibido por syslog</description>
  </rule>
</group>
```

---

## 4.17 Estado final

Al finalizar el sprint se consiguió:

- VM Wazuh con disco ampliado a 40 GB.
- Swap configurada mediante `/swapfile`.
- Wazuh desplegado mediante Docker.
- Wazuh Indexer funcionando sin errores de memoria.
- Dashboard accesible vía HTTPS.
- Agente `srv-it` instalado y activo.
- Agente `srv-ot` instalado.
- Comunicación permitida desde `srv-ot` hacia Wazuh por los puertos 1514 y 1515.
- Configuración centralizada de FIM para detectar cambios en `/etc/hosts`.
- pfSense enviando logs remotos a Wazuh por UDP 514.
- Wazuh recibiendo logs de pfSense en `archives.json`.
- Wazuh generando alertas visibles en `Security events` mediante la regla local `100200`.

---

## 4.18 Recomendación final

Tras finalizar las pruebas y guardar las evidencias, se recomienda desactivar el log completo para evitar consumo excesivo de disco.

En la configuración del manager, volver a dejar:

```xml
<logall>no</logall>
<logall_json>no</logall_json>
```

Después reiniciar:

```bash
cd ~/wazuh-docker/single-node
docker compose restart wazuh.manager
```

Con esto se mantiene la generación de alertas, pero se evita almacenar todos los logs crudos en `archives.json`.
