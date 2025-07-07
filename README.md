# OpenBao Docker Compose Setup

It's designed to be fairly production-ready while still being simple to deploy.

## Notes

**OpenBao 2.1.0+ doesn't have a web UI anymore.** You'll need to use the CLI or API for everything. If you really need a web interface, you might want to stick with an older version or look into other third-party tools.

**You'll need to manually unseal OpenBao every time it restarts.** This is by design for security, but you can  use Auto Unseal with cloud KMS (like AWS KMS, read below)

**TLS is disabled by default.** If you're running this behind nginx on the same server, it's fine. If you're exposing this directly or running distributed, you'll want to enable TLS.

**Client libraries**: OpenBao intends to remain API compatible with HashiCorp Vault. This means that most of the existing libraries for Vault should also work with OpenBao. But at the time of writing, openbao does support atleast one official client library for golang. [[Source]](https://openbao.org/api-docs/libraries/)

## Getting started

1. clone this repo and copy sample.env to .env [ if you want to use postgres ]

2. edit the `.env` file and change the PostgreSQL password to something secure  [ if you want to use postgres ]
 
3. Start everything:
   ```bash
   docker compose up -d
   ```

4. Initialize OpenBao (only do this once):
   ```bash
   docker compose exec openbao bao operator init
   ```

   Here's what this will give you:
   ```bash
    Unseal Key 1: [REDACTED]
    Unseal Key 2: [REDACTED]
    Unseal Key 3: [REDACTED]
    Unseal Key 4: [REDACTED]
    Unseal Key 5: [REDACTED]

    Initial Root Token: [REDACTED]

    Vault initialized with 5 key shares and a key threshold of 3.

    Please securely distribute the key shares printed above. When the Vault is re-sealed,
    restarted, or stopped, you must supply at least 3 of these keys to unseal it
    before it can start servicing requests.

    Vault does not store the generated root key. Without at least 3 keys to
    reconstruct the root key, Vault will remain permanently sealed!

    It is possible to generate new unseal keys, provided you have a quorum of
    existing unseal keys shares. See "bao operator rekey" for more information.
    ```


5. Unseal OpenBao (you'll need to do this every time server restarts, or use AWS KMS for auto unseal):
   ```bash
   docker compose exec openbao bao operator unseal
   ```
   Enter one of the unseal keys when prompted. You'll need to run this command 3 times with different keys (assuming you used the default threshold).
   
   Every time you run this command, you will see the unseal progress `(Unseal Progress 1/3)`

6. OpenBao should now be running at `http://localhost:8200`

## Day-to-day operations

Check if OpenBao is running and unsealed:
```bash
docker compose exec openbao bao status
```

Seal OpenBao (locks it until you unseal again):
```bash
docker compose exec openbao bao operator seal
```

Back up your database:
```bash
docker compose exec db pg_dump -U openbao openbao > backup.sql
```

## Making it more production-ready

### Auto Unseal
If you don't want to manually unseal every time, you can configure Auto Unseal with AWS KMS, Azure Key Vault, etc. Just uncomment and configure the seal block in `bao.conf`.

### TLS
If you're not using a reverse proxy, enable TLS by setting `tls_disable = false` and providing certificate paths in the listener block.

### High Availability
[Read openbao docs](https://openbao.org/docs/internals/high-availability/)

## For development only

If you're just testing and don't want to deal with unsealing, you can use dev mode instead:

```yaml
openbao:
  image: openbao/openbao:2.1.0
  command: ['server', '-dev', '-dev-listen-address=0.0.0.0:8200']
  ports:
    - 8200:8200
```

This runs in memory and starts unsealed, but you'll lose all data when the container stops.

## Security notes

- Store your unseal keys securely and split them among trusted people [ not needed if you use cloud KMS for auto unseal ]
- Use TLS if you're not behind a reverse proxy

## Why OpenBao instead of HashiCorp Vault?

OpenBao is a community fork of Vault that maintains the MPL license after HashiCorp changed their licensing. If you were already using Vault and want to avoid the licensing changes, or if you prefer community-driven development, OpenBao is a solid choice.

## Official Openbao docs:

- [OpenBao Documentation](https://openbao.org/docs/)