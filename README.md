# Instalación de Snort 3 en Raspberry Pi 5

Esta es una guía actualizada (2025) para compilar, instalar y configurar Snort 3 NIDS en una Raspberry Pi 5 corriendo Raspberry Pi OS. Esta guía soluciona problemas comunes de dependencias y errores de memoria (SIGABRT) específicos de la arquitectura ARM64. 

## Prerrequisitos

* **Hardware:** Raspberry Pi 5. (En mi caso 8GB de RAM y 128GB de memoria en microSD)
* **OS:** Raspberry Pi OS.
* **Red:** Conexión a internet para descargar paquetes.

## ¿Qué es Snort?

Snort es un sistema de detección y prevención de intrusiones (IDS/IPS) de código abierto que monitorea el tráfico de red en tiempo real para identificar y alertar sobre actividades maliciosas.

## 1. Instalación de Dependencias

Actualizar sistema e instalar herramientas de compilación y librerías requeridas:

```bash
sudo apt update && sudo apt upgrade -y

sudo apt install -y build-essential autotools-dev libdumbnet-dev \
libluajit-5.1-dev libpcap-dev zlib1g-dev pkg-config libhwloc-dev \
cmake liblzma-dev openssl libssl-dev cpputest libsqlite3-dev \
libtool uuid-dev git autoconf bison flex libcmocka-dev \
libnetfilter-queue-dev libunwind-dev libmnl-dev libpcre2-dev

2. Compilar e Instalar LibDAQ
Snort 3 requiere la última versión de DAQ desde el código fuente.

mkdir -p ~/snort_src
cd ~/snort_src
git clone [https://github.com/snort3/libdaq.git](https://github.com/snort3/libdaq.git)
cd libdaq
./bootstrap
./configure
make
sudo make install

3. Compilar e Instalar Snort 3
Nota importante: En Raspberry Pi 5 (ARM64), la librería tcmalloc puede causar errores SIGABRT. Se compila sin ella para garantizar estabilidad.

cd ~/snort_src
git clone [https://github.com/snort3/snort3.git](https://github.com/snort3/snort3.git)
cd snort3

# Configuración SIN tcmalloc para evitar crash en ARM64
./configure_cmake.sh --prefix=/usr/local

cd build
make -j$(nproc)
sudo make install #Va a tardar un buen rato así que no te preocupes.
sudo ldconfig

Verificar instalación:
snort -V

4. Configuración de Reglas
Estructura de directorios:
sudo mkdir -p /usr/local/etc/rules
sudo mkdir -p /usr/local/etc/so_rules
sudo mkdir -p /usr/local/etc/lists
sudo touch /usr/local/etc/rules/local.rules
sudo touch /usr/local/etc/lists/default.blocklist
sudo touch /usr/local/etc/lists/default.allowlist

Descargar reglas de la comunidad
cd ~/snort_src
wget [https://www.snort.org/downloads/community/snort3-community-rules.tar.gz](https://www.snort.org/downloads/community/snort3-community-rules.tar.gz)
tar xzf snort3-community-rules.tar.gz
sudo cp snort3-community-rules/snort3-community.rules /usr/local/etc/rules/

Editar snort.lua
Editar /usr/local/etc/snort/snort.lua y modificar la sección ips:

ips =
{
    include = { "/usr/local/etc/rules/snort3-community.rules", "/usr/local/etc/rules/local.rules" },
    variables = default_variables
}

5. Prueba de Funcionamiento (ICMP)
Añadir regla de prueba en /usr/local/etc/rules/local.rules:
alert icmp any any -> any any (msg:"Ping detectado"; sid:1000001; rev:1;)

Ejecutar en modo consola (ajustar eth0 a tu interfaz):
sudo snort -c /usr/local/etc/snort/snort.lua -i eth0 -A alert_fast

6. Configurar como Servicio (Systemd)
Para que Snort arranque automáticamente al iniciar la Raspberry.

Crear usuario:
sudo useradd -r -s /usr/sbin/nologin -M -c SNORT_IDS snort

Crear servicio /etc/systemd/system/snort3.service:
[Unit]
Description=Snort 3 NIDS Daemon
After=syslog.target network.target

[Service]
Type=simple
# Asegúrate de cambiar -i eth0 por tu interfaz (ej. wlan0)
ExecStart=/usr/local/bin/snort -c /usr/local/etc/snort/snort.lua -s 65535 -k none -l /var/log/snort -D -i eth0 -m 0x1b
ExecStop=/bin/kill -9 $MAINPID

[Install]
WantedBy=multi-user.target

Activar servicio:
sudo mkdir -p /var/log/snort
sudo chmod -R 5775 /var/log/snort
sudo chown -R snort:snort /var/log/snort
sudo systemctl enable snort3
sudo systemctl start snort3

FINAL: He decidido hacer esta guía porque hay muy poco tutorial sobre como instalar Snort en una Raspberry Pi.
