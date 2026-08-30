# Renovación del certificado cliente de `talosconfig`

## Qué pasó (2026-08-29)

El certificado cliente `os:admin` embebido en el Secret `talosconfig`
(namespace `backup-system`, usado por el CronJob `etcd-backup` para tomar
snapshots de etcd vía `talosctl`) venció el **2026-08-28 23:48 GMT**. Los
certificados que genera `talosctl gen config` / `talosctl config new` tienen
por defecto **1 año de vigencia** (`--crt-ttl`, default `8760h`) y Talos no
los rota solo — es una credencial estática que hay que renovar a mano.

Síntoma: el CronJob `etcd-backup` empezó a fallar todas las noches con:

```
error reading file: rpc error: code = Unavailable desc = connection error:
desc = "error reading server preface: remote error: tls: expired certificate"
```

Esto también bloquea cualquier otro uso de `talosctl` contra el cluster con
esa misma identidad (ej. una recuperación de emergencia).

La CA del cluster (`cluster.ca`) sigue siendo válida hasta 2035 — solo se
vence el certificado *cliente*, no la CA.

## Cómo renovarlo

Necesitás `talosctl` instalado localmente y alguno de los `controlplane-N.yaml`
del repo `talos-config` (tiene embebida la clave privada de la CA del
cluster, por eso alcanza con eso, sin necesitar un `talosconfig` ya válido).

### 1. Extraer un secrets bundle desde un controlplane.yaml existente

```bash
talosctl gen secrets \
  --from-controlplane-config /ruta/a/talos-config/controlplane-1.yaml \
  -o /tmp/secrets.yaml --force
```

### 2. Generar un talosconfig nuevo a partir de ese bundle

```bash
talosctl gen config synergia-k8s https://10.42.20.4:6443 \
  --with-secrets /tmp/secrets.yaml \
  -t talosconfig \
  -o /tmp/talosconfig-new --force
```

Confirmá que la fecha de expiración quedó en el futuro:

```bash
TALOSCONFIG=/tmp/talosconfig-new talosctl config info
```

### 3. (Opcional) Pedir un cert de mayor vigencia usando el nuevo talosconfig

Si en vez de 1 año querés más tiempo, usá el talosconfig recién generado
(ya válido) para pedirle al cluster uno nuevo con otra vigencia — no hace
falta repetir los pasos 1-2 para esto:

```bash
TALOSCONFIG=/tmp/talosconfig-new talosctl config new /tmp/talosconfig-Nanios \
  -e 10.42.20.4 -n 10.42.20.5 \
  --crt-ttl 87600h \
  --roles os:admin
# 87600h = 10 años. Ajustar según lo que se quiera.
```

### 4. Actualizar el Secret cifrado con SOPS en el repo

El Secret `secret-talosconfig.yaml` (`bootstrap/kubernetes/apps/secrets/`)
tiene una clave `stringData.config` con el contenido completo del
talosconfig como bloque YAML. Hay que reemplazar ese bloque por el
contenido de `/tmp/talosconfig-new` (o el que se haya generado en el paso
3) y volver a cifrar:

```bash
cd synergia-k8s

python3 <<'PYEOF'
import subprocess

decrypted = subprocess.run(
    ["sops", "-d", "bootstrap/kubernetes/apps/secrets/secret-talosconfig.yaml"],
    capture_output=True, text=True, check=True
).stdout

with open("/tmp/talosconfig-new") as f:
    new_config = f.read()

lines = decrypted.splitlines()
out = []
i = 0
replaced = False
while i < len(lines):
    line = lines[i]
    out.append(line)
    if line.strip() == "config: |":
        replaced = True
        indent = " " * (len(line) - len(line.lstrip()) + 4)
        for cfg_line in new_config.splitlines():
            out.append(indent + cfg_line if cfg_line else "")
        i += 1
        base_indent = len(line) - len(line.lstrip())
        while i < len(lines) and (lines[i].strip() == "" or (len(lines[i]) - len(lines[i].lstrip())) > base_indent):
            i += 1
        continue
    i += 1

assert replaced, "config block not found!"
with open("bootstrap/kubernetes/apps/secrets/secret-talosconfig.yaml", "w") as f:
    f.write("\n".join(out) + "\n")
PYEOF

sops -e -i bootstrap/kubernetes/apps/secrets/secret-talosconfig.yaml

# Verificar que quedó bien, sin imprimir la clave privada:
sops -d bootstrap/kubernetes/apps/secrets/secret-talosconfig.yaml \
  | grep "crt:" | awk '{print $2}' | base64 -d | openssl x509 -noout -dates
```

### 5. Commit, push, y forzar reconciliación de Flux

```bash
git add bootstrap/kubernetes/apps/secrets/secret-talosconfig.yaml
git commit -m "[backup] renew talosconfig client cert"
git push origin main
```

Si Flux tarda en levantarlo (puede haber que forzar el `GitRepository` y el
`Kustomization` `apps-secrets-config` a mano vía la anotación
`reconcile.fluxcd.io/requestedAt`), o simplemente esperar el intervalo de
5 minutos.

### 6. Verificar que el backup real funciona

Corré un Job manual (no hace falta esperar al próximo 3am):

```bash
kubectl create job etcd-backup-verify -n backup-system --from=cronjob/etcd-backup
kubectl logs -f -n backup-system job/etcd-backup-verify
kubectl delete job etcd-backup-verify -n backup-system
```

### 7. Limpieza

Borrar los archivos temporales locales — en particular `/tmp/secrets.yaml`
tiene la clave privada completa de la CA del cluster:

```bash
rm -f /tmp/secrets.yaml /tmp/talosconfig-new /tmp/talosconfig-Nanios
```

## Recordatorio automático

Para no repetir este incidente, hay un CronJob (`talosconfig-cert-check`,
namespace `backup-system`, corre a diario) que calcula los días restantes
del certificado y publica la métrica `talos_client_cert_expiry_days` a un
Prometheus Pushgateway. Una alerta (`TalosConfigCertExpiringSoon`, definida
en `bootstrap/kubernetes/apps/monitoring/prometheus/app/configmap-prometheus.yaml`)
dispara por Telegram (mismo receiver que ya usan el resto de las alertas de
este cluster) cuando falten menos de 7 días.

Ver `bootstrap/kubernetes/apps/tools/backup/app/talosconfig-cert-check.yaml`.
