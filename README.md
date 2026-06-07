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
## Fase 2: Escaneo con Nmap / Zenmap
## Fase 3: Identificación de vulnerabilidades (NSE)
## Fase 4: Explotación - distcc (CVE-2004-2687)
## Fase 5: Reverse Shell
## Fase 6: Explotación - vsFTPd 2.3.4 (CVE-2011-2523)
## Conclusiones y lecciones aprendidas
