
# Infraestructura de Servidor de Correo (DNS + Postfix/Dovecot)

## 📖 Descripción General
Este repositorio documenta el despliegue de una arquitectura completa de correo electrónico virtualizada. El sistema se divide en dos nodos principales: un servidor dedicado a la resolución de nombres (DNS) y otro encargado de la transferencia y entrega de correos (MTA/MDA) utilizando la suite Postfix y Dovecot.

## 🛠️ Requisitos Previos
Para desplegar este entorno, es necesario contar con las siguientes herramientas:
* [Vagrant](https://developer.hashicorp.com/vagrant/install) (Gestión de máquinas virtuales)
* [Oracle Virtualbox](https://www.virtualbox.org/wiki/Downloads) (Hipervisor)
* [Mozilla Thunderbird](https://www.thunderbird.net/es-ES/) (Cliente de correo para pruebas)

---

## 🌐 Nodo 1: Servidor DNS
[Configuración de Infraestructura]

Este nodo es el encargado de traducir los nombres de dominio a direcciones IP. Su correcta configuración es vital para que el dominio `example.test` sea reconocible y el tráfico de correo se enrute correctamente.

* 🖥️ **Hostname:** `dns.example.test`
* 🌐 **Dirección IP:** `192.168.57.10`

### Puesta en Marcha
1. Abrir una terminal en el directorio raíz del proyecto.
2. Ejecutar `vagrant up` para iniciar las instancias.
    * El script de aprovisionamiento se encargará de instalar las dependencias automáticamente.
3. **Configuración del Resolver:** Para que el sistema anfitrión localice las máquinas, se debe apuntar al DNS creado (ver nota final sobre Windows).

### Verificación del Servicio
* **Prueba de conectividad (Ping):**
    * Ejecutar: `ping -c 3 srv.example.test`
    * Si el DNS funciona, la respuesta debe provenir de la IP `192.168.57.20`.

### Funcionalidades Activas

| Servicio | Estado |
| :--- | :--- |
| **Resolución de Nombres** | `host srv.example.test` resuelve a `192.168.57.20`. |
| **Registros MX** | El dominio identifica el servidor de correo autorizado. |
| **Interconexión** | Visibilidad completa entre máquinas y anfitrión. |

---

## 📧 Nodo 2: Servidor de Correo
[Configuración del Servicio SMTP/IMAP]

Esta máquina virtual aloja los servicios de correo. Utiliza **Postfix** para el envío (protocolo SMTP) y **Dovecot** para el almacenamiento y acceso a los buzones (protocolos IMAP/POP3) usando el formato *Maildir*.

* 🖥️ **Hostname:** `srv.example.test`
* 🌐 **Dirección IP:** `192.168.57.20`

### Credenciales de Acceso

| Usuario | Password | Email |
| :--- | :---: | :--- |
| **Usuario 1** | `1` | `usuario1@example.test` |
| **Usuario 2** | `2` | `usuario2@example.test` |

### Configuración del Cliente (Thunderbird)

Para validar el funcionamiento, utilizamos Mozilla Thunderbird configurado manualmente con los siguientes parámetros:

1. Ir a **Configuración de cuenta > Añadir cuenta de correo**.
2. Introducir nombre, email y contraseña, y seleccionar "Configurar manualmente".
3. Aplicar la siguiente tabla de conexión:

* **Servidor Entrante (IMAP):** `srv.example.test` | Puerto `143` | SSL: `Ninguna`.
* **Servidor Saliente (SMTP):** `srv.example.test` | Puerto `25` | SSL: `Ninguna`.
* **Autenticación:** Seleccionar `Contraseña normal`.
* **Advertencia de Seguridad:** Al no usar certificados firmados por una CA oficial, aparecerá una alerta. Es necesario marcar "Confirmar excepción de seguridad".

**Estado final:** La cuenta se añade correctamente y muestra la bandeja de entrada.

### Pruebas de Funcionamiento

| Prueba | Resultado |
| :--- | :--- |
| **SMTP (Envío)** | Comunicación exitosa de Usuario 1 hacia Usuario 2. |
| **IMAP (Recepción)** | Usuario 2 visualiza el mensaje entrante. |
| **Persistencia** | Los mensajes se almacenan en `~/Maildir` dentro del servidor. |
| **Seguridad** | Acceso restringido mediante contraseña (`asir`). |

---

## 💻 Guía de Implementación Técnica

A continuación se documentan los comandos exactos ejecutados en las máquinas virtuales para replicar la instalación.

### 1. Despliegue del DNS (`dns`)
Acceso mediante `vagrant ssh dns` y elevación de privilegios con `sudo su`.

**Instalación del paquete:**
```bash
apt update && apt install bind9 -y
```

**Configuracion de zonas**
- Se define la zona maestra en `/etc/bind/named.conf.local`
- Se crea el fichero de registros en `/etc/bind/db.example.test` (A, CNAME, MX).

**Reinicio del demonio:**
```bash
systemctl restart bind9
```

### 2. Despliegue del Mail Server (`srv`)
Acceso mediante `vagrant ssh srv` con privilegios de root.
Instalación de la paquetería:

```bash
apt update && apt install postfix dovecot-imapd dovecot-pop3d -y
```

**🖥️ Asistente de Configuración (Debconf): Durante la instalación de Postfix se seleccionaron las siguientes opciones:**
1. Tipo: `Sitio de Internet`.
2. Nombre del sistema: `example.test`.

Ajustes en Postfix (`main.cf`): Edición del fichero `/etc/postfix/main.cf`:

```bash
nano /etc/postfix/main.cf
```
Se añadieron/modificaron las directivas para Maildir y redes de confianza:

```bash
home_mailbox = Maildir/

mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128 192.168.57.0/24
```
Aplicar cambios en Postfix:

```bash
systemctl restart postfix
```

Ajustes en Dovecot: Configuración en /etc/dovecot/conf.d/ para habilitar la autenticación y localización de buzones:

```bash
mail_location = maildir:~/Maildir

disable_plaintext_auth = no
```
Validación Final: Envío de Correo
Para confirmar el éxito del despliegue, se realizó una prueba de flujo completo:

1. Redacción: `usuario1` envía un email a `usuario2`.
2. Verificación: `usuario2` actualiza su bandeja de entrada.

---

## ⚠️ NOTA IMPORTANTE: Configuración en Windows (Hosts)
Dado que el cliente de correo (Thunderbird) se ejecuta en el sistema anfitrión Windows y no en Linux, fue necesario realizar una configuración adicional para que el sistema pudiera encontrar las máquinas virtuales por su nombre.

Se editó el archivo de sistema Hosts (ubicado en C:\Windows\System32\drivers\etc\hosts) con permisos de administrador, añadiendo las siguientes líneas para forzar la resolución de nombres local:

```bash
192.168.57.20   srv.example.test
192.168.57.20   smtp.example.test
192.168.57.20   imap.example.test
```

