````markdown
# Prueba de Concepto: Servicio Web de Alta Disponibilidad -- por v0id0100

![main_image](images/main_image.png)


## Índice:
- [1. ¿Qué es Alta Disponibilidad?](#1-que-es-alta-disponibilidad)
- [2. Arquitectura del Proyecto](#2-arquitectura-del-proyecto)
- [3. Herramientas que vamos a usar](#3-herramientas-que-vamos-a-usar)
- [4. Ejemplo de Desarrollo Web](#4-ejemplo-de-desarrollo-web)
- [5. Sincronización de Datos entre nodos](#5-sincronizacion-de-datos-entre-nodos)
- [6. Implementación del Balanceador de Carga](#6-implementacion-del-balanceador-de-carga)
- [7. Sincronización / Comprobación de estado entre nodos](#7-sincronizacion--comprobacion-de-estado-entre-nodos)
- [8. Seguridad y Firewall](#8-seguridad-y-firewall)
- [9. Prueba de Concepto](#9-prueba-de-concepto)
- [10. Análisis de Mejoras y Puntos de Falla](#10-analisis-de-mejoras-y-puntos-de-falla)
- [11. ¿Quién soy?](#11-quien-soy)
- [12. Agradecimientos a](#12-agradecimientos-a)

---

## 1. ¿Qué es Alta Disponibilidad?
Alta Disponibilidad es la definición de que un **servidor o servicio sea accesible y operativo el 100% del tiempo**, o al menos minimizando los periodos de inactividad.

Se usa mucho hoy en día en redes sociales, bancos y otro tipo de aplicaciones.

### ¿Por qué esta PoC es Alta Disponibilidad?
En este proyecto voy a implementar un clúster web, un conjunto de nodos que trabajan juntos; el cliente solo ve un portal web, pero detrás hay 2 nodos (solo para probar el concepto) trabajando y sirviendo el portal al mismo tiempo.

- Intentando eliminar los **Puntos Únicos de Falla (Single Points of Failure)**, que son los puntos que si uno falla, causaría la caída de todo el sistema.

- Implementando **failover**: si uno de los nodos falla, el otro se activará inmediatamente sin que el cliente lo perciba porque el DNS (o la IP en este caso) no cambia.

- Para esto voy a usar una herramienta llamada ***conntrackd*** que lo que hace es comprobar continuamente si su *pareja* sigue viva.

---

## 2. Arquitectura del Proyecto:
| ROL | NOMBRE HOST | IP | Función|
|-----|-------------|----|--------|
| FW/LB Master | Lb01 | 192.168.1.10 | Nodo Activo, gestiona la VIP |
| FW/LB Backup | Lb02 | 192.168.1.11 | Nodo Pasivo, sincroniza la funcionalidad |
| Servidor Web | Web01 | 192.168.1.21 | Servicio Web Apache |
| Servidor Web | Web01 | 192.168.1.22 | Servicio Web Apache |
| IP Virtual (VIP) | - - - - - - - - - - - - | 192.168.1.100 | IP flexible, punto de entrada |

---

## 3. Herramientas que vamos a usar:
En esta PoC necesitamos algunos requisitos antes de proceder:
- Para el servicio web en los nodos **Web01, Web02**:
    ```bash
    sudo apt install apache2 php -y
    ```
- Para sincronizar datos entre nodos en **Lb01, Lb02**:
    ```bash
    # En lb02 (Backup)
    sudo apt install openssh-server
    # En lb01 (Master)
    sudo apt install openssh-client
    # En ambos: Lb01, Lb02
    sudo apt install rsync
    ```
- Para balancear la carga entre nodos:
    ```bash
    # En lb01 (Master)
    sudo apt install haproxy
    ```
- Para controlar la dirección VIP:
    ```bash
    # En ambos: Lb01, Lb02
    sudo apt install keepalived -y
    ```
- Para comprobar el estado entre nodos:
    ```bash
    # En ambos: Lb01, Lb02
    sudo apt install conntrackd -y
    ```
- Para programar tareas:
    - `crontab`
- Como en todo desarrollo web necesitas un firewall, **EN TODOS LOS NODOS**:
    ```bash
    sudo apt install ufw -y
    ```

---

## 4. Ejemplo de Desarrollo Web:
Voy a poner un código HTML y PHP sencillo para ver la web en producción, **AQUÍ PUEDES PONER TU PROPIO CÓDIGO**:
**En Lb01 y Lb02**:
```html
<h1>Bienvenido a mi servicio web de alta disponibilidad</h1>
<footer>
    Servido por: <?php echo gethostname(); ?> <br>
    IP: <?php echo $_SERVER['SERVER_ADDR']; ?>
</footer>
```

Ahora tienes que reiniciar el servidor para aplicar el HTML en la web de producción:
```bash
sudo sytemctl restart apache2
# Siempre es recomendable comprobar si todo está OK:
sudo systemctl status apache
```
Debe mostrarse:
![image.png](images/image.png)

---

## 5. Sincronización de Datos entre nodos:
Primero debemos tener un **par de claves (Key Pair)** para iniciar sesión del Master al Backup y así sincronizar datos:
Entonces, en **Web01** generaremos las claves:
```bash
ssh-keygen -t ed25519
# Las opciones siguientes no importan, puedes pulsar enter
```

Ahora las tenemos que enviar al servidor:
```bash
ssh-copy-id Web02@192.168.1.22
# Pon la contraseña de Lb02 y tendrás la clave en /home/username/.ssh/authorizedkeys
```

Ahora en **Web01** crearemos un script para sincronizar automáticamente dos veces al día.
```bash
#!/bin/bash

SOURCE=/var/www/html/documentname.php
DESTINATION="Web02@192.168.1.22:/var/www/html/documentname.php"
LOGFILE="/var/log/rsync_sync.log"

echo "--- Sync started at $(date) ---" >> $LOGFILE
rsync -avz $SOURCE $DESTINATION >> $LOGFILE 2>&1

if [ $? -eq 0 ]; then
    echo "Sync successful." >> $LOGFILE
else
    echo "Sync FAILED. Check network or SSH-keys." >> $LOGFILE
fi
```

Luego hay que dar permiso de ejecución:
```bash
chmod +x script.sh
```

Ahora lo programaremos para ejecutarse 2 veces al día:

Pulsa `crontab -e` y añade al final del archivo:
```text
0 */12 * * * /bin/bash /home/username/script.sh
```

---

## 6. Implementación del Balanceador de Carga:
Voy a usar HAProxy en **Lb01 (Master)** para balancear entre nodos.

- Archivo de configuración (`/etc/haproxy/haproxy.cfg`), (**pon esto al final del archivo**):
    ```text
    frontend http_front
    bind *:80
    default_backend web_backend
    backend web_backend
    balance roundrobin
    server web01 192.168.1.21:80 check
    server web02 192.168.1.22:80 check
    ```
- Para ver tu interfaz usa:
    ```bash
    ip a
    ```
- Una vez hecho esto, debes configurar la dirección **VIP** (`/etc/keepalived/keepalived.conf`):
    ```text
    vrrp_instance VI_1 {
        state MASTER
        interface YOURINTERFACE
        virtual_router_id 51
        priority 100
        advert_int 1
        authentication {
            auth_type PASS
            auth_pass 1111
        }
        virtual_ipaddress {
            192.168.1.100
        }
    }
    ```
- Recarga y verifica el estado:
    ```bash
    sudo systemctl restart keepalived && sudo systemctl status keepalived
    ```

- Cuando terminemos el **Master**, ahora configura el **Slave** o **Backup**:
    ```text
    vrrp_instance VI_1 {
        state SLAVE
        interface YOURINTERFACE
        virtual_router_id 51
        priority 90
        advert_int 1
        authentication {
            auth_type PASS
            auth_pass 1111
        }
        virtual_ipaddress {
            192.168.1.100
        }
    }
    ```
- Reinicia y comprueba el estado.

---

## 7. Sincronización / Comprobación de estado entre nodos:
- **conntrackd** se usa para comprobar cada pocos segundos el estado entre nodos y si uno falla enviar el estado al otro.

- Una vez instalado (ver [paso 3](#3-herramientas-que-vamos-a-usar)) debes añadir al final del archivo (`/etc/conntrackd/conntrackd.conf`):

    ```text
    Sync {
        Mode FTFW {
            ResendQueueSize 1024
            ACKWindowSize 300
        }
        Multicast {
            IPv4_address 225.0.0.50
            Group 3780
            Interface YOURINTERFACE
        }
    }
    ```

- Como siempre recarga y verifica el estado correcto.

---

## 8. Seguridad y Firewall:
Es importante abrir los puertos correctos en cada servidor:

- En el clúster web (Web01, Web02):
    ```bash
    sudo ufw allow 80/tcp
    # Permitiremos ssh SOLO desde nuestra red
    sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp

    # Activar el firewall
    sudo ufw enable
    ```

- En los Load Balancers / Firewalls (Lb01, Lb02):
    ```bash
    sudo ufw allow 80/tcp
    # Permitiremos ssh SOLO desde nuestra red
    sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp

    # Activar el firewall
    sudo ufw enable
    ```

- Debemos permitir las conexiones de **VRRP** (VIP) y **conntrackd**:
    ```bash
    # En Lb01:
    sudo ufw allow from 192.168.1.11 to any port 3780 proto udp
    ```

    ```bash
    # En Lb02:
    sudo ufw allow from 192.168.1.10 to any port 3780 proto udp
    ```

- Finalmente recarga el firewall:
    ```bash
    sudo ufw reload
    ```

---

## 9. Prueba de Concepto:
Primero accedemos al portal de entrada en **192.168.1.100**:
![image2](images/image2.png)

Pruebas de Alta Disponibilidad:
- Ahora haremos un *ping* infinito para probar la disponibilidad y apagaremos el ***Master***; debería fallar un ping pero la VIP debería moverse al ***Slave***:
![images3](images/image3.png)

- Comprobando que el slave tiene la IP .100:
![image4](images/image4.png)

- Verifica los logs en */var/log/conntrackd.log*:
![image5](images/image5.png)

- Probando *failover* entre nodos web:
    - Aquí apagaré Web01 y comprobaré si Web02 responde:
    ![alt text](images/image6.png)

    Responde a través de la misma IP y si comprobamos el texto sabremos que responde el Servidor 2 (*192.168.1.22*)

---

## 10. Análisis de Mejoras y Puntos de Falla:
- Si Web01 falla antes de sincronizar los datos en Web02, todos los cambios realizados en ese periodo se perderán. **Debería sincronizarse cada vez que se actualice código.**
- La configuración de HAProxy solo está en Lb01; si ese Balanceador falla, todo fallará porque es un *punto único de falla*.
- UFW está configurado solo en una dirección; si hay un failover provocado por Lb01, las reglas no se actualizarán automáticamente, esto debería ser bidireccional.

---

## 11. ¿Quién soy?
- Actualmente estoy estudiando ciberseguridad y haciendo unas prácticas en una empresa trabajando con Amazon Web Services.
- En mi tiempo libre me gusta mejorar mis conocimientos de hacking ético, programar herramientas open-source; puedes verlas en mi perfil. Es un mundo asombroso: cuanto más sabes, más te das cuenta de lo mucho que aún te queda por aprender.
- Es mi segunda Prueba de Concepto pero no será la última, estoy bastante seguro de ello.

---

## 12. Agradecimientos a:
- A mis profesores, que me enseñaron cómo funciona IT y a investigar siempre por nueva información y actualizarme.
- A mi novia por apoyarme en todo y tratar de entender cosas que para personas normales son muy enrevesadas.
- Y, por supuesto, a mis padres por apoyarme con los estudios.
- Gracias por leer esta PoC hasta aquí, ¡ahaha!

---

<h1 style="text-align: center;">Por v0id0100</h1>
````
