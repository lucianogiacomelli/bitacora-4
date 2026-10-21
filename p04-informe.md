# Informe de Laboratorio N.° 4 - El servidor está lento

**Grupo:** bitacora-4  
**Integrantes:**
* Fernando Rubiales
* Luciano Giacomelli

**Evidencias:** [p04-capturas.pdf](./p04-capturas.pdf)

---

## 1. Foto del antes (Línea de base)

* **Fecha y hora:** `mar 20 oct 2026 22:36:01 -03`
* **Cantidad de procesadores:** 1 núcleo detectado con `nproc`.
* **Carga inicial:**
  ```text
  sysadmin@srv1:~$ uptime
   22:36:01 up 3 min, 1 user, load average: 0,43, 0,38, 0,17

Memoria RAM y Swap:

sysadmin@srv1:~$ free -h
               total        used        free      shared  buff/cache   available
Mem:           960Mi       215Mi       557Mi       560Ki       327Mi       745Mi
Swap:          974Mi          0B       974Mi

Muestreo inicial de vmstat: La columna wa (I/O wait) se mantuvo en 0% con el procesador ocioso (id) en 99-100%

sysadmin@srv1:~$ vmstat 1 5
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in    cs us sy id wa st
 2  0      0 570624  15248 319756    0    0  1293   881  406   431  9 22 63  5  0
 1  0      0 570624  15248 319756    0    0     0     0  275    58  0  1 99  0  0
 1  0      0 570624  15248 319756    0    0     0     0  353    27  0  1 99  0  0
 1  0      0 570624  15248 319756    0    0     0     0  329    33  0  0 100 0  0
 1  0      0 570624  15248 319756    0    0     0     0  354    27  0  1 99  0  0

Monitoreo base (P03): Se verificó el registro continuo de /var/log/healthcheck.log:

2026-10-20 22:14:22 srv1 OK disco=11% mem=22% ssh escucha
2026-10-20 22:14:38 srv1 ALERTA disco=11% mem=22% ssh escucha
2026-10-20 22:20:01 srv1 OK disco=11% mem=22% ssh escucha
2026-10-20 22:21:42 srv1 OK disco=11% mem=23% ssh escucha

2. Configuración y Validación del Pasante
Creación de usuario y directorio:

  Comando: sudo groupadd soporte && sudo useradd -m -G soporte -s /bin/bash pasante
  
  Directorio con SGID: sudo install -d -m 2770 -g soporte /srv/soporte
  
  Permisos resultantes: drwxrws--- 2 root soporte 4096 oct 20 22:37 /srv/soporte

Regla acotada de sudoers:

  Archivo temporal validado: /tmp/pasante.sudo: parsed OK mediante sudo visudo -cf
  
  Instalación final: /etc/sudoers.d/pasante con permisos 440.

Pruebas de validación:

  Privilegios permitidos (sudo -l): Confirma únicamente (root) /usr/bin/systemctl restart nginx.
  
  Prueba negativa: Intento de lectura a /var/log/auth.log retorna Permiso denegado.
  
  Prueba positiva: Ejecución de sudo systemctl restart nginx autorizada por sudo.
  
  Persistencia SGID: Archivo /srv/soporte/prueba.txt hereda automáticamente el grupo soporte.

3. Diagnóstico del Escenario de Saturación

Escenario aplicado: sudo bash escenario.sh en srv1.

Identificación del cuello de botella:

  vmstat: La columna wa trepó a un rango sostenido de 66% a 75%, con una escritura continua (bo) de más de 1500 bloques/segundo.

procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in    cs us sy id wa st
 2  1      0 501552  35096 348892    0    0   332   302  350   182  2  7 88  3  0
 1  1      0 501552  35096 348892    0    0     0  1632  596  1010  0 34  0 66  0
 2  0      0 501552  35096 348892    0    0     0  1540  577   972  0 27  0 73  0
 2  0      0 501552  35096 348892    0    0     0  1552  577   960  0 28  0 72  0
 2  1      0 501552  35096 348892    0    0     0  1628  595  1006  0 25  0 75  0

  top: Se detectó el proceso dd (PID 1223) en estado D (Uninterruptible Sleep, bloqueado esperando respuesta de I/O del disco). El load average alcanzó 2,11 en un único núcleo.

Recurso saturado: Disco (cuello de botella de entrada/salida - I/O Bottleneck).

4. Resolución e Intervención
Detección del proceso padre:

  El script supervisor /usr/local/bin/sincronizador-respaldos.sh (PID 1216) relanzaba automáticamente a dd al ser terminado.

Acciones ejecutadas:

  Terminación del padre: sudo kill 1216

  Terminación del hijo residual: sudo kill -9 1269
  
  Limpieza del entorno: sudo bash escenario.sh limpiar

Recuperación del sistema:

  El tiempo de espera de I/O (wa) se redujo a 0% - 3%, el procesador ocioso (id) volvió a 96% - 100%, y el load average inició su descenso a 0,92.

sysadmin@srv1:~/asl-alumnos/p04$ uptime
 22:56:13 up 23 min, 2 users, load average: 0,92, 1,57, 1,03
sysadmin@srv1:~/asl-alumnos/p04$ vmstat 1 3
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in    cs us sy id wa st
 2  0      0 399004 130516 354936    0    0   223  1093  563   671  2 13 63 23  0
 1  0      0 399004 130516 354936    0    0     0     0  266    31  0  0 100 0  0
 1  0      0 399004 130516 354936    0    0     0    16  273    43  0  1 96  3  0

5. Preguntas de Cierre
1. El load average de 1 minuto es 4 y el de 15 es 1. ¿El problema está empezando o terminando? ¿Y si fuera al revés?

  Con 1 min en 4 y 15 min en 1, el problema está empezando (empeorando), ya que la carga inmediata supera notablemente la media histórica. Si fuera al revés (1 min en 1 y 15 min en 4), el problema está terminando (recuperándose) tras ceder un embotellamiento previo.

2. ¿Por qué renice puede no cambiar nada cuando el problema es de disco?

  Porque renice solo altera la prioridad de turnos en el planificador de CPU. Cuando el bloqueo es de disco (wa elevado y procesos en estado D), la tarea no compite por tiempo de procesador, sino que espera al controlador de almacenamiento físico.

3. Le diste al pasante permiso para reiniciar un servicio. ¿Qué podría hacer con ese permiso que no esté en el espíritu del ticket?
 
  Podría provocar una denegación de servicio (DoS) reiniciando el servicio en bucle permanente durante horarios pico, o aplicar un reinicio sobre configuraciones defectuosas  dejando el servicio inoperativo.

4. Tu healthcheck de P03, ¿habría detectado esto? Si no, escribí la condición que le agregarías.

  No lo habría detectado, ya que el script de P03 verificaba porcentaje de almacenamiento ocupado (df), uso de memoria (free) y puerto SSH (ss), pero no tiempos de espera de I/O. Condición correctiva:

[ "$(vmstat 1 2 | tail -1 | awk '{print $16}')" -ge 20 ] && return 1

5. El proceso que encontraste corría desde antes de ayer a la tarde, cuando empezaron las quejas. ¿Qué significa eso y cómo lo verificarías?

  Indica que el script comenzó como una tarea legítima o programada, pero derivó en degradación por congestión o acumulación imprevista de datos. Se verificaría analizando métricas de sar (sysstat), logs en journalctl o la marca temporal de ejecución mediante ps -eo pid,stime,args.


 
