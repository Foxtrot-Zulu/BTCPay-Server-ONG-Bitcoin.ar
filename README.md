## Documentación de Arquitectura e Instalación — BTCPay Server ONG BITCOIN.AR

### Descripción General

Este deployment separa la infraestructura Bitcoin entre múltiples entornos:

- VM Azure:
  - BTCPay Server
  - NBXplorer
  - phoenixd
  - PostgreSQL
  - nginx reverse proxy
  - Tailscale

- Nodo Bitcoin externo:
  - Bitcoin Core
  - Tailscale
  - Infraestructura existente

La arquitectura fue diseñada para:

- Evitar almacenar la blockchain en Azure
- Mantener RPC/ZMQ/P2P privados via Tailscale
- Conectar Azure con el nodo externo usando ACLs de Tailscale
- Integrar una wallet de phoenixd para Lightning Network

---

## Arquitectura Final

```text
                       ┌─────────────────────────┐
                       │    Red Bitcoin P2P      │
                       └──────────┬──────────────┘
                                  │
                 ┌────────────────────────────────────┐
                 │  Bitcoin Core (Externo)            │
                 │------------------------------------│
                 │ - Bitcoin Core                     │
                 │ - Tailscale                        │
                 │ - Fulcrum / infraestructura extra  │
                 └───-───────────┬────────────────────┘
                                 │
                   RPC/ZMQ/P2P vía Tailscale
                                 │
                                 ▼
                 ┌────────────────────────────────────┐
                 │         Azure BTCPay VM            │
                 │------------------------------------│
                 │ - BTCPay Server                    │
                 │ - NBXplorer                        │
                 │ - phoenixd                         │
                 │ - PostgreSQL                       │
                 │ - nginx                            │
                 │ - Tailscale                        │
                 └────────────────────────────────────┘
```

---

## Instalación BTCPay Server

### Directorio Base

```bash
/root/BTCPayServer
```

### Red Docker

```text
generated_default
```

### Contenedores Principales

| Servicio | Contenedor |
|---|---|
| BTCPay Server | generated_btcpayserver_1 |
| NBXplorer | generated_nbxplorer_1 |
| PostgreSQL | generated_postgres_1 |
| nginx | nginx |
| phoenixd | phoenixd |

---

## Configuración Bitcoin Core

### Archivo bitcoin.conf

```bash
nodo externo: /media/datadrive/.bitcoin/bitcoin.conf
```

### Configuración RPC Final

```conf
server=1

rpcbind=127.0.0.1
rpcbind=192.168.20.40
rpcbind=<IP_TAILSCALE_NODO_BITCOIN>

rpcallowip=127.0.0.1
rpcallowip=10.0.0.0/8
rpcallowip=172.0.0.0/8
rpcallowip=192.168.20.0/24
rpcallowip=100.0.0.0/8

whitelist=<IP_TAILSCALE_AZURE>
```

### P2P Bind

```conf
bind=192.168.20.40:8333
bind=<IP_TAILSCALE_NODO_BITCOIN>:8333
```

### ZMQ

```conf
zmqpubrawblock=tcp://<IP_TAILSCALE_NODO_BITCOIN>:28332
zmqpubrawtx=tcp://<IP_TAILSCALE_NODO_BITCOIN>:28333
```

### Inicio

Bitcoin Core debe arrancarse siempre con el datadir explícito:

```bash
bitcoind -daemon -datadir=/media/datadrive/.bitcoin
```

---

## Configuración Tailscale

### Objetivo

Utilizar Tailscale para:
- RPC Bitcoin Core
- ZMQ Bitcoin Core
- P2P Bitcoin Core (conexión NBXplorer → nodo)

---

## ACLs Tailscale

### Desde el administrador web de Tailscale, editar la config ACL:

```json
{
  "tagOwners": {
    "tag:bitcoin-node": ["autogroup:admin"],
    "tag:btcpay-azure": ["autogroup:admin"]
  },

  "grants": [
    {
      "src": ["tag:btcpay-azure"],
      "dst": ["tag:bitcoin-node"],
      "ip": ["*:8332", "*:8333", "*:28332", "*:28333"]
    }
  ],

  "ssh": [
    {
      "action": "check",
      "src": ["autogroup:member"],
      "dst": ["autogroup:self"],
      "users": ["autogroup:nonroot", "root"]
    }
  ]
}
```

---

## Levantar Tailscale en ambos entornos

### Nodo Bitcoin

```bash
sudo tailscale up --advertise-tags=tag:bitcoin-node
```

### Azure BTCPay

```bash
sudo tailscale up --advertise-tags=tag:btcpay-azure
```

---

## Configuración NBXplorer

### docker-compose.generated.yml

Parámetros importantes:

```yaml
NBXPLORER_BTCRPCURL: http://<IP_TAILSCALE_NODO_BITCOIN>:8332/

NBXPLORER_BTCNODEENDPOINT: <IP_TAILSCALE_NODO_BITCOIN>:8333

NBXPLORER_BTCRPCUSER: <BTCRPCUSER>

NBXPLORER_BTCRPCPASSWORD: <BTCRPCPASSWORD>
```

### Resultado Final

- RPC vía Tailscale
- P2P Bitcoin vía Tailscale
- ZMQ vía Tailscale
- Sincronización estable

---

## Integración phoenixd

### Instalada en:

```bash
/root/BTCPayServer/volumes/phoenixd
```

### Archivos Importantes Wallet

| Archivo | Función |
|---|---|
| seed.dat | Seed Lightning |
| phoenix.mainnet.*.db | Base de datos wallet |
| phoenix.conf | Configuración y passwords API |
| phoenix.log | Logs runtime |

---

## Contenedor phoenixd

### Deployment Final

Phoenixd está integrado en el `docker-compose.generated.yml` como servicio:

```yaml
phoenixd:
  restart: unless-stopped
  container_name: phoenixd
  image: acinq/phoenixd:latest
  volumes:
    - /root/BTCPayServer/volumes/phoenixd:/phoenix/.phoenix
```

### Importante

Mount correcto:

```text
/phoenix/.phoenix
```

NO montar directamente:

```text
/phoenix
```

Eso rompe la imagen porque oculta el binario interno.

---

## Verificación phoenixd

### Indicadores Startup Correcto

```text
connected to lightning peer
listening on http://0.0.0.0:9740
```

---

## Configuración Lightning BTCPay

### Tipo Lightning

```text
Custom
```

### Connection String

```text
type=phoenixd;server=http://phoenixd:9740/;password=<HTTP_PASSWORD>
```

Como:
- BTCPay y phoenixd comparten la misma red Docker
- Docker DNS resuelve automáticamente `phoenixd`

No es necesario exponer puertos públicamente.

Deployment final:
- acceso interno Docker únicamente
- API Lightning no expuesta públicamente

---

## Backups

### Rutas Críticas

#### BTCPay

```bash
/root/BTCPayServer/.env
/root/BTCPayServer/btcpayserver-docker/Generated/docker-compose.generated.yml
```

#### phoenixd

```bash
/root/BTCPayServer/volumes/phoenixd
```

Especialmente:
- seed.dat
- *.db
- phoenix.conf

#### Nodo Bitcoin Core

```bash
/media/datadrive/.bitcoin/bitcoin.conf
```

### Comando Backup DB

```bash
docker exec generated_postgres_1 pg_dump -U postgres btcpayservermainnet > /root/backup_btcpay_$(date +%Y%m%d_%H%M).sql
```

---

## Monitoreo Recomendado

### BTCPay

```bash
docker logs generated_btcpayserver_1 --tail 100
```

### NBXplorer

```bash
docker logs generated_nbxplorer_1 --tail 100
```

### phoenixd

```bash
docker logs phoenixd --tail 100
```

---

## Seguridad

- No subas **`.env`** ni `volumes/**` con claves a repositorios públicos.
- En producción, **HTTPS** delante de BTCPay (nginx) y restringe **9740** si no la necesitas expuesta.
- Antes de actualizar versiones, lee notas de migración de BTCPay Server y NBXplorer.
- Luego de cada `btcpay-update.sh`, verificar que el hook post-update de Vaultwarden se ejecutó correctamente.

---

## Lecciones Aprendidas

### Bitcoin Core sin datadir explícito

Al reiniciar Bitcoin Core sin `-datadir`, arranca desde `~/.bitcoin` (bloque génesis).
Siempre usar:

```bash
bitcoind -daemon -datadir=/media/datadrive/.bitcoin
```

Recomendado: configurar el servicio systemd con el flag `-datadir` para evitar este problema.

### Tailscale para todo el tráfico NBXplorer

Todo el tráfico entre NBXplorer y Bitcoin Core va por Tailscale: RPC, ZMQ y P2P.

---

## Estado Final

Todos los componentes funcionando:

- BTCPay Server
- NBXplorer (RPC + ZMQ + P2P via Tailscale)
- Bitcoin Core remoto
- phoenixd Lightning
- Facturación Lightning
- Pagos Lightning testeados correctamente
- Vaultwarden integrado con nginx BTCPay (notas a la intalación de vaultwarden: https://github.com/Foxtrot-Zulu/vaultwarden-bitcoin.ar/blob/main/README.md)

