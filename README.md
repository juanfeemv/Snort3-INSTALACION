# 🚨 Instalación de **Snort 3** en Raspberry Pi 5  
Guía actualizada **2025**, probada en ARM64 y libre de los típicos errores de memoria (💥 *SIGABRT*) al compilar Snort en Raspberry Pi.

> Esta guía existe porque prácticamente **no hay tutoriales completos** sobre Snort 3 en Raspberry Pi… así que aquí tienes uno funcional y probado.

---

## 🧰 Prerrequisitos

- **Hardware:** Raspberry Pi 5 (recomendado 8GB RAM).  
- **OS:** Raspberry Pi OS 64-bit.  
- **Red:** Conexión a internet.  

---

## 🛡️ ¿Qué es Snort?

Snort es un sistema IDS/IPS que analiza el tráfico de red en tiempo real para detectar actividades maliciosas. Es flexible, potente y perfecto para proyectos de ciberseguridad en Raspberry Pi.

---

# 1️⃣ Instalación de dependencias

Actualiza el sistema e instala todo lo necesario:

```bash
sudo apt update && sudo apt upgrade -y

sudo apt install -y build-essential autotools-dev libdumbnet-dev \
libluajit-5.1-dev libpcap-dev zlib1g-dev pkg-config libhwloc-dev \
cmake liblzma-dev openssl libssl-dev cpputest libsqlite3-dev \
libtool uuid-dev git autoconf bison flex libcmocka-dev \
libnetfilter-queue-dev libunwind-dev libmnl-dev libpcre2-dev
```

---

# 2️⃣ Compilar e instalar **LibDAQ** 📦

```bash
mkdir -p ~/snort_src
cd ~/snort_src
git clone https://github.com/snort3/libdaq.git
cd libdaq
./bootstrap
./configure
make
sudo make install
```

---

# 3️⃣ Compilar e instalar **Snort 3** 🐍

⚠️ *Importante:* En Raspberry Pi 5 (ARM64), `tcmalloc` provoca **SIGABRT**. Lo compilamos sin ella.

```bash
cd ~/snort_src
git clone https://github.com/snort3/snort3.git
cd snort3

# Configuración SIN tcmalloc
./configure_cmake.sh --prefix=/usr/local

cd build
make -j$(nproc)
sudo make install
sudo ldconfig
```

Verifica:

```bash
snort -V
```

---

# 4️⃣ Configuración de reglas 📜

Crear estructura de directorios:

```bash
sudo mkdir -p /usr/local/etc/rules
sudo mkdir -p /usr/local/etc/so_rules
sudo mkdir -p /usr/local/etc/lists

sudo touch /usr/local/etc/rules/local.rules
sudo touch /usr/local/etc/lists/default.blocklist
sudo touch /usr/local/etc/lists/default.allowlist
```

Descargar las reglas de comunidad:

```bash
cd ~/snort_src
wget https://www.snort.org/downloads/community/snort3-community-rules.tar.gz
tar xzf snort3-community-rules.tar.gz
sudo cp snort3-community-rules/snort3-community.rules /usr/local/etc/rules/
```

Editar `snort.lua`:

```lua
ips =
{
    include = {
        "/usr/local/etc/rules/snort3-community.rules",
        "/usr/local/etc/rules/local.rules"
    },
    variables = default_variables
}
```

---

# 5️⃣ Probar Snort (ejemplo ICMP) 🧪

Añade una regla simple en `local.rules`:

```bash
alert icmp any any -> any any (msg:"Ping detectado"; sid:1000001; rev:1;)
```

Ejecuta Snort en consola:

```bash
sudo snort -c /usr/local/etc/snort/snort.lua -i eth0 -A alert_fast
```

*Cambia `eth0` por tu interfaz de red real.*

---

# 6️⃣ Ejecutar Snort como servicio (systemd) ⚙️

Crear usuario:

```bash
sudo useradd -r -s /usr/sbin/nologin -M -c SNORT_IDS snort
```

Crear servicio:

```bash
sudo nano /etc/systemd/system/snort3.service
```

Pega:

```
[Unit]
Description=Snort 3 NIDS Daemon
After=syslog.target network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/snort -c /usr/local/etc/snort/snort.lua -s 65535 -k none -l /var/log/snort -D -i eth0 -m 0x1b
ExecStop=/bin/kill -9 $MAINPID

[Install]
WantedBy=multi-user.target
```

Activar:

```bash
sudo mkdir -p /var/log/snort
sudo chmod -R 5775 /var/log/snort
sudo chown -R snort:snort /var/log/snort

sudo systemctl enable snort3
sudo systemctl start snort3
```

---

# 🎉 Conclusión

Esta guía existe porque apenas hay documentación clara y funcional sobre cómo instalar **Snort 3 en Raspberry Pi 5**… y después de pelear con dependencias, errores ARM64 y `tcmalloc`, por fin queda una versión *estable y reproducible*.  
