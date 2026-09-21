> **Aviso Importante / Reglas de Publicación:**
> - Solo se deben publicar writeups de máquinas que estén en estado **Retirada (Retired)** en Hack The Box.
> - **Nunca** publicar flags (`user.txt` o `root.txt`), credenciales reales no anonimizadas ni datos o configuraciones de conexión VPN.

# [Nombre de la Máquina]

- **Dificultad:** Fácil / Media / Difícil / Insane
- **Sistema Operativo:** Linux / Windows
- **Fecha de resolución:** AAAA-MM-DD
- **Puntos clave:** <!-- TODO: tags o técnicas principales -->

---

## Resumen
<!-- Breve descripción de la máquina, vectores de entrada y resumen del camino de explotación -->

## Reconocimiento
<!-- Escaneo de puertos (nmap), enumeración web, servicios descubiertos y recopilación inicial -->

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn <IP> -oG allPorts
```

## Foothold
<!-- Vector inicial de acceso y obtención de la primera shell -->

## Escalada de privilegios
<!-- Enumeración interna, vectores de elevación (sudo, SUID, capabilities, kernels, credenciales) y acceso root/SYSTEM -->

## Por qué funciona
<!-- Explicación concisa del fallo de diseño, mala configuración, técnica o CVE explotado en pocas líneas -->
