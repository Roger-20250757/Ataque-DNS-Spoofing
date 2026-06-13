# DNS Spoofing / DNS Poisoning Attack — Networking Lab

**Estudiante:** Roger Rodriguez  
**Matricula:** 20250757   
**Fecha:** 12 de junio de 2026  
**Link:** https://youtu.be/nQchsKQuzgs

---

## Descripcion del Ataque

El **DNS Spoofing** (tambien conocido como DNS Poisoning) es un ataque donde el atacante intercepta las consultas DNS de la victima y responde con una direccion IP falsa, redirigiendo al usuario hacia una pagina web controlada por el atacante sin que este lo note.

---

## Topologia

```
Cloud0(SSH)
    |
   eth1
    |
[Kali Linux]                    [Ubuntu Victima]
 25.7.10.10                       25.7.20.20
   eth0                              ens3
    |                                 |
   e0/0                             e0/2
    |                                 |
    +--------[SW-1 CiscoIOL L2]------+
                    |
                   e0/1 (trunk)
                    |
                  fa0/0
                    |
               [R1 - 3725]
           fa0/0.10: 25.7.10.1
           fa0/0.20: 25.7.20.1
```

### Direccionamiento IP

| Dispositivo     | IP          | VLAN    | Mascara | Gateway    |
|-----------------|-------------|---------|---------|------------|
| Kali Linux      | 25.7.10.10  | VLAN 10 | /24     | 25.7.10.1  |
| Ubuntu Victima  | 25.7.20.20  | VLAN 20 | /24     | 25.7.20.1  |
| R1 fa0/0.10     | 25.7.10.1   | VLAN 10 | /24     | —          |
| R1 fa0/0.20     | 25.7.20.1   | VLAN 20 | /24     | —          |

---

## Herramientas Utilizadas

| Herramienta    | Version   | Uso                                        |
|----------------|-----------|--------------------------------------------|
| Ettercap       | 0.8.4     | ARP Poisoning + DNS Spoof plugin           |
| DNSChef        | 0.4       | Servidor DNS falso                         |
| Apache2        | 2.4       | Servidor web para alojar la pagina falsa   |
| EVE-NG         | Community | Plataforma de emulacion                    |
| Kali Linux     | 2026.1    | Sistema atacante                           |
| Ubuntu Server  | 22.04     | Sistema victima                            |

---

## Requisitos

- Kali Linux con permisos root
- Apache2 instalado: `sudo apt install apache2 -y`
- Ettercap instalado: `sudo apt install ettercap-text-only -y`
- DNSChef instalado: `sudo apt install dnschef -y`
- Archivo `/etc/ettercap/etter.dns` configurado
- Conectividad entre Kali y Ubuntu-Victima

---

## Uso del Script

```bash
# Clonar el repositorio
git clone https://github.com/tu-usuario/dns-spoofing.git
cd dns-spoofing

# Dar permisos de ejecucion
chmod +x dns_spoof_attack.py

# Ejecutar como root
sudo python3 dns_spoof_attack.py
```

---

## Parametros del Script

| Parametro      | Valor         | Descripcion                        |
|----------------|---------------|------------------------------------|
| ATTACKER_IP    | 25.7.10.10    | IP del atacante (Kali - VLAN 10)   |
| VICTIM_IP      | 25.7.20.20    | IP de la victima (Ubuntu - VLAN 20)|
| GATEWAY_IP     | 25.7.20.1     | Gateway de la victima              |
| INTERFACE      | eth0          | Interfaz de red del atacante       |
| FAKE_DOMAIN    | itla.edu.do   | Dominio a falsificar               |

---

## Funcionamiento

```
1. Apache2 sirve la pagina falsa en 25.7.10.10
2. DNSChef responde itla.edu.do → 25.7.10.10
3. Ettercap envenena el ARP entre victima y gateway
4. Victima consulta itla.edu.do → recibe 25.7.10.10
5. Victima carga la pagina falsa de ITLA
```

---

## Capturas de Pantalla

### 1. Configuracion etter.dns
|<img width="151" height="31" alt="image" src="https://github.com/user-attachments/assets/d46c5e5f-f982-406d-b33b-55b5e2c154b6" />
|

### 2. Ettercap corriendo — ARP Poisoning activo
|<img width="319" height="260" alt="image" src="https://github.com/user-attachments/assets/110a0423-fdda-4d46-9d5a-98d6aef8de40" />
|

### 3. Verificacion — nslookup desde Ubuntu Victima
|<img width="284" height="92" alt="image" src="https://github.com/user-attachments/assets/afef5d63-a30c-4af2-9d9b-a5cd50e91a52" />
|

### 4. Pagina falsa cargada en la victima
|<img width="467" height="315" alt="image" src="https://github.com/user-attachments/assets/04b201b5-f66e-4bdc-9955-5848b36e0bd6" />
|

---

## Contra-Medida

Agregar una entrada estatica en `/etc/hosts` de la victima para los dominios criticos, forzando la resolucion hacia la IP real e ignorando cualquier respuesta DNS manipulada.

```bash
# En la maquina victima
echo '104.26.13.23   itla.edu.do' >> /etc/hosts

# Verificar que resuelve la IP real
ping -c 3 itla.edu.do
# Resultado: PING itla.edu.do (104.26.13.23) — IP real
```

Otras contramedidas recomendadas:
- **DNSSEC** — valida criptograficamente las respuestas DNS
- **DNS sobre HTTPS (DoH)** — cifra las consultas DNS
- **Dynamic ARP Inspection (DAI)** — previene el ARP Poisoning en el switch
