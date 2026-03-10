# SIEM-LOCAL
 SIEM Lab Personal — Grafana + Loki + Promtail
🛡️ SIEM Lab Personal — Grafana + Loki + Promtail

Lab de ciberseguridad que armé para practicar monitoreo de logs y detección de amenazas desde el lado defensivo (Blue Team). Forma parte de mi plan de estudio para orientarme hacia roles de SOC / Incident Responder.


¿De qué se trata esto?
La idea fue simple: quería ver en tiempo real qué pasa en un sistema Linux cuando ejecuto ataques comunes. En lugar de solo practicar el lado ofensivo, quería también detectar esas mismas acciones como lo haría un analista SOC.
Para eso armé un SIEM casero con herramientas open source:

Loki guarda los logs
Promtail los recolecta desde la VM de Kali y los manda a Loki
Grafana los visualiza y permite hacer queries

Todo corre en Docker en mi máquina Windows, y la VM Kali Linux está en VirtualBox conectada por red host-only.

🏗️ Arquitectura
┌──────────────────────────┐         ┌─────────────────────────────────┐
│   KALI LINUX VM          │         │   WINDOWS HOST (Docker)         │
│   192.168.56.20          │         │   192.168.56.1                  │
│                          │         │                                 │
│  ┌──────────────────┐    │         │  ┌───────────────────────────┐  │
│  │  Promtail 3.0.0  │────┼────────►│  │  Loki 3.0.0  :3100        │  │
│  │  lee journald    │    │  push   │  │  almacena los logs        │  │
│  └──────────────────┘    │         │  └─────────────┬─────────────┘  │
│                          │         │                │                │
│  Captura:                │         │  ┌─────────────▼─────────────┐  │
│  • comandos sudo         │         │  │  Grafana 12.4.1  :3000    │  │
│  • intentos SSH          │         │  │  dashboards y queries     │  │
│  • eventos systemd       │         │  └───────────────────────────┘  │
│  • todo el journal       │         │                                 │
└──────────────────────────┘         │  ┌───────────────────────────┐  │
                                     │  │  Promtail (Windows)       │  │
                                     │  │  logs de contenedores     │  │
                                     │  └───────────────────────────┘  │
                                     └─────────────────────────────────┘
Red: Adaptador host-only de VirtualBox (192.168.56.0/24)

🧰 Qué usé
HerramientaVersiónPara quéGrafana12.4.1Ver los logs, hacer queries, dashboardsLoki3.0.0Guardar y indexar los logsPromtail3.0.0Recolectar logs (VM Kali + Docker)Docker Desktop4.63.0Correr el stack en WindowsVirtualBox—Virtualización de Kali LinuxKali LinuxrollingEntorno de práctica

🚀 Cómo levantarlo
Lo que necesitás tener instalado

Windows con Docker Desktop
VirtualBox con una VM Kali Linux
Adaptador de red host-only configurado (192.168.56.0/24)

1. Clonar el repo
bashgit clone https://github.com/Desaparecido0000/siem-lab.git
cd siem-lab
2. Levantar el stack en Windows
powershelldocker compose up -d
docker ps
Debería verse así:
grafana-siem     Up    0.0.0.0:3000->3000/tcp
loki             Up    0.0.0.0:3100->3100/tcp
promtail-windows Up
3. Abrir el firewall de Windows
Esto es necesario para que la VM pueda mandar logs a Loki:
powershellNew-NetFirewallRule -DisplayName "Loki SIEM Lab" `
  -Direction Inbound -Protocol TCP -LocalPort 3100 -Action Allow
4. Instalar Promtail en Kali
bashwget https://github.com/grafana/loki/releases/download/v3.0.0/promtail-linux-amd64.zip
unzip promtail-linux-amd64.zip
sudo mv promtail-linux-amd64 /usr/local/bin/promtail
sudo chmod +x /usr/local/bin/promtail
5. Configurar Promtail en Kali
Crear /etc/promtail/config.yml:
yamlserver:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/promtail-positions.yaml

clients:
  - url: http://192.168.56.1:3100/loki/api/v1/push

scrape_configs:
  - job_name: kali-journal
    journal:
      max_age: 12h
      path: /var/log/journal
      labels:
        job: kali-journal
        host: kali-lab
    relabel_configs:
      - source_labels: ['__journal__comm']
        target_label: command
      - source_labels: ['__journal_priority_keyword']
        target_label: level
      - source_labels: ['__journal__systemd_unit']
        target_label: unit
6. Crear el servicio systemd
bashsudo tee /etc/systemd/system/promtail.service << EOF
[Unit]
Description=Promtail
After=network.target

[Service]
ExecStart=/usr/local/bin/promtail -config.file=/etc/promtail/config.yml
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable promtail
sudo systemctl start promtail
7. Entrar a Grafana
Abrir http://localhost:3000 — usuario: admin / contraseña: siem1234
El datasource de Loki se configura automáticamente. Ir a Explore y probar:
logql{host="kali-lab"}
Si aparecen logs — el lab está funcionando ✅

🔍 Queries útiles en Grafana (LogQL)
Una de las partes más interesantes del lab fue aprender a escribir queries para detectar actividad sospechosa. Acá van las que más usé:
logql# Ver todos los logs de Kali
{host="kali-lab"}

# Solo comandos sudo
{host="kali-lab", command="sudo"}

# Intentos de autenticación fallidos
{host="kali-lab"} |= "Failed" or "failure"

# Actividad SSH
{host="kali-lab", unit="sshd.service"}

# Buscar cualquier mención a root
{host="kali-lab"} |= "root"

# Solo errores
{host="kali-lab", level="err"}

# Logs de los contenedores Docker
{job="docker"}

🧪 Escenarios practicados
Lo más valioso del lab fue poder ejecutar ataques reales y ver cómo quedan registrados en el SIEM. Acá un resumen de lo que practiqué:
EscenarioTécnica MITRECómo se ve en los logsExplotar binario SUIDT1548.001command="find" ejecutado como rootAbusar sudo mal configuradoT1548.003command="sudo" + vim + shellLeer /etc/shadowT1003.008command="sudo" + catCrackear hashes con johnT1110.001Registro en journalEnvenenar /etc/hostsT1565.001command="tee" sobre /etc/hostsDetectar borrado de logsT1562.002Gaps en el journalEnumeración de procesosT1057command="ps", command="pstree"Enumeración de redT1049command="ss", command="tcpdump"

📁 Estructura del proyecto
siem-lab/
├── docker-compose.yml          # Define todo el stack
├── promtail-windows.yml        # Config de Promtail para logs de Docker
├── provisioning/
│   └── datasources/
│       └── loki.yaml           # Datasource de Loki (auto-provisionado)
└── README.md

🐛 Problemas que tuve y cómo los resolví
context deadline exceeded en Promtail
→ El firewall de Windows estaba bloqueando el puerto 3100. Se resuelve agregando la regla del paso 3.
No aparecen datos en Grafana con {host="kali-lab"}
→ Verificar que Loki tiene el label:
powershellcurl "http://localhost:3100/loki/api/v1/label/host/values"
# Debe mostrar: {"data":["kali-lab"]}
Loki responde Ingester not ready
→ Es normal al arrancar, esperar 15-20 segundos y reintentar.
Promtail no inicia — puerto 9080 ocupado
→ Ya hay una instancia corriendo. Verificar con sudo systemctl status promtail.

📚 Recursos que usé

Documentación de Grafana Loki
Configuración de Promtail
MITRE ATT&CK
LogQL — lenguaje de queries de Loki
GTFOBins — referencia de escalada de privilegios


👤 Sobre este proyecto
Armé este lab como parte de un plan de estudio autodidacta de Sistemas Operativos y Ciberseguridad, enfocado en el lado defensivo. El objetivo a mediano plazo es orientarme hacia roles de SOC Nivel 1 / Incident Responder.
Lo que practiqué acá: Kali Linux · Docker · Grafana · Loki · journald · tcpdump · john · iptables · MITRE ATT&CK · privilege escalation · log analysis
