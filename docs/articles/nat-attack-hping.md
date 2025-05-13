# **CONFIGURACIÓN NAT Y ATAQUE CON HPING MIKROTIK**

## Configuración de la topología de red

Se importaron tres routers Mikrotik con la versión 7.11.2, utilizando GNS3 para establecer la conexión y simular su funcionamiento. Conforme a la topología definida, se configuraron los tres routers, cada uno con su respectiva subred de conexión, manejando la interfaz de WAN para salir a internet y una interfaz de LAN para el manejo local de los dispositivo a utilizar en este laboratorio.

![img][image1]

### Identidad de cada router

Se configuró la identidad para cada router y poder manejarlos de una manera más fácil.

**Router 1**  
![img][image2]

**Router 2**  
![img][image3]

**Router 3**  
![img][image4]

### Configuración de red NAT

Utilizando la configuración de red NAT para conectarnos con Winbox de la aplicación local donde se está administrando la conexión con el router R1.  
![img][image5]

Configuración de la Red LAN de la red para proporcionar NAT a la conexión de la red y poder acceder a las diferentes configuraciones de proveer internet a los dos routers y máquina de windows 10 para proveer su configuración.

![img][image6]

Configuramos la dhcp para proveer una IP a la red LAN que se creó para los dos routers.

Se crea una dirección IP para la interfaz de red ether2-LAN para proveer de una dirección de red válida.

![][image7]

Para configurar el NAT y tener acceso a internet a traves de los dos routers conectados a la interfaz LAN del R1, creamo la siguiente teconfiguracion de NAT con **masquerade.**  
![][image8]

Regla de firewall para internet con sus diferentes DNS de salida a internet.  
![][image9]

### Configuración de DHCP en el router

![img][image10]

**Prueba de conectividad dentro del rango configurado**  
Al configurar el DHCP al servidor y las reglas de NAT, observamos que ya no genera una dirección de red dentro del rango configurado y con las reglas de firewall pertinentes de mikrotik.  
![img][image11]

Traza ICMP desde el router principal hacia la máquina host configurada.  
![img][image12]

Y me guarda en la tabla de ARP las diferentes configuraciones de la máquina conectada.  
![img][image13]

Y de acuerdo a esas configuraciones tenemos una asignación por DHCP a los routers R2 y R3, con su respectiva IP.  
![img][image14]

![img][image15]

### Investigando tráfico en tiempo real con TORCH

Se va a realizar el análisis de tráfico dentro de la red LAN para el tráfico entre los diferentes routers conectados a la LAN.  
![img][image16]

**Tráfico desde R2 y R3**  
Se realiza un tráfico hacia internet desde el R3, y analizamos el tráfico que se genera con TORCH.

```bash
[admin@R2] > ping 1.1.1.1
[admin@R3] > ping 8.8.8.8
```

![img][image17]

Resultado se logra ver el tráfico que se genera con estos paquetes enviados desde R2 y R3 hacia internet.  
![img][image18]

Y se puede observar el diferente tráfico que se genera y mirando los anchos de banda que se van generando por cada petición.

![img][image19]

### Tráfico capturado con Wireshark

Para este método se realiza la captura en la interfaz de red de LAN con el siguiente comando para capturar una traza ICMP que se envía desde R3.

```bash
[admin@R3] > ping 8.8.8.8
```

![img][image20]

En la topología se colocó a escuchar en el punto de unión entre el switch y el R1, donde está nuestro punto de control con mikrotik como puerta de enlace hacia la WAN.

![img][image21]

Paquete capturado con wireshark  
![img][image22]

## Ataque a la red utilizando hping3

La función de **hping3** viene preinstalada en las herramientas de Kali Linux, con lo cual vamos hacer uso de ella para esta actividad.  
![img][image23]

### Simulación de tráfico y pruebas de conectividad

Con el siguiente comando podemos ver nuestra conectividad y poder conocer las maquinas que estan conectadas en nuestra red

```bash
sudo arp-scan -I eth0 --localnet
```

Se puede observar con el escaneo las IP de los routers R1, R2, R3, ya tenemos las IPs y ahora procedemos a realizar algunas pruebas de conectividad.  
![img][image24]

* **Utilizando ping hacia los routers e internet**

Enviamos una traza ICMP a los diferentes router e internet para poder observar si tenemos conectividad.

![img][image25]

![img][image26]

![img][image27]

![img][image28]

* **Utilizando traceroute  hacia los routers e internet**

Los paquetes enviados hacia internet realizan la búsqueda del camino hasta llegar al dominio de google, con lo que se puede observar que hay conectividad.

![img][image29]

El traceroute realizada a los routers se puede observar que es directo ya que está conectado directamente a la red LAN donde se encuentra conectados y la ruta es más directa y rápida de encontrar.

![img][image30]

* **Utilizando hping3 para escaner algunos dominios**

Este comando realizará un traceroute hacia `www.google.com` utilizando paquetes ICMP en lugar de los paquetes UDP que utiliza por defecto el comando traceroute. El comando mostrará la lista de routers por los que pasan los paquetes en su camino hacia `www.google.com`, y dado que se usa -V, se dará información detallada sobre cada salto.

```bash
sudo hping3 --traceroute -V -1 www.google.com
```

![img][image31]

En el siguiente comando se realiza un escaneo SYN de puertos en el rango 0 al 50 sobre la dirección R1=172.16.0.1. Está diseñado para verificar qué puertos están abiertos en el host de destino. Al usar el flag SYN (-S), hping3 simula el primer paso del proceso de conexión TCP (el handshake), pero no completa la conexión. Si un puerto está abierto, el host responderá con un paquete SYN-ACK; si está cerrado, el host responderá con un paquete RST.

```bash
sudo hping3 -S --scan 0-50 172.16.0.1 -c 1
```

![img][image32]

### Ataques simulados con hping3

Vamos a realizar un ataque de inundación de peticiones hacia los routers para ver su funcionamiento y cómo se comportan.

#### Realizando ataque con hping3 al R2

**IP Spoof**
Al lanzar el comando realizando un IP Spoof a la dirección de R2, con un trafico con DNS 1.1.1.1 enviando diferentes paquetes, y el trafico se ve reflejado en la interfaz de wireshark como el tráfico se envía con varios paquetes de ASK.

```bash
sudo hping3 172.16.0.253 -t 128 -a 1.1.1.1
```

Captura de tráfico con wireshark

![img][image33]

#### Realizando ataque con hping3 al R3

**IP Spoof**  
Realizamos el mismo procedimiento hacia R3 y observamos el tráfico.

```bash
sudo hping3 172.16.0.254 -t 128 -a 1.1.1.1
```

![img][image34]

![img][image35]

Cuando se realiza el ataque enviando más con hping3 con 10000 paques se observa que la carga del sistema del router R1 comienza a elevar el consumo de recursos.

```bash
sudo hping3 -I eth0 -c 10000 -S --faster 172.16.0.1
```

![img][image36]

En la captura de wireshark se observa las peticiones que se hacen al router.

![img][image37]

Y cada vez que se realicen más peticiones el sistema de mikrotik, se observa un consumo de los procesos de la CPU del sistema, la capturas de consumo se realizan dentro de winbox para el router R1 de mikrotik.

![img][image38]

### Mitigación

**Reglas de firewall**  
En las siguientes reglas de firewall están diseñadas para proteger contra ataques de SYN flood, un tipo de ataque de Denegación de Servicio (DoS) en el que el atacante envía un gran número de solicitudes TCP SYN (inicialización de conexión) con el fin de agotar los recursos del servidor.

Este comando comando redirige las nuevas conexiones TCP SYN a la cadena SYN-Protect.

```bash
/ip firewall filter add chain=input protocol=tcp\ tcp-flags=syn connection-state=new\ action=jump jump-target=SYN-Protect\ comment="Flood protect" disabled=no
```

Este comando permite un número limitado de conexiones nuevas por segundo (400), para mitigar ataques SYN flood.

```bash
/ip firewall filter add chain=SYN-Protect protocol=tcp\ tcp-flags=syn limit=400,5\ connection-state=new action=accept\ comment="" disabled=no
```

Este comando descarta cualquier conexión SYN nueva que exceda el límite, bloqueando posibles ataques que intenten sobrecargar el sistema.

```bash
/ip firewall filter add chain=SYN-Protect protocol=tcp\ tcp-flags=syn connection-state=new \ action=drop comment="Si se supera el límite de la segunda regla, será bloqueado." disabled=no
```

Se ejecutan en los tres router para mitigar este ataque.  
![img][image39]

**Reglas de firewall configuradas**  
Estas reglas ya configuradas ayudan a mitigar el tráfico dentro de la red, disminuyendo los ataques que se realicen.

![img][image40]

## Recomendaciones

Filtrado de paquetes en los routers: Utiliza Access Control Lists (ACLs) en los routers para bloquear tráfico no autorizado, específicamente los paquetes que no son necesarios para las funciones de la red. Esto puede evitar escaneos de puertos y ataques de rastreo.

Habilita un firewall de inspección profunda (Deep Packet Inspection, DPI) en el router para monitorear el contenido de los paquetes y detectar comportamientos anómalos o sospechosos, como intentos de escaneo de puertos.

Prevenir ataques de escaneo de puertos como TCP SYN Cookies, donde se configure en el router o firewall la opción de habilitar la función de SYN cookies para prevenir ataques de tipo SYN flood, en los que un atacante intenta llenar la tabla de conexiones a través del envío masivo de solicitudes SYN.

Implementar VLANs (Virtual LANs) para segmentar el tráfico dentro de la red. Esto aislará diferentes tipos de dispositivos y reducirá la superficie de ataque. Los ataques como el escaneo de puertos serán limitados a la VLAN en la que se encuentre el atacante.

<!-- ---------------------------------------- -->
<!-- Seccion de imagenes -->  
[image1]: ./img/nat-hping/img1.png
[image2]: ./img/nat-hping/img2.png
[image3]: ./img/nat-hping/img3.png
[image4]: ./img/nat-hping/img4.png
[image5]: ./img/nat-hping/img5.png
[image6]: ./img/nat-hping/img6.png
[image7]: ./img/nat-hping/img7.png
[image8]: ./img/nat-hping/img8.png
[image9]: ./img/nat-hping/img9.png
[image10]: ./img/nat-hping/img10.png
[image11]: ./img/nat-hping/img11.png
[image12]: ./img/nat-hping/img12.png
[image13]: ./img/nat-hping/img13.png
[image14]: ./img/nat-hping/img14.png
[image15]: ./img/nat-hping/img15.png
[image16]: ./img/nat-hping/img16.png
[image17]: ./img/nat-hping/img17.png
[image18]: ./img/nat-hping/img18.png
[image19]: ./img/nat-hping/img19.png
[image20]: ./img/nat-hping/img20.png
[image21]: ./img/nat-hping/img21.png
[image22]: ./img/nat-hping/img22.png
[image23]: ./img/nat-hping/img23.png
[image24]: ./img/nat-hping/img24.png
[image25]: ./img/nat-hping/img25.png
[image26]: ./img/nat-hping/img26.png
[image27]: ./img/nat-hping/img27.png
[image28]: ./img/nat-hping/img28.png
[image29]: ./img/nat-hping/img29.png
[image30]: ./img/nat-hping/img30.png
[image31]: ./img/nat-hping/img31.png
[image32]: ./img/nat-hping/img32.png
[image33]: ./img/nat-hping/img33.png
[image34]: ./img/nat-hping/img34.png
[image35]: ./img/nat-hping/img35.png
[image36]: ./img/nat-hping/img36.png
[image37]: ./img/nat-hping/img37.png
[image38]: ./img/nat-hping/img38.png
[image39]: ./img/nat-hping/img39.png
[image40]: ./img/nat-hping/img40.png
