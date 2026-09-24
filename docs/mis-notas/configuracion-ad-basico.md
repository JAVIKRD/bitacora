# Guía: Configuración de un Entorno Active Directory Básico

Guía práctica paso a paso para desplegar un laboratorio de **Active Directory** en Windows Server, unir una estación de trabajo Windows 10 al dominio, crear Unidades Organizativas (OU) y dar de alta usuarios para prácticas de seguridad ofensiva y pentesting.

---

## Fases del Despliegue

1. **Configuración de IP estática**
2. **Cambio del nombre del servidor**
3. **Instalación del rol AD DS (Servicios de Dominio de Active Directory)**
4. **Promoción a Controlador de Dominio (Domain Controller)**
5. **Unión de un equipo cliente Windows 10 al Dominio**
6. **Creación de Unidades Organizativas (OU) y Usuarios de Dominio**
7. **Verificación de autenticación remota con NetExec**

---

## 1. Configuración de IP estática

Para que el Controlador de Dominio funcione de forma estable y pueda resolver peticiones DNS de los clientes, es indispensable configurar una dirección IP fija.

Accedemos a las conexiones de red:

![Configuración Adaptador](assets/Pasted%20image%2020260924040811.png)

Abrimos las propiedades del protocolo IPv4:

![Propiedades IPv4](assets/Pasted%20image%2020260924040959.png)

Asignamos una dirección IP fija dentro del rango de nuestro laboratorio, asegurando que el servidor DNS preferido apunte a la propia máquina o a `127.0.0.1`:

![IP Estática Configurada](assets/Pasted%20image%2020260924042053.png)

### Ajuste del Firewall de Windows

En un entorno de laboratorio controlado podemos desactivar el firewall de Windows para evitar bloqueos en las fases iniciales de enumeración y pruebas:

![Ajustes de Firewall](assets/Pasted%20image%2020260924042423.png)

Desactivamos los tres perfiles de red (Dominio, Privado y Público):

![Firewall Desactivado](assets/Pasted%20image%2020260924042459.png)

Comprobamos conectividad mediante `ping` desde nuestra máquina atacante:

![Prueba de Ping](assets/Pasted%20image%2020260924042559.png)

---

## 2. Cambio del Nombre del Servidor

Asignamos un nombre descriptivo al servidor (por ejemplo, `DC01` o `ServerDC`):

![Propiedades del Servidor](assets/Pasted%20image%2020260924042914.png)

Hacemos doble clic en el nombre del equipo y definimos el nuevo identificador:

![Cambio de Nombre](assets/Pasted%20image%2020260924043011.png)

Reiniciamos el sistema operativo para aplicar los cambios y verificamos que el nuevo nombre se refleje correctamente:

![Nombre Aplicado](assets/Pasted%20image%2020260924043343.png)

---

## 3. Instalación del Rol AD DS

Abrimos el **Server Manager** y seleccionamos **Add roles and features**:

![Server Manager Roles](assets/Pasted%20image%2020260924043457.png)

![Asistente de Roles](assets/Pasted%20image%2020260924043555.png)

Marcamos el rol **Active Directory Domain Services (AD DS)** y aceptamos las características requeridas:

![Selección AD DS](assets/Pasted%20image%2020260924043734.png)

Avanzamos en el asistente manteniendo las opciones por defecto y procedemos a la instalación:

![Instalación en Curso](assets/Pasted%20image%2020260924043926.png)

Una vez completada la instalación, cerramos el asistente:

![Instalación Completa](assets/Pasted%20image%2020260924044611.png)

---

## 4. Promover el Servidor a Controlador de Dominio

En el Server Manager aparecerá un icono de notificación triangular amarillo. Hacemos clic en **Promote this server to a domain controller**:

![Notificación Promover DC](assets/Pasted%20image%2020260924044740.png)

![Asistente de Configuración](assets/Pasted%20image%2020260924044802.png)

Seleccionamos **Add a new forest** e indicamos el nombre raíz del dominio (por ejemplo, `domain.com`):

![Nuevo Bosque](assets/Pasted%20image%2020260924044933.png)

Establecemos la contraseña para el modo de restauración de servicios de directorio (DSRM):

![Contraseña DSRM](assets/Pasted%20image%2020260924045206.png)

Mantenemos el nivel funcional en Windows Server 2016, avanzamos en las comprobaciones de requisitos previos y pulsamos **Install**:

![Instalación del DC](assets/Pasted%20image%2020260924045650.png)

El servidor se reiniciará automáticamente y ya estará operando como Controlador de Dominio.

---

## 5. Unir un Equipo Windows 10 al Dominio

En la máquina cliente con Windows 10, accedemos a la configuración de red IPv4:

![Red Windows 10](assets/Pasted%20image%2020260924051103.png)

Configuramos una IP dentro del mismo segmento de red y asignamos como **servidor DNS primario la dirección IP de nuestro Controlador de Dominio**:

![DNS hacia el DC](assets/Pasted%20image%2020260924051348.png)

Nos aseguramos de que el firewall permita la comunicación mutua:

![Firewall Windows 10](assets/Pasted%20image%2020260924051854.png)

Pulsamos `Win + R` y ejecutamos `sysdm.cpl`:

```cmd
sysdm.cpl
```

![Ejecutar sysdm.cpl](assets/Pasted%20image%2020260924052100.png)

En las propiedades del sistema hacemos clic en **Change...** (Cambiar):

![Propiedades del Sistema](assets/Pasted%20image%2020260924052235.png)

Cambiamos de Grupo de Trabajo (Workgroup) a **Domain** e ingresamos el nombre del dominio:

![Unirse al Dominio](assets/Pasted%20image%2020260924052529.png)

Ingresamos las credenciales de un usuario administrador del dominio:

![Credenciales de Dominio](assets/Pasted%20image%2020260924052924.png)

Aparecerá el mensaje de confirmación de bienvenida al dominio:

![Bienvenida al Dominio](assets/Pasted%20image%2020260924052954.png)

Reiniciamos la máquina cliente para completar la integración.

---

## 6. Gestión de Usuarios y Unidades Organizativas (OU)

En el Controlador de Dominio, abrimos la consola de administración ejecutando `dsa.msc` (`Win + R`):

```cmd
dsa.msc
```

![Ejecutar dsa.msc](assets/Pasted%20image%2020260924145339.png)

### ¿Qué es una Unidad Organizativa (OU) y por qué es clave en seguridad?

Una **Organizational Unit (OU)** es un contenedor lógico dentro del Active Directory que permite estructurar usuarios, grupos y equipos.

1. **Organización lógica**: Permite clasificar por departamentos (`IT`, `Ventas`, `Recursos Humanos`, `Servidores`).
2. **Delegación de permisos y abuso de ACLs**: Es posible delegar el control sobre una OU específica sin otorgar permisos de `Domain Admin`. Por ejemplo, permitir que un operador de soporte pueda restablecer contraseñas de la OU `Ventas`. En auditorías ofensivas y laboratorios de pentesting, herramientas como BloodHound permiten mapear y explotar relaciones de permisos sobre OUs (`GenericAll`, `WriteDacl`, `ForceChangePassword`).
3. **Aplicación granular de Directivas de Grupo (GPO)**: Permite vincular políticas específicas para cada sector de la empresa.

### Creación de una Unidad Organizativa

Abrimos **Active Directory Users and Computers**:

![Consola ADUC](assets/Pasted%20image%2020260924150955.png)

![Estructura del Dominio](assets/Pasted%20image%2020260924151103.png)

Hacemos clic derecho sobre nuestro dominio, seleccionamos **New** -> **Organizational Unit**:

![Nueva OU](assets/Pasted%20image%2020260924151408.png)

Asignamos un nombre a la OU (por ejemplo, `lab-users`):

![Nombre de OU](assets/Pasted%20image%2020260924151640.png)

### Creación de Usuarios de Dominio

Hacemos clic derecho sobre la OU `lab-users`, seleccionamos **New** -> **User**:

![Nuevo Usuario](assets/Pasted%20image%2020260924151915.png)

Completamos los datos del usuario:

![Datos del Usuario](assets/Pasted%20image%2020260924152228.png)

Asignamos una contraseña robusta y marcamos la opción **Password never expires** (desmarcando *User must change password at next logon* para facilitar las pruebas automatizadas):

![Configuración de Contraseña](assets/Pasted%20image%2020260924152646.png)

> [!TIP]
> Si dejamos marcada la opción *"User must change password at next logon"*, al intentar autenticar con herramientas como NetExec obtendremos un error de contraseña expirada hasta que se inicie sesión interactivamente en la estación de trabajo.

---

## 7. Verificación del Acceso Remoto con NetExec

Desde nuestra máquina de auditoría verificamos que las credenciales del usuario sean válidas contra el servicio SMB:

```bash
nxc smb 192.168.100.50 -u 'jose' -p '1725j@vik'
```

![NetExec Verificación Exitosa](assets/Pasted%20image%2020260924152902.png)

Observamos el resultado `[+] domain.com\jose:1725j@vik`, confirmando la autenticación exitosa.

A partir de este entorno base se pueden crear múltiples cuentas, grupos y relaciones de pertenencia para simular escenarios de auditoría y explotación de Active Directory.
