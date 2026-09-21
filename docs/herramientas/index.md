# 🛠️ Hojas de Truco (Cheatsheets)

Colección organizada de chuletas técnicas, comandos esenciales y guías tácticas para auditorías de seguridad, Active Directory, pivoting y post-explotación.

---

## Índice de Hojas de Truco

| Herramienta / Técnica | Categoría | Descripción principal | Enlace directo |
| :--- | :--- | :--- | :--- |
| **NetExec (NXC)** | Active Directory / Red | Enumeración masiva de SMB, WinRM, LDAP, MSSQL, dump de SAM/LSA/NTDS y password spraying. | [Abrir guía →](nxc.md) |
| **Sliver C2** | C2 / Post-Explotación | Gestión de listeners, compilación de implantes, beacons, ejecución en memoria (.NET) y SOCKS5. | [Abrir guía →](sliver-c2.md) |
| **SeImpersonatePrivilege** | Escalada de Privilegios | Arsenal de exploits Potato (PrintSpoofer, GodPotato, JuicyPotato, RoguePotato) para Windows. | [Abrir guía →](seimpersonate.md) |
| **Microsoft SQL Server** | Bases de Datos | Enumeración de versiones, usuarios sysadmin, hashes, tablas sensibles y linked servers. | [Abrir guía →](mssql.md) |
| **Chisel & Pivoting** | Redes / Túneles | Reenvío de puertos locales y remotos, túneles reversos y pivoting dinámico con SOCKS5. | [Abrir guía →](chisel-pivoting.md) |

---

## Acceso Rápido por Tarjetas

<div class="grid cards" markdown>

-   :material-network:{ .lg .middle } __[NetExec (NXC)](nxc.md)__

    ---

    Comandos para SMB, WinRM, LDAP, volcado de credenciales (SAM, LSA, NTDS) y módulos avanzados.

-   :material-sword-cross:{ .lg .middle } __[Sliver C2](sliver-c2.md)__

    ---

    Flujo completo de trabajo para C2: listeners mTLS/HTTP/DNS, implantes, armory y evasión.

-   :material-shield-crown:{ .lg .middle } __[SeImpersonatePrivilege](seimpersonate.md)__

    ---

    Elevación rápida a SYSTEM mediante PrintSpoofer, GodPotato, JuicyPotato y RoguePotato.

-   :material-database:{ .lg .middle } __[Enumeración MSSQL](mssql.md)__

    ---

    Consultas SQL para auditoría profunda de servidores de bases de datos y servidores vinculados.

-   :material-transit-connection-variant:{ .lg .middle } __[Chisel & Pivoting](chisel-pivoting.md)__

    ---

    Configuración de túneles reversos, reenvío de puertos múltiples y proxychains con SOCKS5.

</div>
