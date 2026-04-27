# Authelia in this cluster

This stack uses the **file** authentication backend. Users live in the Kubernetes ConfigMap `authelia-users` (`app/configmap-authelia-users.yaml`), mounted into the Authelia pod as `/config/users_database.yml` (see `configmap-authelia-values.yaml`).

Official reference: [File-based authentication](https://www.authelia.com/configuration/first-factor/file/).

---

## Adding a user

### 1. Choose a username and password

- **Username** is the YAML key under `users:` (e.g. `alice`). That value is what people type on the Authelia sign-in page (unless you configure otherwise).
- **Never** commit a plaintext password. Store only an **Argon2id** digest in Git.

### 2. Generate a password digest

Use the same Authelia major/minor version as your running image (check the Authelia pod image tag if unsure).

```bash
# Replace 'YourSecurePasswordHere' with the password you want.
docker run --rm ghcr.io/authelia/authelia:4.39.13 \
  authelia crypto hash generate argon2 --password 'YourSecurePasswordHere'
```

The command prints a line starting with `Digest: ` followed by a string beginning with `$argon2id$...`. **Copy only the digest** (the part starting with `$argon2id$`), not the `Digest: ` prefix.

### 3. Edit the users ConfigMap

Edit `app/configmap-authelia-users.yaml`. Under `data.users_database.yml: |`, add a block under `users:`:

```yaml
    users:
      admin:
        disabled: false
        displayname: Administrator
        password: "$argon2id$v=19$m=..."
        email: admin@synergia.net.ar
        groups:
          - admins
      alice:
        disabled: false
        displayname: Alice Example
        password: "$argon2id$v=19$m=...paste_digest_here..."
        email: alice@synergia.net.ar
        groups:
          - admins
```

Rules:

- **Indentation** under `users_database.yml: |` must stay consistent (this file uses four spaces before each key under `users:`).
- Wrap the digest in **double quotes** so YAML does not treat `$` oddly.
- **email** must be unique per user.
- **groups** are strings; use groups that match your **access control** and **OIDC client** policies in `configmap-authelia-values.yaml` (e.g. `admins`).

### 4. Roll out the change

After you **commit and push**, let Flux reconcile, or apply manually:

```bash
kubectl apply -f bootstrap/kubernetes/apps/security/authelia/app/configmap-authelia-users.yaml
kubectl rollout restart deployment/authelia -n security
```

Until the pod restarts (or the ConfigMap is reloaded), old user data may still be in memory.

---

## Disabling or removing a user

- **Disable** (keeps the entry, blocks login): set `disabled: true` for that user.
- **Remove**: delete the user’s YAML block under `users:`.

Then apply / commit the same way as above.

---

## Password rotation

Repeat **Generate a password digest** with a new password, replace the `password:` field for that user, commit/push, and restart the Authelia deployment (or wait for GitOps).

---

## Where this ties to authorization

- **Portal / Forward Auth** (e.g. Traefik dashboard): controlled by `access_control` in `app/configmap-authelia-values.yaml` (domains and `policy`).
- **OIDC clients** (e.g. Homebox): client settings and `authorization_policy` are in the same values ConfigMap.

If a user can sign in but cannot use an app, check **groups** and those policies, not only the user file.

---

## Troubleshooting

| Symptom | Check |
|--------|--------|
| Invalid credentials after edit | Digest copied fully; quotes in YAML; pod restarted. |
| User exists but no access to a host | `access_control.rules` for that `domain` in `configmap-authelia-values.yaml`. |
| YAML errors on apply | Indentation under `users_database.yml: \|`; validate with `kubectl apply --dry-run=client -f ...`. |

---

## Further reading

- [User database YAML format](https://www.authelia.com/reference/guides/passwords/)
- [Authelia configuration](https://www.authelia.com/configuration/prologue/introduction/)
