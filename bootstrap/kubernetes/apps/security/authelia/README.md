# Authelia in this cluster

This stack uses the **file** authentication backend.
- **Where the users live:** in the Secret `security/authelia-users`, kept in
  Git as `apps/secrets/secret-authelia-users.yaml` and encrypted with SOPS
  (age). Flux decrypts it in the `apps-secrets-config` Kustomization.
- **How Authelia reads it:** the pod mounts it as `/config/users_database.yml`
  with `subPath` (see `app/configmap-authelia-values.yaml`).

Official reference: [File-based authentication](https://www.authelia.com/configuration/first-factor/file/).

---

## Adding a user or changing a password

### 1. Generate a password digest

Use the same Authelia version as the running image. The simplest way is the
running pod itself. Leave out `--password`: the command then asks for the
password on the terminal, so it never ends up in the shell history.

```bash
kubectl -n security exec -it deploy/authelia -- authelia crypto hash generate argon2
```

The command prints a line starting with `Digest: `, followed by a string that
begins with `$argon2id$`. Copy only the digest, from `$argon2id$` on.

### 2. Edit the encrypted Secret

```bash
sops bootstrap/kubernetes/apps/secrets/secret-authelia-users.yaml
```

`sops` decrypts the file into your editor and encrypts it again when you save.
You need the age key in `~/Library/Application Support/sops/age/keys.txt`.
Under `stringData.users_database.yml`:

```yaml
users:
  alice:
    disabled: false
    displayname: Alice Example
    password: "$argon2id$v=19$m=...paste_digest_here..."
    email: alice@example.com
    groups:
      - admins
```

Rules:

- **Never** commit a plaintext password, or the decrypted file.
- Wrap the digest in **double quotes**, so YAML does not mangle the `$`.
- **email** must be unique per user.
- **groups** must match the `subject` of the access control rules (see below).
  Today the groups are `admins` and `finance`.

Before committing, check that the file is still fully encrypted:
`grep -c argon2` must return `0`.

### 3. Roll it out

Commit, push, and let Flux apply the Secret:
`flux reconcile kustomization apps-secrets-config --with-source`.

**Then restart Authelia.** The file is mounted with `subPath`, so the running
pod never sees the change by itself:

```bash
kubectl -n security rollout restart deployment/authelia
```

---

## Disabling or removing a user

- **Disable** (keeps the entry, blocks login): set `disabled: true`.
- **Remove**: delete the user's block under `users:`.

Then commit and restart as above.

---

## Where this ties to authorization

- **Forward auth** (every admin UI behind Traefik): `access_control` in
  `app/configmap-authelia-values.yaml`.
  - **A rule without a `subject` matches every signed-in user.**
    `default_policy: deny` only applies when no rule matches the domain. Every
    admin UI's rule therefore names `group:admins`; give a new rule a subject
    too.
  - After changing these values, run
    `flux reconcile helmrelease authelia -n security`. The HelmRelease reads
    the ConfigMap through `valuesFrom` and does not notice changes by itself.
- **OIDC clients** (e.g. Homebox): client settings and `authorization_policy`
  are in the same values ConfigMap.

If a user can sign in but cannot use an app, check the user's groups and the
rule's subject, not only the user file.

---

## Troubleshooting

| Symptom | Check |
|--------|--------|
| Invalid credentials after an edit | The digest was copied in full, it is quoted, and the pod was restarted. |
| A user can sign in but gets 403 on a host | That domain's `access_control` rule and its `subject` groups. |
| The Authelia pod is stuck in ContainerCreating | The `authelia-users` Secret is missing: check the `apps-secrets-config` Kustomization and the SOPS decryption. |

---

## Further reading

- [User database YAML format](https://www.authelia.com/reference/guides/passwords/)
- [Access control rules and subjects](https://www.authelia.com/configuration/security/access-control/)
