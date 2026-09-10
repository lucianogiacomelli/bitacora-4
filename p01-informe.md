# Informe de Laboratorio N.º 1 - Onboarding
**Grupo:** bitacora-4  
**Integrantes:**
* Luciano Giacomelli
* Fernando Rubiales

---

## 1. Evidencia de Conexión Remota
* **Comando de verificación:** `test -n "$SSH_CONNECTION" && hostname`
* **Salida obtenida:**
```text
srv2
```

---

## 2. Inventario del Servidor (Shell Survival Kit)

### 1. ¿Qué máquina es esta y qué sistema operativo corre?
* **Comando:** `hostnamectl`
* **Salida:**
```text
 Static hostname: srv2
       Icon name: computer-vm
         Chassis: vm 🖴
      Machine ID: f67811bfb45e45089e5a5b76d851f709
         Boot ID: 2c7e30b0cadc473c968bd8741d90c898
  Virtualization: oracle
Operating System: Debian GNU/Linux 12 (bookworm)
          Kernel: Linux 6.1.0-52-amd64
    Architecture: x86-64
 Hardware Vendor: innotek GmbH
  Hardware Model: VirtualBox
Firmware Version: VirtualBox
```

### 2. ¿Qué direcciones de red tiene?
* **Comando:** `ip -br a`
* **Salida:**
```text
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp0s3           UP             10.0.2.15/24 fd17:625c:f037:2:a00:27ff:fe70:6cbf/64 fe80::a00:27ff:fe70:6cbf/64
enp0s8           UP             192.168.100.10/24 fe80::a00:27ff:fe88:dcb8/64
```

### 3. ¿Cuánto disco hay y cuánto queda libre?
* **Comando:** `df -h`
* **Salida:**
```text
S.ficheros     Tamaño Usados  Disp Uso% Montado en
udev             455M      0  455M   0% /dev
tmpfs             97M   568K   96M   1% /run
/dev/sda1         19G   1,9G   16G  11% /
tmpfs            481M      0  481M   0% /dev/shm
tmpfs            5,0M      0  5,0M   0% /run/lock
tmpfs             97M      0   97M   0% /run/user/1000
```

### 4. ¿Cuánta memoria RAM tiene y cuánto está en uso?
* **Comando:** `free -h`
* **Salida:**
```text
               total       usado       libre  compartido   búf/caché   disponible
Mem:           960Mi       225Mi       730Mi       568Ki       138Mi       735Mi
Inter:         974Mi          0B       974Mi
```

### 5. ¿Qué servicios están corriendo?
* **Comando:** `systemctl list-units --type=service --state=running`
* **Salida:**
```text
  UNIT                      LOAD   ACTIVE SUB     DESCRIPTION
  cron.service              loaded active running Regular background program processing daemon
  dbus.service              loaded active running D-Bus System Message Bus
  getty@tty1.service        loaded active running Getty on tty1
  ssh.service               loaded active running OpenBSD Secure Shell server
  systemd-journald.service  loaded active running Journal Service
  systemd-logind.service    loaded active running User Login Management
  systemd-timesyncd.service loaded active running Network Time Synchronization
  systemd-udevd.service     loaded active running Rule-based Manager for Device Events and Files
  user@1000.service         loaded active running User Manager for UID 1000
  wpa_supplicant.service    loaded active running WPA supplicant

LOAD   = Reflects whether the unit definition was properly loaded.
ACTIVE = The high-level unit activation state, i.e. generalization of SUB.
SUB    = The low-level unit activation state, values depend on unit type.
10 loaded units listed.
```

### 6. ¿Quién entró últimamente?
* **Comando:** `last`
* **Salida:**
```text
sysadmin pts/0        192.168.100.100  Thu Sep  3 16:58   still logged in
sysadmin pts/0        192.168.100.100  Thu Sep  3 16:38 - 16:40  (00:02)
sysadmin pts/0        192.168.100.100  Thu Sep  3 16:30 - 16:30  (00:00)
sysadmin tty1                          Thu Sep  3 16:17   still logged in
reboot   system boot  6.1.0-52-amd64   Thu Sep  3 16:16   still running
reboot   system boot  6.1.0-52-amd64   Wed Sep  2 08:19   still running
sysadmin tty1                          Wed Sep  2 08:05 - down   (00:13)
reboot   system boot  6.1.0-52-amd64   Wed Sep  2 08:05 - 08:19  (00:14)
sysadmin tty1                          Wed Sep  2 08:00 - down   (00:03)
reboot   system boot  6.1.0-52-amd64   Wed Sep  2 07:59 - 08:03  (00:03)
sysadmin tty1                          Wed Sep  2 07:56 - down   (00:03)
reboot   system boot  6.1.0-52-amd64   Wed Sep  2 07:54 - 07:59  (00:05)
sysadmin tty1                          Thu Aug 27 21:47 - down   (00:00)
reboot   system boot  6.1.0-52-amd64   Thu Aug 27 21:47 - 21:48  (00:00)
sysadmin tty1                          Thu Aug 27 16:32 - down   (00:07)
reboot   system boot  6.1.0-52-amd64   Thu Aug 27 16:31 - 16:39  (00:07)
sysadmin tty1                          Thu Aug 27 16:30 - down   (00:00)
reboot   system boot  6.1.0-52-amd64   Thu Aug 27 16:30 - 16:31  (00:00)

wtmp empieza Thu Aug 27 16:30:23 2026
```

---

## 3. Preguntas de Navegación

### 7. Existencia de sshd_config y archivo modificado más recientemente en /etc
* **Comandos:**
  * **Verificación de existencia:** `ls -l /etc/ssh/sshd_config`
  * **Archivo más reciente:** `ls -lt /etc | head -n 10`
* **Salida de existencia:**
```text
-rw-r--r-- 1 root root 3223 may  5 07:26 /etc/ssh/sshd_config
```
* **Salida de listado temporal (/etc):**
```text
total 724
-rw-r--r-- 1 root root     25 sep  3 16:17 resolv.conf
-rw-r--r-- 1 root root    201 sep  2 08:00 hosts
-rw-r--r-- 1 root root      5 sep  2 07:57 hostname
-rw-r--r-- 1 root root   3530 ago 27 16:34 mailcap
-rw-r--r-- 1 root root  12039 ago 27 16:34 ld.so.cache
drwxr-xr-x 2 root root   4096 ago 27 16:34 bash_completion.d
drwxr-xr-x 2 root root   4096 ago 27 16:34 alternatives
-rw-r----- 1 root shadow   733 ago 27 16:34 shadow
-rw-r--r-- 1 root root   1283 ago 27 16:34 passwd
```
* **Respuesta:** El archivo modificado más recientemente dentro de `/etc` es `resolv.conf` (con fecha del 3 de septiembre a las 16:17).

### 8. Registro de últimos eventos de SSH
* **Comando:** `sudo journalctl -u ssh -n 10`
* **Salida:**
```text
sep 03 16:37:28 srv2 sshd[601]: pam_unix(sshd:auth): check pass; user unknown
sep 03 16:37:31 srv2 sshd[601]: Failed password for invalid user susadmin from 192.168.100.100 port 38440 ssh2
sep 03 16:37:33 srv2 sshd[601]: Connection closed by invalid user susadmin 192.168.100.100 port 38440 [preauth]
sep 03 16:37:33 srv2 sshd[601]: PAM 1 more authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=192.168.100.100
sep 03 16:38:35 srv2 sshd[611]: Accepted password for sysadmin from 192.168.100.100 port 42122 ssh2
sep 03 16:38:35 srv2 sshd[611]: pam_unix(sshd:session): session opened for user sysadmin(uid=1000) by (uid=0)
sep 03 16:38:35 srv2 sshd[611]: pam_env(sshd:session): deprecated reading of user environment enabled
sep 03 16:58:23 srv2 sshd[695]: Accepted password for sysadmin from 192.168.100.100 port 50904 ssh2
sep 03 16:58:23 srv2 sshd[695]: pam_unix(sshd:session): session opened for user sysadmin(uid=1000) by (uid=0)
sep 03 16:58:23 srv2 sshd[695]: pam_env(sshd:session): deprecated reading of user environment enabled
```
* **Explicación de las líneas registradas:**
  * `Failed password for invalid user susadmin`: Se registró un intento de conexión SSH fallido desde la IP `192.168.100.100` debido a un error en el nombre de usuario (`susadmin` en lugar de `sysadmin`).
  * `Accepted password for sysadmin`: Autenticación correcta del usuario `sysadmin` mediante contraseña desde el cliente (`192.168.100.100`), asignando un puerto de origen efímero para la comunicación (puertos 42122 y 50904).
  * `session opened for user sysadmin(uid=1000)`: El módulo PAM inicializó exitosamente la sesión interactiva del shell para el usuario con UID 1000.

---

## 4. Respaldo de Cierre
* **Comando de creación:** `sudo tar -czvf /home/sysadmin/respaldos/etc-p01.tar.gz -C / etc`
* **Comando de comprobación:** `tar -xzOf /home/sysadmin/respaldos/etc-p01.tar.gz etc/hostname && hostname`
* **Salida de comprobación:**
```text
srv2
srv2
```

---

## 5. Falla Inyectada de Cierre
* **Síntoma observado:** Connection refused al intentar conectar por SSH desde el cliente.
* **Escalón de la escalera con la evidencia decisiva:** Escalón 3 (`systemctl status ssh`).
* **Salida del escalón que reveló el problema:**
```text
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/lib/systemd/system/ssh.service; enabled; preset: enabled)
     Active: inactive (dead)
```
* **Causa identificada:** El servicio `ssh` fue detenido en el servidor.
* **Arreglo aplicado:** `sudo systemctl start ssh`

---

## 6. Preguntas de Cierre

### 1. ¿Por qué el curso prohíbe trabajar en la consola gráfica de VirtualBox si "es lo mismo"? ¿Qué situación real simula esa regla?
* **Respuesta:** Simula la operativa real de administración de servidores en entornos de producción, centros de datos o nubes (AWS, Azure, GCP), donde las máquinas no tienen periféricos de entrada/salida conectados ni entorno de escritorio, obligando a operar de forma remota, segura y automatizable a través de terminales cifradas por SSH.

### 2. De los servicios que encontraste corriendo, elegí dos: ¿qué hacen y qué pasaría si los detenés?
* **Servicio 1 (`ssh.service`):** Gestiona el demonio de OpenSSH para permitir el acceso remoto cifrado. Si se detiene, se pierden las conexiones remotas entrantes y no es posible administrar el servidor a distancia salvo recurriendo a la consola física.
* **Servicio 2 (`systemd-journald.service`):** Es el colector de logs y eventos del sistema operativo. Si se detiene, el servidor deja de registrar advertencias, inicios de sesión o errores del sistema operativo y sus servicios, impidiendo auditorías y tareas de diagnóstico.

### 3. En la falla inyectada: ¿en qué escalón de la escalera apareció la evidencia decisiva? ¿Tu primera hipótesis era otra? ¿Qué te hizo descartarla?
* **Respuesta:** La evidencia decisiva apareció en el Escalón 3 con `systemctl status ssh`, que mostró el servicio en estado `inactive (dead)`. La primera hipótesis era un problema de red (cable desconectado o IP mal configurada), pero fue descartada en el Escalón 1 porque el comando `ping 192.168.100.10` respondía con 0% de pérdida de paquetes.

### 4. ¿Qué dato del informe inicial le importa más al gerente y cuál al próximo sysadmin que herede el servidor? ¿Por qué no son el mismo?
* **Respuesta:** Al gerente le importan los datos de **capacidad y recursos** (`df -h`, `free -h`) para evaluar costos, necesidades de compra o ciclo de vida del hardware. Al sysadmin le importan la **versión exacta del SO** (`hostnamectl`), los **servicios corriendo** (`systemctl`) y el **direccionamiento de red** (`ip -br a`) para planificar el mantenimiento, la compatibilidad de software, la seguridad y el monitoreo diario.
