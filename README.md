# Documentación de Arquitectura e Instalación — BTCPay Server ONG BITCOIN.AR

### Descripción General

Este deployment separa la infraestructura Bitcoin entre múltiples entornos:

- VM Azure:
  - BTCPay Server
  - NBXplorer
  - phoenixd
  - Contenedor Tor
  - PostgreSQL
  - nginx reverse proxy

- Nodo Bitcoin externo:
  - Bitcoin Core
  - Hidden Services Tor
  - Tailscale
  - Infraestructura existente

La arquitectura fue diseñada para:

- Evitar almacenar la blockchain en Azure
- Mantener el tráfico P2P Bitcoin sobre Tor
- Mantener RPC/ZMQ privados
- Conectar Azure con el nodo externo usando ACLs de Tailscale
- Integrar una wallet de phoenixd para Lighning Network

---

## Arquitectura Final

```text
                       ┌─────────────────────────┐
                       │    Red Bitcoin P2P      │
                       └──────────┬──────────────┘
                                  │
                               Tor P2P
                                  │
                                  ▼
                 ┌────────────────────────────────────┐
                 │  Bitcoin Core (Externo)            │
                 │------------------------------------│
                 │ - Bitcoin Core                     │
                 │ - Hidden Services Tor              │
                 │ - Tailscale                        │
                 │ - Fulcrum / infraestructura extra  │
                 └───-───────────┬────────────────────┘
                                 │
                       RPC/ZMQ vía Tailscale
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
                 │ - Contenedor Tor                   │
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
| Tor | tor |
| nginx | nginx |
| phoenixd | phoenixd |

---

## Configuración Bitcoin Core

### Archivo bitcoin.conf

```bash
nodo externo: /media/datadrive/.bitcoin/bitcoin.conf
```

## Configuración RPC Final

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

## ZMQ

```conf
zmqpubrawblock=tcp://<IP_TAILSCALE_NODO_BITCOIN>:28332
zmqpubrawtx=tcp://<IP_TAILSCALE_NODO_BITCOIN>:28333
```

---

## Hidden Services Tor

### Configuración torrc

```conf
HiddenServiceDir /var/lib/tor/bitcoin-rpc/
HiddenServiceVersion 3
HiddenServicePort 8332 127.0.0.1:8332
HiddenServicePort 8333 127.0.0.1:8333
HiddenServicePort 28332 127.0.0.1:28332
HiddenServicePort 28333 127.0.0.1:28333
```

### Endpoint Onion Bitcoin

Usado por NBXplorer:

```text
<ONION_BITCOIN>:8333
```

---

## Configuración Tailscale

### Objetivo

Utilizar:
- Tor para tráfico P2P Bitcoin
- Tailscale únicamente para RPC/ZMQ

Esto evita:
- problemas de estabilidad RPC-over-onion
- errores SOCKS/.NET dentro de NBXplorer

---

## ACLs Tailscale

### Luego de instalar Tailscale en ambos entornos, desde el administrador web de tailscale, editar la config ACL: 

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
      "ip": ["*:8332", "*:28332", "*:28333"]
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

NBXPLORER_BTCNODEENDPOINT: <ONION_BITCOIN>:8333

NBXPLORER_SOCKSENDPOINT: tor:9050
```

### Resultado Final

- RPC vía Tailscale
- P2P Bitcoin vía Tor
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

```bash
docker run -d \
  --name phoenixd \
  --restart unless-stopped \
  --network generated_default \
  -v /root/BTCPayServer/volumes/phoenixd:/phoenix/.phoenix \
  acinq/phoenixd:latest
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
- En producción, **HTTPS** delante de BTCPay (Caddy, nginx, Traefik) y restringe **9740** si no la necesitas expuesta.
- Antes de actualizar versiones, lee notas de migración de BTCPay Server y NBXplorer.


## Lecciones Aprendidas

### RPC-over-Onion

La conectividad RPC vía Tor es muy inestable.
- Tailscale para RPC/ZMQ resultó mucho más estable

Decisión final:
- Tor solo para P2P Bitcoin
- Tailscale para RPC/ZMQ

Es la arquitectura recomendada para producción.

---

## Estado Final

Todos los componentes funcionando:

- BTCPay Server
- NBXplorer
- Bitcoin Core remoto
- Tor P2P
- Tailscale RPC/ZMQ
- phoenixd Lightning
- Facturación Lightning
- Pagos Lightning testeados correctamente

