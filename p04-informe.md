# Práctica 4: "El servidor está lento"
**Materia:** Administración de Servicios en Linux
**Entorno:** Debian 12 (srv1 / cli1)
**Fecha:** 20 de Octubre de 2026
**Anexo de evidencias:** [Ver PDF de capturas](./p04-capturas.pdf)

---

## 1. Foto del antes (Línea de base con la máquina sana)

Se toman las métricas basales para conocer el estado normal del servidor antes de cualquier saturación.

### Fecha y hora exacta de la medición
Permite certificar que la foto del antes es previa al incidente.

sysadmin@srv1:~$ date
mar 20 oct 2026 22:36:01 -03

### Cantidad de procesadores del sistema
Indica cuántos núcleos tiene la máquina para poder comparar correctamente el load average.

sysadmin@srv1:~$ nproc
1

### Tiempo activo y promedio de carga inicial
Muestra que el servidor arrancó hace 3 minutos y los load averages (1, 5 y 15 min) están bajos y por debajo de 1.

sysadmin@srv1:~$ uptime
 22:36:01 up 3 min, 1 user, load average: 0,43, 0,38, 0,17

### Estado de memoria RAM y Swap
Comprueba que la memoria swap está en 0B de uso y hay suficiente memoria libre y disponible.

sysadmin@srv1:~$ free -h
               total        used        free      shared  buff/cache   available
Mem:           960Mi       215Mi       557Mi       560Ki       327Mi       745Mi
Swap:          974Mi          0B       974Mi

### Muestreo de actividad general con vmstat
Cinco muestras de 1 segundo. La primera no cuenta por ser el histórico desde el arranque; las restantes muestran wa (I/O wait) en 0% y el procesador ocioso (id) en 99-100%.

sysadmin@srv1:~$ vmstat 1 5
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in    cs us sy id wa st
 2  0      0 570624  15248 319756    0    0  1293   881  406   431  9 22 63  5  0
 1  0      0 570624  15248 319756    0    0     0     0  275    58  0  1 99  0  0
 1  0      0 570624  15248 319756    0    0     0     0  353    27  0  1 99  0  0
 1  0      0 570624  15248 319756    0    0     0     0  329    33  0  0 100 0  0
 1  0      0 570624  15248 319756    0    0     0     0  354    27  0  1 99  0  0

### Línea de base del registro de healthcheck (P03)
Comprobación de que el monitoreo implementado en la práctica anterior viene registrando actividad de forma automática.

sysadmin@srv1:~$ tail -n 10 /var/log/healthcheck.log
2026-10-20 22:14:22 srv1 OK disco=11% mem=22% ssh escucha
2026-10-20 22:14:38 srv1 ALERTA disco=11% mem=22% ssh escucha
2026-10-20 22:20:01 srv1 OK disco=11% mem=22% ssh escucha
2026-10-20 22:21:42 srv1 OK disco=11% mem=23% ssh escucha

---

## 2. Configuración y validación del usuario pasante

### Creación del grupo, usuario y directorio con bit SGID
Se crea el grupo soporte, el usuario pasante asociado, y el directorio con permiso 2770 (SGID) para que cualquier archivo creado herede el grupo soporte.

sysadmin@srv1:~$ sudo groupadd soporte
sysadmin@srv1:~$ sudo useradd -m -G soporte -s /bin/bash pasante
sysadmin@srv1:~$ sudo install -d -m 2770 -g soporte /srv/soporte

### Asignación de contraseña
Useradd deja las cuentas bloqueadas por defecto; se establece contraseña para habilitar el uso de sudo.

sysadmin@srv1:~$ sudo passwd pasante
passwd: contraseña actualizada correctamente

### Verificación de permisos y SGID en el directorio
Se verifica que el directorio cuente con drwxrws--- y pertenezca a root:soporte.

sysadmin@srv1:~$ ls -ld /srv/soporte
drwxrws--- 2 root soporte 4096 oct 20 22:37 /srv/soporte

### Definición y validación sintáctica de la regla sudoers
Para evitar dejar a todos los administradores bloqueados por un error sintáctico, la regla se escribe en /tmp, se valida con visudo -cf, y luego se instala en /etc/sudoers.d/.

sysadmin@srv1:~$ echo 'pasante ALL=(root) /usr/bin/systemctl restart nginx' > /tmp/pasante.sudo
sysadmin@srv1:~$ sudo visudo -cf /tmp/pasante.sudo
/tmp/pasante.sudo: parsed OK
sysadmin@srv1:~$ sudo install -m 440 -o root -g root /tmp/pasante.sudo /etc/sudoers.d/pasante

### Comprobación de comandos permitidos (sudo -l)
Se valida desde la identidad del pasante que únicamente tiene autorización para reiniciar nginx.

sysadmin@srv1:~$ sudo -u pasante sudo -l
[sudo] contraseña para pasante:
Matching Defaults entries for pasante on srv1:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User pasante may run the following commands on srv1:
    (root) /usr/bin/systemctl restart nginx

### Prueba negativa: Acceso denegado a archivos protegidos
Comprobación de que el pasante no puede leer logs administrativos del sistema.

sysadmin@srv1:~$ sudo -u pasante cat /var/log/auth.log
cat: /var/log/auth.log: Permiso denegado

### Prueba positiva: Ejecución del servicio autorizado con sudo
Se ejecuta el comando concedido. Responde que la unidad no existe (se instala formalmente en P09), confirmando que sudo validó la orden y no denegó el permiso.

sysadmin@srv1:~$ sudo -u pasante sudo /usr/bin/systemctl restart nginx
Failed to restart nginx.service: Unit nginx.service not found.

### Prueba de persistencia de grupo mediante SGID
El pasante crea un archivo en /srv/soporte/ y se verifica que pertenece al grupo soporte y no a su grupo personal.

sysadmin@srv1:~$ sudo -u pasante ls -l /srv/soporte/
total 0
-rw-r--r-- 1 pasante soporte 0 oct 20 22:43 prueba.txt

---

## 3. Diagnóstico del Escenario de Saturación

Se actualiza el repositorio y se ejecuta el escenario de carga provisto por la cátedra.

sysadmin@srv1:~/asl-alumnos/p04$ sha256sum -c SHA256SUMS
escenario.sh: La suma coincide
sysadmin@srv1:~/asl-alumnos/p04$ sudo bash escenario.sh
Listo. Tu servidor está lento.

### Medición 1: vmstat 1 5 (Aislamiento de I/O)
Las mediciones reflejan que la columna wa (espera de operaciones de entrada/salida) salta a valores críticos de entre 66% y 75%, con una tasa sostenida de escritura en disco (bo) superior a los 1500 bloques por segundo.

sysadmin@srv1:~/asl-alumnos/p04$ vmstat 1 5
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in    cs us sy id wa st
 2  1      0 501552  35096 348892    0    0   332   302  350   182  2  7 88  3  0
 1  1      0 501552  35096 348892    0    0     0  1632  596  1010  0 34  0 66  0
 2  0      0 501552  35096 348892    0    0     0  1540  577   972  0 27  0 73  0
 2  0      0 501552  35096 348892    0    0     0  1552  577   960  0 28  0 72  0
 2  1      0 501552  35096 348892    0    0     0  1628  595  1006  0 25  0 75  0

### Medición 2: top (Identificación del estado de proceso)
El load average trepó a 2,11 (en un único núcleo). El proceso dd (PID 1223) se encuentra en estado D (Uninterruptible Sleep), bloqueado a la espera de que el disco termine de responder.

sysadmin@srv1:~/asl-alumnos/p04$ top -b -n 1 | head -n 20
top - 22:47:40 up 15 min, 1 user, load average: 2,11, 0,83, 0,38
Tareas: 78 total, 2 running, 76 sleeping, 0 stopped, 0 zombie
%Cpu(s): 0,0 us, 91,7 sy, 0,0 ni, 0,0 id, 0,0 wa, 0,0 hi, 8,3 si, 0,0 st
MiB Mem:  960,8 total,  445,3 free,  238,7 used,  417,5 buff/cache
MiB Intercambio: 975,0 total,  975,0 free,    0,0 used.  722,1 avail Mem

  PID USER   PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
   38 root    0 -20       0      0      0 I   8,0   0,0   0:16.09 kworker+
  166 root   20   0       0      0      0 R   4,0   0,0   0:11.91 jbd2/sd+
 1223 root   20   0    5520    932    840 D   4,0   0,1   0:01.92 dd
 1224 sysad  20   0   11476   5104   3224 R   4,0   0,5   0:00.14 top

Recurso saturado confirmado: Disco (cuello de botella en I/O).

---

## 4. Identificación del responsable e Intervención

### Localización del proceso padre
Al verificar el árbol de procesos, se descubre que matar al proceso dd hace que vuelva a aparecer con otro PID porque está siendo invocado por un script padre supervisor (sincronizador-respaldos.sh, PID 1216).

sysadmin@srv1:~/asl-alumnos/p04$ ps -fp 1216
UID        PID  PPID  C STIME TTY          TIME CMD
root      1216     1  0 22:45 ?        00:00:00 /bin/bash /usr/local/bin/sincronizador-respaldos.sh

### Intervención y eliminación de la carga
Se finaliza el proceso padre para que deje de generar subprocesos y se elimina el proceso hijo dd activo.

sysadmin@srv1:~/asl-alumnos/p04$ sudo kill 1216
sysadmin@srv1:~/asl-alumnos/p04$ sudo kill -9 1269

### Foto del después (Comprobación del restablecimiento)
La cola de espera de entrada/salida colapsa a valores normales (wa en 0%-3%), el procesador vuelve a estar ocioso (id en 96%-100%) y el load average comienza a caer.

sysadmin@srv1:~/asl-alumnos/p04$ uptime
 22:56:13 up 23 min, 2 users, load average: 0,92, 1,57, 1,03
sysadmin@srv1:~/asl-alumnos/p04$ vmstat 1 3
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in    cs us sy id wa st
 2  0      0 399004 130516 354936    0    0   223  1093  563   671  2 13 63 23  0
 1  0      0 399004 130516 354936    0    0     0     0  266    31  0  0 100 0  0
 1  0      0 399004 130516 354936    0    0     0    16  273    43  0  1 96  3  0

---

## 5. Preguntas de Cierre

1. El load average de 1 minuto es 4 y el de 15 es 1. ¿El problema está empezando o terminando? ¿Y si fuera al revés?
Con 1 min en 4 y 15 min en 1, el problema está empezando (empeorando), ya que la carga a corto plazo es sensiblemente mayor a la media de largo plazo. Si fuera al revés (1 min en 1 y 15 min en 4), el problema está terminando (recuperándose), indicando que hubo una congestión previa que ya cedió.

2. ¿Por qué renice puede no cambiar nada cuando el problema es de disco?
Porque renice modifica la prioridad de turnos en el planificador de CPU. Cuando el cuello de botella es saturación de disco (I/O wait en estado D), los procesos no están disputando CPU, sino esperando respuesta física del almacenamiento.

3. Le diste al pasante permiso para reiniciar un servicio. ¿Qué podría hacer con ese permiso que no esté en el espíritu del ticket?
Podría causar una denegación de servicio (DoS) reiniciando el servicio web en bucle continuo durante horario de producción, o reiniciar un servicio cuya configuración esté rota dejándolo completamente inoperativo.

4. Tu healthcheck de P03, ¿habría detectado esto? Si no, escribí la condición que le agregarías.
El script de P03 medía porcentaje de almacenamiento lleno (df), uso de memoria (free) y escucha en puerto 22 (ss); por lo tanto no lo habría detectado, ya que el colapso fue por tiempo de espera y tasa de I/O (wa), no por disco lleno. Condición para agregarlo:
[ "$(vmstat 1 2 | tail -1 | awk '{print $16}')" -ge 20 ] && return 1

5. El proceso que encontraste corría desde antes de ayer a la tarde, cuando empezaron las quejas. ¿Qué significa eso y cómo lo verificarías?
Significa que el proceso pudo haber iniciado como una tarea normal o de mantenimiento programado (como sincronización de copias), pero que debido a un volumen imprevisto o atasco en el almacenamiento entró en un bucle de degradación posterior. Se verificaría analizando registros de auditoría en journalctl, métricas históricas en sar o la fecha exacta de inicio con ps -eo pid,stime,args.
