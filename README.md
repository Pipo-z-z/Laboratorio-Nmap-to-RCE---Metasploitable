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






## Fase 3: Identificación de vulnerabilidades (NSE)






## Fase 4: Explotación - distcc (CVE-2004-2687)
## Fase 5: Reverse Shell
## Fase 6: Explotación - vsFTPd 2.3.4 (CVE-2011-2523)
## Conclusiones y lecciones aprendidas
