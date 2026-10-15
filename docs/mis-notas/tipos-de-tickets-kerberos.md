# ¿Qué Significa Forjar un Ticket Kerberos?

Forjar significa crear una credencial de Kerberos que parece legítima, sin seguir necesariamente el procedimiento normal de emisión.

En condiciones normales, el KDC crea tickets utilizando información criptográfica que el cliente no debería poder falsificar por su cuenta.

La oportunidad de ataque aparece cuando alguien obtiene secretos que permiten producir tickets que un sistema de destino acepta como auténticos.

Piensa en una acreditación de empresa que tiene elementos de seguridad que solo conoce la entidad emisora. Si alguien consigue el secreto utilizado para fabricar esas acreditaciones, podría crear una que parezca válida. El problema no es simplemente dibujar una tarjeta falsa: es conseguir que el mecanismo de verificación la acepte.

---

## Árbol de las principales técnicas

### A. Golden Ticket
- Se falsifica un **TGT**.
- Se basa habitualmente en obtener el secreto criptográfico: esto es el hash de la cuenta `krbtgt` del dominio.
- *Analogía:* Alguien consigue el secreto que permite fabricar acreditaciones generales del sistema.

### B. Silver Ticket
- Se falsifica un **ticket de servicio**.
- Se basa en obtener la clave de la cuenta que utiliza el servicio de destino.
- *Analogía:* Alguien consigue el secreto con el que se validan los pases de una oficina específica.

### C. Diamond Ticket
- Se parte de un **TGT legítimo** y se modifica su contenido.
- En el enfoque clásico, se aprovecha el acceso al secreto de `krbtgt` para volver a proteger criptográficamente el ticket modificado.
- *Analogía:* Se obtiene una acreditación auténtica y se altera su información antes de volver a sellarla con un secreto comprometido.

Son técnicas relacionadas, pero no son intercambiables. Cada una parte de una credencial distinta y depende de qué secreto se ha comprometido.

---

## ¿Qué es `krbtgt` y por qué importa tanto?

En Active Directory existe una cuenta especial llamada `krbtgt`. No es un usuario normal que inicia sesión y abre aplicaciones.

Su secreto criptográfico se utiliza para proteger los TGT del dominio. Por eso, si un atacante obtiene el material criptográfico (ejemplo: el hash adecuado de esa cuenta), puede llegar a fabricar TGT que el dominio acepte, siempre que se cumplan las condiciones necesarias.

Imagina esta jerarquía:

![Jerarquía krbtgt](assets/Pasted%20image%2020261011195033.png)

Aquí está el punto fundamental: el Golden Ticket no consiste en robar directamente la contraseña del administrador del dominio. Consiste en comprometer una clave que participa en la protección de los TGT y abusar de la confianza que el dominio deposita en ellos.

---

## ¿Cómo ocurre el ataque, de principio a fin?

1. **Paso 1. El atacante obtiene acceso a material criptográfico sensible.**  
   En el escenario clásico, consigue el secreto de `krbtgt`. Eso suele implicar un compromiso importante del dominio, aunque las circunstancias concretas dependen del entorno.

2. **Paso 2. Construye un TGT.**  
   El atacante prepara una credencial con datos como la identidad del usuario, el dominio y la información de autorización que pretende presentar.  
   En lugar de limitarse a pedir un TGT legítimo al KDC, genera uno utilizando el secreto comprometido.

3. **Paso 3. Protege criptográficamente el ticket.**  
   El atacante utiliza el material criptográfico que posee para crear una protección que pueda superar las comprobaciones correspondientes.  
   Esta es la razón por la que conocer únicamente el formato de un ticket no basta para falsificar uno aceptable.

4. **Paso 4. Presenta el TGT al KDC.**  
   Si el KDC lo acepta como válido, el atacante puede solicitar tickets para servicios del dominio bajo la identidad representada por ese TGT.

5. **Paso 5. Intenta acceder a un servicio.**  
   El atacante presenta el ticket de servicio correspondiente al recurso que busca comprometer. El servicio valida la credencial y aplica sus reglas de autorización.

### Un detalle que debes entender bien

El Golden Ticket no convierte automáticamente a una persona en administrador de todos los equipos.

El alcance depende de la validez del ticket, los datos de autorización, la configuración del dominio, los controles del servicio y otros factores. Una falsificación puede permitir privilegios elevados, pero no elimina mágicamente todas las comprobaciones del sistema.

👉 **Ver práctica:** [Creación de Golden Ticket](creacion-de-tickets.md#creacion-de-golden-ticket)

---

## Silver Ticket: el mismo concepto, pero con otra puerta

Ahora imagina que no tienes comprometido el secreto de `krbtgt`, pero sí tienes la clave de una cuenta de servicio.

Por ejemplo, una cuenta asociada a un servidor de archivos.

En este escenario, el objetivo cambia: en lugar de fabricar un TGT, el atacante fabrica un ticket de servicio.

![Silver Ticket Esquema](assets/Pasted%20image%2020261012024926.png)

---

## Comparación directa

| Característica | Golden Ticket | Silver Ticket |
| :--- | :--- | :--- |
| **Qué se falsifica** | TGT | Ticket de servicio |
| **Secreto comprometido en el caso clásico** | `krbtgt` | Clave de la cuenta del servicio |
| **Objetivo inicial** | Obtener tickets de servicio del KDC | Acceder al servicio objetivo |
| **Interacción con el KDC** | Se utiliza para solicitar tickets posteriores | El ticket falsificado puede presentarse directamente al servicio |
| **Alcance potencial** | Amplio dentro del dominio | Centrado en el servicio afectado, aunque el impacto depende de sus privilegios y relaciones |

**Regla mental:** Golden = falsificas la acreditación general. Silver = falsificas el pase para una oficina específica.

Y ojo: Silver Ticket no significa necesariamente que el acceso vaya a ser limitado o poco importante. Si el servicio comprometido tiene permisos elevados o controla información crítica, el impacto puede ser enorme.

👉 **Ver práctica:** [Creación de Silver Ticket](creacion-de-tickets.md#creacion-de-silver-ticket)

---

## Diamond Ticket: ¿qué cambia?

El Diamond Ticket se entiende mejor si lo comparas con el Golden Ticket:

- **Golden:** construyes un TGT falsificado utilizando el secreto comprometido de `krbtgt`.
- **Diamond:** partes de un TGT legítimo, modificas determinados datos y vuelves a protegerlo utilizando el secreto criptográfico comprometido.

¿Por qué hacerlo de otra manera? Porque el enfoque parte de una credencial que fue emitida legítimamente por el KDC. Eso cambia cómo se construye el ticket, pero no significa que el resultado sea indetectable.

La idea no es que Diamond sea una versión mágicamente invisible de Golden. Ambos dependen de condiciones específicas, y la detección requiere examinar más que el mero hecho de que el ticket tenga una apariencia normal.

---

## ¿Qué información lleva un ticket y por qué puedes modificarla?

Para entender qué hace posible una falsificación, necesitas conocer cuatro piezas:

1. **Identidad:**  
   ¿Quién dice ser el usuario del ticket? Por ejemplo, una cuenta de usuario del dominio.

2. **Datos de autorización:**  
   Incluyen información sobre los grupos y privilegios asociados a la identidad, representada en parte mediante el **PAC (Privilege Attribute Certificate)**.

3. **Validez temporal:**  
   El ticket contiene información como su período de validez. También intervienen las políticas del dominio y la sincronización horaria.

4. **Protección criptográfica:**  
   Permite verificar la autenticidad e integridad de la información protegida. Un atacante no puede cambiar libremente los datos y esperar que sigan siendo válidos sin poder generar la protección criptográfica correspondiente.

Por ejemplo, imagina que un ticket representa a un usuario que pertenece al grupo de empleados normales. El atacante querría manipular la información de autorización para representar a una identidad con privilegios mayores.

Pero cambiar un dato no basta: el ticket debe superar las verificaciones criptográficas y las comprobaciones del entorno. Además, la autorización efectiva depende de cómo el servicio interprete esa información.

El PAC es especialmente relevante porque transporta información de autorización, pero no es una contraseña ni equivale por sí solo a una prueba de que el usuario es administrador.

![PAC y estructura del ticket](assets/Pasted%20image%2020261014235626.png)
