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

