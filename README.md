# 📡 Conmutación y Teletráfico — Segundo Parcial
**Fundación Universitaria Compensar**

---

## 📋 Contenido del Repositorio

```
├── parcial_conmutacion.ipynb   ← Notebook principal (Google Colab)
├── README.md                   ← Este archivo
```

---

# PARTE 1 — CONCEPTUAL

## 1.1 NetFlow vs sFlow

### NetFlow
Trabaja a nivel de **flujo completo**: agrupa todos los paquetes con la misma 5-tupla en una entrada de caché stateful. El router acumula bytes/paquetes y exporta el registro cuando el flujo expira (active timer ~60 s, idle timer ~15 s).

**Limitación a 100 Gbps:**
```
100 × 10⁹ bits/s ÷ (512 × 8 bits/pkt) ≈ 24.4 millones pkt/s
→ Millones de entradas de estado simultáneas → desbordamiento de tabla
```

### sFlow
Trabaja a nivel de **paquete individual**: toma 1 de cada N paquetes aleatoriamente (stateless), copia el encabezado y lo exporta inmediatamente. Vive en el **ASIC** del switch.

```
Ratio 1:10.000 a 100 Gbps → ~2.440 muestras/s → overhead < 0.001%
```

### ¿Por qué elegir sFlow para Top Talkers a 100 Gbps?

| Criterio | NetFlow | sFlow |
|----------|---------|-------|
| Estado en dispositivo | Sí (millones de entradas) | No (stateless) |
| Overhead CPU/RAM | Alto → riesgo de pérdida de flujos | Mínimo → ASIC puro |
| Latencia de reporte | Al expirar el flujo (hasta 60 s) | Casi tiempo real |
| Escalabilidad | Limitada | Lineal, sin límite teórico |
| Precisión | Exacta | Estadística (~±5%) |

**La ley de grandes números garantiza** que un flujo con el 30% del tráfico aparecerá en el ~30% de las muestras → identificable como top talker con alta confianza estadística.

---

## 1.2 Los 5 Campos de la 5-Tuple

| # | Campo | Layer | Ejemplo | Para medir aplicaciones |
|---|-------|-------|---------|------------------------|
| 1 | IP origen | L3 | 192.168.1.10 | Identifica el host emisor |
| 2 | IP destino | L3 | 10.0.0.5 | Identifica el host receptor |
| 3 | Puerto origen | L4 | 54321 (efímero) | Cambia en cada sesión |
| 4 | **Puerto destino** | L4 | **443 (HTTPS)** | **Clave para clasificar app** |
| 5 | Protocolo IP | L3 | 6=TCP, 17=UDP | Tipo de transporte |

### Puertos para HTTP vs SSH

```
HTTP  → puerto destino 80  (TCP)
HTTPS → puerto destino 443 (TCP)
SSH   → puerto destino 22  (TCP)
YOLO  → puerto destino 5555 (UDP, en este laboratorio)
```

El **colector NetFlow** debe filtrar por `dst_port` para separar el tráfico por aplicación.

---

## 1.3 IP Accounting — Interpretación y Asimetría

```
Source         Destination    Packets   Bytes
192.168.1.10   10.0.0.5       1500      120000
192.168.1.10   10.0.0.8        800       64000
10.0.0.5       192.168.1.10     50        4000
```

### Interpretación
- `192.168.1.10` es el **top talker**: envía 2300 paquetes en total, es la fuente más activa.
- Tamaño promedio por paquete = 120000/1500 = **80 bytes** → paquetes de cabecera pequeña (posible flood o escaneo, no transferencias de archivos grandes).

### Asimetría extrema: 1500 enviados vs 50 recibidos (ratio 30:1)

Indicaría alguna de estas situaciones:

1. **Exfiltración de datos** — el host interno envía volúmenes masivos hacia fuera (data leak).
2. **Ataque DDoS / UDP flood** — envío masivo sin esperar respuesta (SYN flood, UDP flood).
3. **Escaneo de red** — el host sondea múltiples destinos sin obtener respuesta.
4. **Backup asimétrico** — transferencia legítima de datos (pero aun así requiere validación).

> ⚠️ Un ratio > 10:1 en paquetes enviados vs recibidos es una señal de alerta en cualquier sistema de monitoreo de tráfico.

---

# PARTE 2.a — Arquitectura Colab: YOLO + NetFlow

## Diagrama de Flujo de Datos

```
┌──────────────────────────────────────────────────────────────────┐
│                    GOOGLE COLAB (host único)                     │
│                                                                  │
│  ┌─────────────────────┐    UDP :5555    ┌──────────────────┐   │
│  │  CONTENEDOR DOCKER  │ ─────────────→ │ COLECTOR NetFlow  │   │
│  │                     │                │  (127.0.0.1:5555) │   │
│  │  📹 test.mp4        │                └────────┬─────────┘   │
│  │  🧠 YOLOv8n         │                         │              │
│  │  🔌 socket UDP      │                    nfstream             │
│  │  IP: 127.0.0.1      │                         │              │
│  └─────────────────────┘                ┌────────▼─────────┐   │
│           │                             │  DASHBOARD       │   │
│           │ tráfico capturado           │  matplotlib      │   │
│           ▼                             │  Top Talkers     │   │
│  ┌─────────────────────┐               │  Anomalías       │   │
│  │  VM VIRTUAL (sim.)  │               └──────────────────┘   │
│  │  softflowd/iptables │                                        │
│  │  IP Accounting      │                                        │
│  └─────────────────────┘                                        │
└──────────────────────────────────────────────────────────────────┘
```

## Comunicación Contenedor YOLO ↔ VM

El contenedor envía resultados de detección vía **UDP** hacia la IP de la VM:
```python
sock.sendto(payload_json, ('10.0.0.100', 9999))
```

El tráfico pasa por el **switch virtual (Linux bridge `docker0`)**, donde `softflowd` lo captura y exporta flujos NetFlow al colector.

```bash
# softflowd en la VM captura tráfico de la interfaz puente
softflowd -i eth0 -n 127.0.0.1:2055 -v 9
```

## Regla IP Accounting con iptables

```bash
# Medir tráfico entre subred del contenedor (172.17.0.0/16) y la VM
iptables -A FORWARD -s 172.17.0.0/16 -d 10.0.0.100 -j ACCEPT
iptables -A FORWARD -s 10.0.0.100 -d 172.17.0.0/16 -j ACCEPT

# Resetear y leer
iptables -Z
# ... (esperar N minutos) ...
iptables -L FORWARD -v -n | grep -E "10.0.0.100|172.17"
```

---

# PARTE 2.b — Arquitectura Estación de Trenes

## Los 5 Contenedores

| Contenedor | Función | Modelo YOLO | IP | Puerto |
|-----------|---------|-------------|-----|--------|
| C1 | Lectura de placas (YOLO + OCR) | YOLOv8m + EasyOCR | 10.0.0.10 | 9001 |
| C2 | Conteo parqueadero | YOLOv8s | 10.0.0.11 | 9002 |
| C3 | Detección personas (aforo) | YOLOv8n | 10.0.0.12 | 9003 |
| C4 | Detección animales | YOLOv8n (custom) | 10.0.0.13 | 9004 |
| C5 | Objetos perdidos | YOLOv8m (custom) | 10.0.0.14 | 9005 |

## Diagrama de Arquitectura Completo

```
FUENTE DE VIDEO (RTSP / archivos)
         │
    rtsp://cam-server/stream[1-5]
         │
┌────────▼─────────────────────────────────────────────┐
│              Open vSwitch (10.0.0.0/24)              │
│  Subred contenedores: 10.0.0.10 – 10.0.0.14          │
│  VMs:                 10.0.0.100 / 10.0.0.101         │
└──┬───┬───┬───┬───┬──────────────┬───────────────┬────┘
   │   │   │   │   │              │               │
   C1  C2  C3  C4  C5           VM1             VM2
 .10 .11 .12 .13 .14          .100            .101
   │   │   │   │   │        (Colector       (Respaldo
   │ UDP video │   │         principal)      redundante)
   └──────────────┘              │               │
     Metadata TCP ────────────→  │  Replicación  │
                                 └──────────────→┘
                                      TCP sync

Flujos de datos:
  ══════ Video anotado (UDP, ~12 Mbps por contenedor)
  ─────  Metadata JSON (TCP, ~0.016 Mbps por contenedor)
  ──▶──  NetFlow exports (UDP 2055) → Colector NetFlow
```

## Cálculo de Throughput

```
Por contenedor:
  Video    = 30 fps × 50 KB/frame × 8 = 12.0 Mbps (UDP)
  Metadata = 10 det/s × 200 B × 8     =  0.016 Mbps (TCP)
  Total    ≈ 12.016 Mbps por contenedor

Total 5 contenedores:
  5 × 12.016 Mbps ≈ 60.08 Mbps
  
  Red requerida: ≥ 100 Mbps (con overhead QoS del 40%)
```

## UDP vs TCP para Video (latencia 2 ms, jitter ±1 ms)

**UDP** es más adecuado porque:
1. TCP retransmite paquetes perdidos → frame llega tarde → congelamiento visible
2. TCP congestion control reduce la tasa de envío → jitter adicional
3. UDP permite **jitter buffer** en el receptor que absorbe ±1 ms sin problema

**Mitigación del jitter:**
- Buffer de dejitter de 2-3 frames (66-100 ms a 30 fps)
- Usar **RTP sobre UDP** con números de secuencia y timestamps
- QoS DSCP EF (Expedited Forwarding) para priorizar video

## Regla NetFlow 5-tuple por Contenedor

```
Contenedor → VM1 (colector):
  src_ip=10.0.0.1X, dst_ip=10.0.0.100, src_port=*, dst_port=900X, proto=17(UDP)

Para detectar top talker en 5 minutos:
  nfdump -r /tmp/flujos.nf -t '2024-01-15 10:00 - 10:05' -s srcip/bytes
```

---

# PARTE EMPÍRICA — Pasos 1 a 5

## Paso 1: YOLO + Generación de Tráfico UDP

```python
# Por cada frame procesado, se envía 1 paquete UDP al colector
payload = json.dumps({'ts': time.time(), 'frame': N, 'detecciones': lista}).encode()
sock.sendto(payload, ('127.0.0.1', 5555))
```

La 5-tuple generada:
```
IP_origen=127.0.0.1, IP_destino=127.0.0.1,
puerto_origen=<efímero>, puerto_destino=5555, protocolo=17 (UDP)
```

## Paso 2: Captura con tcpdump + Análisis de 5-tuple

```bash
# Capturar mientras corre YOLO
sudo tcpdump -i lo -c 200 udp port 5555 -w /tmp/flujos.pcap &

# Analizar con nfstream (Python) o nfdump
nfdump -r /tmp/flujos.nf -q -o "fmt:%ts %sa %da %sp %dp %pr %pkt %byt"
```

Salida ejemplo:
```
2024-01-15 10:00:00.123  127.0.0.1  127.0.0.1  54321  5555  17  30  2100
                         %sa        %da        %sp    %dp   %pr %pkt %byt
```

## Paso 3: Top Talkers agrupados por IP origen

```python
# Con nfdump
result = subprocess.run(['nfdump', '-r', '/tmp/flujos.nf', '-q', '-o', 'fmt:%sa %pkt'], ...)
# Agrupar y ordenar por paquetes → top 3
```

## Paso 4: IP Accounting con iptables

```bash
# Crear regla + resetear contadores
sudo iptables -A INPUT -p udp --dport 5555 -j ACCEPT
sudo iptables -Z

# Después de correr YOLO, leer estadísticas
sudo iptables -L INPUT -v -n
# → 30 paquetes, 2100 bytes

# Ver SOLO tráfico UDP al puerto 5555
sudo iptables -L -v -n | grep "dpt:5555"
```

## Paso 5: Anomalía y Detección

```bash
# Flood Python (simula hping3)
# Genera 500 paquetes en < 1 segundo → aumento anómalo en contadores
```

Resultado esperado:
```
Antes del flood:  30 paquetes
Después del flood: 530 paquetes → ANOMALÍA: +500 paquetes en < 1s
Velocidad: ~5000 pkt/s → posible ataque UDP flood
```

---

## Respuestas a las Preguntas Finales

### P1: 5-tuple del flujo capturado con nfdump
```
IP origen:      127.0.0.1
IP destino:     127.0.0.1
Puerto origen:  54321 (efímero, asignado por el SO)
Puerto destino: 5555
Protocolo:      17 (UDP)
```

### P2: Bytes contabilizados con iptables
```bash
sudo iptables -L INPUT -v -n
# Chain INPUT: 30 pkts, 2100 bytes para udp dpt:5555

# Solo puerto 5555:
sudo iptables -L -v -n | grep "dpt:5555"
```

### P3: Campo para diferenciar HTTP vs YOLO
El **puerto destino** (campo 4 de la 5-tuple):
- HTTP → dst_port = 80
- YOLO → dst_port = 5555

### P4: Muestreo estilo sFlow (1 de cada 10)
```python
RATIO = 10
if frame_num % RATIO == 0:
    sock.sendto(payload, (COLECTOR_IP, COLECTOR_PORT))
```
**Ventaja en enlace lento:** Reduce el tráfico de telemetría en 90% (de 6 KB/s a 600 B/s), manteniendo representatividad estadística. Esencial para redes IoT, 4G saturadas, o cuando se tienen cientos de cámaras.

---

## ▶️ Cómo Ejecutar en Google Colab

1. Abrir [Google Colab](https://colab.research.google.com/)
2. `Archivo → Subir notebook` → seleccionar `parcial_conmutacion.ipynb`
3. Ejecutar celdas **en orden** (Ctrl+Enter o Runtime → Run All)
4. Todos los resultados, gráficas y análisis se generan automáticamente.

> ℹ️ Tiempo estimado de ejecución completa: ~5-8 minutos (depende de la descarga del modelo YOLO).
