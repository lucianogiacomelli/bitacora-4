# Informe de Laboratorio N.º 3 - Scripting: que el servidor se revise solo

**Grupo:** bitacora-4
**Integrantes:**
* Luciano Giacomelli (lucianogiacomelli8@gmail.com)

---

## 1. Implementación del Script de Diagnóstico

* **Ruta:** `/usr/local/bin/healthcheck.sh`
* **Log:** `/var/log/healthcheck.log`
* **Estructura y diseño:** Se desarrollaron tres funciones independientes (`uso_disco`, `uso_memoria` y `ssh_escucha`). Cada fallo incrementa un contador `alertas` para asegurar la ejecución completa de todos los testeos sin salidas tempranas. Si el contador es 0 emite `OK` y código de salida `0`; ante cualquier falla emite `ALERTA` y código de salida `1`.
* **Validación con ShellCheck:**
  * Comando: `shellcheck /usr/local/bin/healthcheck.sh`
  * Resultado: Código `0` sin observaciones ni advertencias sintácticas.

---

## 2. Verificación de Rutas Lógicas y Salidas

### Camino nominal (salida 0)

* **Comando:** `sudo /usr/local/bin/healthcheck.sh; echo "salió con: $?"`
* **Salida terminal:** `salió con: 0`
* **Registro en log:**

```text
2026-09-16 17:07:50 srv2 OK disco=11% mem=23% ssh=escucha
```

### Camino de alerta forzada (salida 1)

* **Comando:** `sudo env UMBRAL_DISCO=1 /usr/local/bin/healthcheck.sh; echo "salió con: $?"`
* **Salida terminal:** `salió con: 1`
* **Registro en log:**

```text
2026-09-16 17:07:51 srv2 ALERTA disco=11% mem=23% ssh=escucha
```

---

## 3. Automatización: Cron vs Systemd Timer

### Evidencia de ejecución con cron

* Tarea configurada en `/etc/cron.d/healthcheck` (`*/5 * * * * root /usr/local/bin/healthcheck.sh`).
* Registro generado automáticamente en `/var/log/healthcheck.log`:

```text
2026-09-16 17:10:02 srv2 OK disco=11% mem=23% ssh=escucha
```

### Evidencia de ejecución con systemd timer

* Unidades creadas en `/etc/systemd/system/healthcheck.service` y `/etc/systemd/system/healthcheck.timer`.
* Salida de `systemctl list-timers healthcheck.timer`:

```text
NEXT                         LEFT          LAST                         PASSED    UNIT               ACTIVATES
Wed 2026-09-16 17:16:39 -03  4min 52s left Wed 2026-09-16 17:11:39 -03  8s ago    healthcheck.timer  healthcheck.service
```

* Registro en `/var/log/healthcheck.log`:

```text
2026-09-16 17:11:39 srv2 OK disco=11% mem=24% ssh=escucha
```

---

## 4. Diagnosis Tejida #2 (Rotura de Cierre)

* **Variante asignada:** Pérdida de permisos de ejecución en el script de diagnóstico (`chmod -x` / modo `644`).
* **Síntoma observable:** El script dejó de correr en segundo plano. Al ejecutarlo manualmente con `sudo /usr/local/bin/healthcheck.sh`, la consola arrojó `sudo: /usr/local/bin/healthcheck.sh: command not found` con código de retorno `1`.
* **Dónde apareció la evidencia:** En la salida de `ls -l /usr/local/bin/healthcheck.sh`:

```text
-rw-r--r-- 1 root root 896 sep 16 17:06 /usr/local/bin/healthcheck.sh
```

* **Causa raíz:** El script carecía del bit de ejecución (`x`). Para que el kernel procese un script invocado directamente por su ruta absoluta e interprete el shebang, es indispensable que tenga permisos de lectura y ejecución.
* **Fix aplicado:** Se restablecieron los permisos ejecutables con `sudo chmod 755 /usr/local/bin/healthcheck.sh`. La verificación posterior arrojó `salió con: 0` y registró `2026-09-16 17:29:26 srv2 OK disco=11% mem=24% ssh=escucha`.

---

## 5. Preguntas de Cierre

### 1. Tu script salió con 1 cuando el disco estaba lleno. ¿Quién lee ese número y qué hace con él? Nombrá dos cosas distintas que podrían leerlo.

El número es el código de retorno del proceso (`exit status`), expuesto en la variable `$?`. Dos entidades distintas que lo consumen:

1. **Un agente de monitoreo (como Zabbix, Nagios o Datadog Agent):** Inspecciona el valor devuelto y, si es distinto de cero, conmuta el estado de la métrica a `CRITICAL` y dispara una alerta por correo o webhook.
2. **Un pipeline de despliegue continuo (CI/CD o script orquestador):** Evalúa el código previo a una actualización (`if ! /usr/local/bin/healthcheck.sh; then exit 1; fi`) abortando el despliegue si el nodo no está saludable.

### 2. El script anda a mano y no bajo cron. Antes de tocar nada: ¿qué tres diferencias hay entre las dos formas de ejecutarlo?

1. **Variable PATH:** Cron arranca con un `PATH` mínimo (`/usr/bin:/bin`), ignorando rutas estándar como `/usr/local/bin` o `/sbin` salvo declaración explícita.
2. **Entorno interactivo y variables de sesión:** Al ejecutar a mano se dispone de una terminal (TTY) y de variables del shell (`$USER`, `$HOME`, cargadas desde `.bashrc`); cron corre como demonio no interactivo en un entorno limpio.
3. **Directorio de trabajo actual (CWD):** De forma manual se hereda el directorio donde está parado el usuario; en cron la tarea inicia por defecto parada en el `$HOME` del usuario o en `/`.

### 3. Elegiste dejar cron o el timer. Justificá con un criterio operativo y decí en qué caso elegirías el otro.

* **Elección adoptada:** Se conservó **systemd timer** programado a las 06:00 AM con `Persistent=true`.
* **Criterio operativo:** La directiva `Persistent=true` asegura que, si el servidor se encuentra apagado durante la ventana de las 06:00 AM, la tarea se dispare inmediatamente al encender la máquina. Además, consolida el estado y las fallas dentro de `journalctl` sin requerir un demonio MTA local de correo.
* **Cuándo elegiría el otro:** Cron es la opción adecuada en contenedores Docker mínimos, entornos embebidos o servidores sin systemd, así como para usuarios estándar sin privilegios para escribir en `/etc/systemd/system/`.

### 4. Durante la prueba, el healthcheck escribió una línea cada 5 minutos. ¿Qué problema traería dejar esa frecuencia durante seis meses y cómo limitarías el crecimiento del log?

* **Cálculo:** 1 línea cada 5 min = 12 líneas/hora = 288 líneas/día. En 6 meses (180 días) se acumulan **51.840 líneas** de texto redundante (~3,3 MB).
* **Problema:** Ensucia el registro con miles de estados nominales idénticos, dificultando búsquedas forenses con `grep` y aumentando el consumo innecesario de almacenamiento.
* **Solución técnica:** Configurar **`logrotate`** en `/etc/logrotate.d/healthcheck` con periodicidad mensual, retención fija y compresión:

```text
/var/log/healthcheck.log {
    monthly
    rotate 6
    compress
    missingok
    notifempty
}
```

### 5. ¿Qué chequeo agregarías vos, pensando en lo que pasó en P02? Escribí la condición en una línea de bash.

Para prevenir la descarga de binarios o scripts no autorizados en directorios temporales compartidos (como el archivo `/tmp/x.sh` de la P02), agregaría una verificación que detecte archivos ejecutables en `/tmp`:

```bash
(( $(find /tmp -maxdepth 1 -type f -executable 2>/dev/null | wc -l) > 0 ))
```
