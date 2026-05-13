# 🎮 Sistema de Videovigilancia con una PS3

> **Autores:** Guillermo Ángel Conesa · Rodrigo Mauricio · Sandro Sánchez Maturana

Sistema de videovigilancia distribuido, de bajo coste y código abierto, construido sobre hardware reacondicionado: una PlayStation 3, un ordenador portátil y una torre de escritorio.

---

## Índice

1. [Introducción y Objetivos](#1-introducción-y-objetivos)
2. [Arquitectura de Hardware y Topología de Red](#2-arquitectura-de-hardware-y-topología-de-red)
3. [Instalación y Preparación de los Sistemas Operativos](#3-instalación-y-preparación-de-los-sistemas-operativos)
4. [Configuración de Red, Enrutamiento y Seguridad](#4-configuración-de-red-enrutamiento-y-seguridad)
5. [Infraestructura Cloud Privada y VPN](#5-infraestructura-cloud-privada-y-vpn)
6. [Sistema de Visión Artificial y Grabación](#6-sistema-de-visión-artificial-y-grabación)
7. [Automatización, Sincronización y Respaldo de Datos](#7-automatización-sincronización-y-respaldo-de-datos)
8. [Pruebas, Resultados y Demostración](#8-pruebas-resultados-y-demostración)
9. [Conclusiones y Mejoras Futuras](#9-conclusiones-y-mejoras-futuras)

---

## 1. Introducción y Objetivos

### 1.1. Justificación del proyecto

El constante avance tecnológico genera un ciclo de obsolescencia prematura en los equipos informáticos, contribuyendo al aumento global de la basura electrónica. Este proyecto transforma una consola de videojuegos descatalogada, un portátil y una torre genérica en una infraestructura integral de videovigilancia, demostrando que es posible desplegar soluciones de seguridad perimetral de bajo coste que:

- Reducen la huella ecológica.
- Eliminan la barrera de entrada económica.
- Garantizan la **soberanía de los datos** del usuario.
- Compiten en funcionalidad y privacidad con ecosistemas comerciales privativos.

### 1.2. Objetivos

**Objetivo principal:** Diseñar, configurar y desplegar una arquitectura de videovigilancia distribuida, funcional y segura, mediante hardware reacondicionado y herramientas de software libre, logrando un sistema autónomo de bajo coste.

**Objetivos secundarios:**

| # | Objetivo | Descripción |
|---|----------|-------------|
| 1 | **Visión Artificial** | Detectar movimiento y rostros en tiempo real con algoritmos optimizados para hardware limitado. |
| 2 | **Infraestructura de Red** | Configurar una topología segmentada con router, NAT y firewall sobre hardware reciclado. |
| 3 | **Nube Privada** | Orquestar almacenamiento híbrido con contenedores para sincronización y respaldo automatizado. |
| 4 | **Conectividad Externa** | Habilitar acceso remoto seguro mediante VPN tipo malla con cifrado de extremo a extremo. |

### 1.3. Alcance y limitaciones

**Alcance:** Monitorización de un entorno interior mediante un único flujo de vídeo. El procesamiento se delega en la Torre PC (edge computing), y las evidencias se comprimen y transitan hacia una nube privada accesible remotamente.

**Limitaciones operativas:**

| Nodo | Limitación |
|------|------------|
| **PS3** | 256 MB de RAM XDR (Cell BE). Descarta interfaces gráficas y bases de datos pesadas. |
| **Torre PC** | Capacidad de cómputo limitada: impide modelos de IA profundos o múltiples cámaras HD simultáneas. |
| **Red externa** | El ancho de banda de subida del ISP doméstico condiciona el streaming remoto y la descarga de evidencias. |

---

## 2. Arquitectura de Hardware y Topología de Red

### 2.1. Reacondicionamiento de Hardware

- **Migración a SSD en PS3:** El HDD mecánico original (5400 RPM, SATA I) presentaba latencias de escritura inasumibles para un servidor de evidencias. La sustitución por un SSD eliminó las latencias mecánicas y permitió que la memoria Swap (necesaria con solo 256 MB de RAM) fuera funcional sin bloquear el kernel durante la compresión.

- **Reciclaje de componentes:** El HDD extraído de la PS3 fue saneado mediante formateo de bajo nivel (eliminando particiones cifradas de GameOS) e instalado en la Torre PC como unidad secundaria para logs y copias de seguridad temporales.

### 2.2. Componentes del Sistema

#### A. Nodo de Visión Artificial — Torre PC

| Parámetro | Detalle |
|-----------|---------|
| **SO** | Xubuntu (Ubuntu + XFCE) |
| **IP** | `192.168.10.4` |
| **Función** | Detección facial, gestión del buffer de vídeo, servidor de streaming MJPEG |
| **Cámara** | Logitech HD (driver UVC nativo en Linux) |

#### B. Nodo de Almacenamiento — PlayStation 3

| Parámetro | Detalle |
|-----------|---------|
| **SO** | Red Ribbon Linux (Debian para PowerPC/PPC64), modo headless |
| **IP** | `192.168.10.5` |
| **Función** | Recepción de vídeos `.avi` por SSH, compresión y mantenimiento del histórico |

#### C. Nodo de Red, Cloud y Pasarela — Portátil

| Parámetro | Detalle |
|-----------|---------|
| **SO** | Lubuntu |
| **IP** | `192.168.10.1` (Gateway y DNS local) |
| **Función** | Router/NAT, Nextcloud (Docker), Firewall UFW, VPN Tailscale |

### 2.3. Topología de Red

```
Internet (WAN)
     │
     │ Wi-Fi (wlan0 / wlp2s0)
     │
┌────▼────────────┐
│    PORTÁTIL     │  192.168.10.1  ← Gateway, NAT, UFW, VPN, Nextcloud
└────┬────────────┘
     │ Ethernet (eth0 / enp1s0) — Cat 5e/6
     │
┌────▼────────────┐
│  SWITCH 5p      │
└──┬──────────┬───┘
   │          │
┌──▼──┐    ┌──▼──┐
│TORRE│    │ PS3 │
│.10.4│    │.10.5│
└─────┘    └─────┘
```

**Subred:** `192.168.10.0/24` — Direccionamiento IP estático para evitar conflictos al arrancar servicios.

El switch garantiza comunicación Full-Duplex, aislando el tráfico interno de videovigilancia del Wi-Fi doméstico y manteniendo la latencia de detección **< 100 ms**.

---

## 3. Instalación y Preparación de los Sistemas Operativos

### 3.1. Red Ribbon Linux en la PS3

La arquitectura Cell BE y los 256 MB de RAM XDR exigían una distribución Linux específica: **Red Ribbon Linux** (Debian para PPC64).

#### 3.1.1. Cuello de botella I/O → Migración a SSD

El entorno Live USB sobre el HDD mecánico original presentaba tiempos de espera de I/O excesivos y cuelgues en el entorno gráfico. La sustitución por un SSD resolvió el cuello de botella, haciendo funcional la partición Swap dentro del margen de los 256 MB de RAM.

#### 3.1.2. Fallo del instalador gráfico

El instalador gráfico estándar consumía la totalidad de la RAM durante la descompresión del sistema base, provocando un congelamiento total. La instalación automatizada no era viable en este hardware.

#### 3.1.3. Instalación manual por CLI y optimización del sistema de archivos

Se diseñó un proceso de instalación manual pura:

1. **Particionado manual** con herramientas de consola (partición raíz + Swap en el SSD).
2. **Migración de ext3 a ext2:** Se eliminó el *journaling* para suprimir las escrituras constantes en disco y el consumo extra de CPU/RAM, a cambio de requerir `fsck` manual ante un corte eléctrico.
3. **Clonación del sistema:** Copia recursiva exacta de la raíz del Live USB (`/`) al SSD.
4. **Configuración del gestor de arranque:** Se editó `kboot.conf` en **Petitboot** para apuntar a la nueva partición del SSD.

#### 3.1.4. Usuario `(live)user`

La clonación directa del sistema Live conservó el nombre de usuario `(live)user`. Se mantuvo en producción por estabilidad: renombrarlo en un sistema PowerPC frágil conlleva riesgo de corrupción de permisos. Al operar exclusivamente como servidor headless administrado por SSH con claves asimétricas, el nombre del usuario local no afecta a la seguridad ni al rendimiento.

---

## 4. Configuración de Red, Enrutamiento y Seguridad

El portátil con **Lubuntu** actúa como router y cortafuegos entre la WAN (Wi-Fi del ISP) y la LAN (switch con Torre y PS3).

### 4.1. Interfaces de red y Netplan

| Interfaz | Rol | Configuración |
|----------|-----|---------------|
| `wlp2s0` | WAN | DHCP (ISP) |
| `enp1s0` | LAN | IP estática `192.168.10.1/24` |

Configuración persistente en `/etc/netplan/`:

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    wlp2s0:
      dhcp4: yes
    enp1s0:
      dhcp4: false
      addresses:
        - 192.168.10.1/24
      nameservers:
        - 8.8.8.8
```

```bash
sudo netplan apply
```

### 4.2. IP Forwarding

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

### 4.3. NAT con iptables

```bash
# 1. Enmascaramiento para tráfico saliente por Wi-Fi
sudo iptables -t nat -A POSTROUTING -o wlp2s0 -j MASQUERADE

# 2. Permitir forwarding LAN → Internet
sudo iptables -A FORWARD -i enp1s0 -o wlp2s0 -j ACCEPT

# 3. Permitir tráfico de respuesta Internet → LAN
sudo iptables -A FORWARD -i wlp2s0 -o enp1s0 -m state --state ESTABLISHED,RELATED -j ACCEPT
```

### 4.4. Firewall UFW

Política por defecto: **denegación de todo el tráfico entrante**. Puertos habilitados en la interfaz LAN:

| Puerto | Protocolo | Servicio |
|--------|-----------|----------|
| 22 | TCP | SSH — administración remota |
| 53 | TCP/UDP | DNS — resolución de nombres en LAN |
| 80 | TCP | HTTP |
| 8080 | TCP | HTTP alternativo / proxies |

```bash
sudo ufw allow in on 192.168.10.0/24 to any port 22,80,8080 proto tcp
sudo ufw allow in on 192.168.10.0/24 to any port 53
sudo ufw enable
```

> `iptables` gestiona el NAT de forma transparente; `UFW` blinda el acceso directo a los servicios del portátil.

---

## 5. Infraestructura Cloud Privada y VPN

### 5.1. Docker en el Portátil

Se utiliza Docker para aislar servicios sin ensuciar el SO base.

```bash
sudo apt update && sudo apt upgrade
sudo apt install curl
# Añadir repositorio oficial de Docker e instalar:
# docker-ce, docker-ce-cli, containerd.io, docker-compose-plugin
sudo usermod -aG docker $USER        # Principio de mínimo privilegio
sudo systemctl enable docker
```

### 5.2. Nextcloud contenerizado

`docker-compose.yml`:

```yaml
version: '3.8'

services:
  db:
    image: mariadb:10.11
    container_name: nextcloud_db
    restart: always
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
    volumes:
      - db_data:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=usuario
      - MYSQL_PASSWORD=wo0FaIOsHfWYQl
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud

  app:
    image: nextcloud:latest
    container_name: nextcloud_app
    restart: always
    ports:
      - "8080:80"
    depends_on:
      - db
    volumes:
      - /mnt/nextcloud_drive/db_data:/var/lib/mysql
      - /mnt/nextcloud_drive/app_data:/var/www/html
    environment:
      - MYSQL_PASSWORD=wo0FaIOsHfWYQl
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_HOST=db

volumes:
  db_data:
  nextcloud_data:
```

**Configuración crítica — Dominios de confianza (`config/config.php`):**

Se añade la IP local del servidor y el rango de la VPN a `trusted_domains` para evitar el error *"Acceso a través de un dominio no confiable"*.

### 5.3. Tailscale VPN

Tailscale se instala de forma **nativa** (no contenerizado) para tener control directo sobre la pila de red del host, simplificando la gestión de UFW y el enrutamiento hacia Docker.

```bash
# Instalación mediante script oficial
curl -fsSL https://tailscale.com/install.sh | sh

# Autenticación del nodo
sudo tailscale up
```

El portátil recibe una IP estática en el rango `100.64.0.0/10` asignada por el servidor de coordinación de Tailscale.

### 5.4. Subnet Router

Permite que clientes externos (p. ej. un móvil con 4G) accedan a los nodos de la LAN usando sus IPs locales.

```bash
# 1. Habilitar IP Forwarding de forma persistente
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-tailscale.conf
sudo sysctl --system

# 2. Anunciar la subred local
sudo tailscale up --advertise-routes=192.168.10.0/24
```

> **Paso 3 — Aprobación Zero Trust:** La ruta debe habilitarse manualmente en la consola de administración web de Tailscale antes de propagarse.

---

## 6. Sistema de Visión Artificial y Grabación

Lenguaje: **Python**. Nodo: **Torre PC** con Xubuntu.

### 6.1. Dependencias

```bash
pip install opencv-python flask
```

| Librería | Uso |
|----------|-----|
| `opencv-python` (cv2) | Captura de vídeo, manipulación de frames, detección facial |
| `Flask` | Servidor de streaming MJPEG ligero |
| `time`, `datetime` | Marcas de tiempo en archivos de evidencia |

### 6.2. Fase 1 — Captura básica de vídeo

```python
import cv2

camara = cv2.VideoCapture(0)  # /dev/video0 en Linux

while True:
    exito, frame = camara.read()
    if not exito:
        break
    cv2.imshow('Vista', frame)
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

camara.release()
cv2.destroyAllWindows()
```

### 6.3. Fase 2 — Detección facial optimizada

Se descartaron modelos de Deep Learning (MediaPipe saturaba la CPU al 85%+) en favor del clasificador **Haar Cascade**, que estabiliza el consumo en **35–45% de CPU** manteniendo **20–24 FPS**.

```python
import cv2

clasificador = cv2.CascadeClassifier(
    cv2.data.haarcascades + 'haarcascade_frontalface_default.xml'
)

while True:
    exito, frame = camara.read()

    # Optimización: reducir tamaño y convertir a escala de grises
    frame_peq = cv2.resize(frame, (0, 0), fx=0.5, fy=0.5)
    gris = cv2.cvtColor(frame_peq, cv2.COLOR_BGR2GRAY)

    caras = clasificador.detectMultiScale(gris, scaleFactor=1.1, minNeighbors=5)

    for (x, y, w, h) in caras:
        x *= 2; y *= 2; w *= 2; h *= 2   # Escalar coordenadas al frame original
        cv2.rectangle(frame, (x, y), (x+w, y+h), (0, 0, 255), 2)
```

### 6.4. Lógica de retención — Temporizador de 5 segundos

Evita cientos de micro-vídeos de 1 segundo; todo el evento queda en un único archivo `.avi`.

```python
import cv2, time
from datetime import datetime

TIEMPO_ESPERA = 5.0
grabando = False
salida_video = None
tiempo_ultima_cara = 0

codec = cv2.VideoWriter_fourcc(*'XVID')

while True:
    exito, frame = camara.read()
    # ... (detección de caras → lista `caras`)

    if len(caras) > 0:
        tiempo_ultima_cara = time.time()
        if not grabando:
            nombre = datetime.now().strftime('%Y%m%d_%H%M%S') + '.avi'
            salida_video = cv2.VideoWriter(nombre, codec, 20.0, (640, 480))
            grabando = True
        salida_video.write(frame)
    else:
        if grabando:
            salida_video.write(frame)
            if time.time() - tiempo_ultima_cara > TIEMPO_ESPERA:
                salida_video.release()
                grabando = False
```

### 6.5. Fase 3 — Streaming MJPEG hacia la PS3

El navegador nativo de la PS3 no soporta WebRTC ni HTML5 avanzado. La solución es un servidor **MJPEG con Flask** compatible con navegadores legacy.

```python
from flask import Flask, Response
import cv2

app = Flask(__name__)

def generar_video():
    while True:
        exito, frame = camara.read()
        _, buffer = cv2.imencode('.jpg', frame)
        yield (b'--frame\r\n'
               b'Content-Type: image/jpeg\r\n\r\n' +
               buffer.tobytes() + b'\r\n')

@app.route('/video')
def video_feed():
    return Response(
        generar_video(),
        mimetype='multipart/x-mixed-replace; boundary=frame'
    )

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

> Accesible desde la PS3 en: `http://192.168.10.4:5000/video`

---

## 7. Automatización, Sincronización y Respaldo de Datos

### 7.1. Autenticación asimétrica SSH

```bash
# En PS3 y Portátil: generar par de claves RSA 4096 bits
ssh-keygen -t rsa -b 4096

# Inyectar clave pública en la Torre
ssh-copy-id usuario@192.168.10.4
```

En Ubuntu 24.04 fue necesario habilitar compatibilidad con claves RSA de la PS3:

```
# /etc/ssh/sshd_config (Torre)
PubkeyAcceptedAlgorithms +ssh-rsa
```

### 7.2. Sincronización Torre → PS3 (`sync_videos.sh`)

```bash
#!/bin/bash
rsync -avz --remove-source-files \
  -e ssh \
  --include="*.avi" \
  --exclude="*" \
  usuario@192.168.10.4:/ruta/origen/ /ruta/destino/
```

| Parámetro | Propósito |
|-----------|-----------|
| `-z` | Compresión en tránsito, reduce uso del switch |
| `--include="*.avi"` | Filtra exclusivamente archivos de vídeo |
| `--remove-source-files` | Borra el original en la Torre **solo** tras confirmar transferencia exitosa |

### 7.3. Sincronización Torre → Nextcloud (`sync_nextcloud.sh`)

Nextcloud indexa sus archivos en una base de datos interna: copiar directamente al volumen de Docker no basta.

```bash
#!/bin/bash
# 1. Transferir vídeos con rsync (igual que PS3)
rsync -avz --remove-source-files \
  -e ssh \
  --include="*.avi" --exclude="*" \
  usuario@192.168.10.4:/ruta/origen/ /mnt/nextcloud_drive/app_data/ruta/

# 2. Ajustar permisos para el usuario www-data del contenedor
chown -R 33:33 /mnt/nextcloud_drive/app_data/ruta/

# 3. Forzar re-indexación en Nextcloud
docker exec --user 33 nextcloud_app \
  php occ files:scan --path="ruta_especifica"
```

### 7.4. Automatización con Cron

```bash
# crontab -e (usuario root)

# Sincronización hacia PS3 cada 5 minutos
*/5 * * * * /ruta/sync_videos.sh > /dev/null 2>&1

# Sincronización hacia Nextcloud cada 10 minutos
*/10 * * * * /ruta/sync_nextcloud.sh > /dev/null 2>&1
```

---

## 8. Pruebas, Resultados y Demostración

### 8.1. Benchmarking de Recursos

#### Torre PC — Visión Artificial

| Algoritmo | CPU | FPS | Viabilidad |
|-----------|-----|-----|------------|
| MediaPipe | > 85% | < 10 | Frame dropping |
| **Haar Cascade** | **35–45%** | **20–24** | Estable |

- Lógica de retención de 5 segundos validada: archivos `.avi` se cierran correctamente al salir el sujeto del encuadre.

#### PlayStation 3 — Almacenamiento

| Estado | RAM consumida |
|--------|---------------|
| Idle | ~85 MB |
| Recepción + compresión | ~210 MB |

Margen de seguridad respetado (256 MB límite). La migración a **ext2** eliminó la sobrecarga del journaling, permitiendo rsync y compresión fluidos.

#### Portátil — Red y Contenedores

| Métrica | Valor |
|---------|-------|
| RAM Nextcloud (Docker) | ~400 MB |
| Latencia interna (NAT) | < 2 ms |

### 8.2. Validación de Acceso Remoto

#### Subnet Router + Tailscale (red 4G/5G)

- Ping exitoso desde móvil externo a Torre (`192.168.10.4`) y PS3 (`192.168.10.5`).
- Streaming MJPEG accesible desde el navegador del móvil con latencia media de **150–200 ms** bajo 4G — óptimo para vigilancia doméstica.
- Nuevos vídeos aparecen automáticamente en la app móvil de Nextcloud tras el escaneo `occ files:scan`.

---

## 9. Conclusiones y Mejoras Futuras

### 9.1. Conclusiones

- **Sostenibilidad:** Dispositivos como la PS3 o un portátil antiguo pueden extender su ciclo de vida con software libre, reduciendo el e-waste y promoviendo el Green Computing.
- **Optimización de recursos críticos:** Superar la barrera de 256 MB de RAM en una arquitectura PowerPC mediante la eliminación de entornos gráficos, sistema de archivos ext2 y SSD demuestra el valor de la ingeniería de sistemas aplicada con criterio.
- **Soberanía digital:** Los datos permanecen exclusivamente bajo control del usuario, sin depender de servidores de terceros ni abrir puertos inseguros en el router doméstico.
- **Resiliencia distribuida:** La segmentación de tareas (visión en Torre, almacenamiento en PS3, pasarela en Portátil) hace que el fallo de un nodo no comprometa a los demás.

### 9.2. Mejoras Futuras

#### Capa de Inteligencia Artificial
- Incorporar un acelerador hardware USB (ej. **Google Coral NPU**) para ejecutar **YOLOv8** en tiempo real, permitiendo clasificar objetos (personas, animales, vehículos) y eliminar falsos positivos.

#### Escalabilidad de Red
- Soporte **multicámara** mediante protocolo **RTSP**.
- Implementación de un IDS (**Suricata**) en el portátil para detección de anomalías en el tráfico.

#### Alertas e IoT
- **Bot de Telegram / API de WhatsApp** para notificaciones push con captura del rostro detectado.
- Integración con **Home Assistant** para disparar acciones físicas (luces, alarmas) ante una intrusión confirmada.

#### Eficiencia Energética
- **Wake-on-LAN (WoL):** mantener PS3 y Torre en suspensión, activándolas solo cuando el portátil detecte actividad relevante.

### 9.3. Reflexión Final

> *La innovación no siempre reside en la adquisición de la última tecnología, sino en la capacidad de orquestar recursos existentes de manera inteligente. La transformación de una consola de juegos en un servidor de seguridad es un testimonio de la versatilidad del software libre y de cómo la ingeniería puede ofrecer soluciones de alta fidelidad con una inversión económica mínima.*

---

## 🛠️ Stack Tecnológico

| Categoría | Tecnología |
|-----------|------------|
| SO Torre | Xubuntu (Ubuntu + XFCE) |
| SO PS3 | Red Ribbon Linux (Debian PPC64) |
| SO Portátil | Lubuntu |
| Visión Artificial | Python, OpenCV (Haar Cascade), Flask (MJPEG) |
| Contenedores | Docker, Docker Compose |
| Nube Privada | Nextcloud + MariaDB |
| VPN | Tailscale (WireGuard) |
| Firewall / NAT | UFW, iptables |
| Red | Netplan, rsync, SSH (claves RSA 4096) |
| Automatización | Cron (Bash scripts) |

---

## 📄 Licencia

Este proyecto es de carácter académico (TFG). Todos los componentes de software utilizados son de código abierto.
