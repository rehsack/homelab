# Secrets Management

Stand: 2026-05-15

## Prinzip

Secrets liegen **verschlüsselt direkt in den YAML-Dateien** im Git.
Verschlüsselung erfolgt feldweise via [SOPS](https://github.com/getsops/sops)
mit [age](https://github.com/FiloSottile/age) als Backend.
Nicht-sensitive Felder bleiben im Klartext — Diffs und Reviews funktionieren normal.

## Voraussetzungen

```bash
brew install sops age
```

Der age-Privatschlüssel wird **pro Projekt** konfiguriert, nicht global.
Public Key: siehe `.sops.yaml` im Repo-Root.

```bash
# Variante A: per Aufruf
SOPS_AGE_KEY_FILE=~/.config/sops/age/homelab.txt sops -d talos/secrets.yaml

# Variante B: direnv (.envrc im Repo-Root, in .gitignore)
export SOPS_AGE_KEY_FILE=~/.config/sops/age/homelab.txt
```

## Verschlüsselte Dateien

| Datei | Verschlüsselte Felder |
|---|---|
| `talos/secrets.yaml` | `id`, `secret`, `bootstraptoken`, `secretboxencryptionsecret`, `token`, `crt`, `key` |
| `talos/controlplane.yaml` | `token`, `secret`, `id`, `secretboxEncryptionSecret`, `crt`, `key` |
| `talos/worker.yaml` | `token`, `secret`, `id`, `crt`, `key` |
| `talos/talosconfig.yaml` | `ca`, `crt`, `key` |
| `applications/forgejo/forgejo-secret.yaml` | alle Werte unter `stringData` |

## Täglicher Umgang

```bash
# Secret-Datei im Editor bearbeiten (entschlüsselt, verschlüsselt beim Speichern)
sops talos/secrets.yaml

# Klartext auf stdout (für Scripting, z.B. talosctl apply-config)
sops -d talos/controlplane.yaml

# talosconfig entschlüsseln für talosctl
sops -d talos/talosconfig.yaml > ~/.talos/config

# Datei nach Änderung verschlüsseln (wenn manuell als Klartext bearbeitet)
sops -e -i talos/secrets.yaml
```

## Nach Fresh Clone

1. age-Privatschlüssel out-of-band erhalten, z.B. nach `~/.config/sops/age/homelab.txt`
2. `SOPS_AGE_KEY_FILE` setzen (per direnv oder Shell-Alias)
3. Fertig — alle Dateien sind im Repo, `sops -d` entschlüsselt on-the-fly

## Nach Secret-Rotation

1. Neue Secrets generieren (`talosctl gen secrets` o.ä.)
2. `sops talos/secrets.yaml` — Werte ersetzen, speichern
3. Auf den Cluster applyen

## Neuen Empfänger hinzufügen

1. Neuen age-Public-Key in `.sops.yaml` ergänzen (kommasepariert oder als Liste)
2. Alle verschlüsselten Dateien mit `sops updatekeys <datei>` re-encrypten

## Reboot-Prozedur (Single-Node mit CephFS)

Bevor ein Node rebootet wird:

```bash
# 1. CephFS-Workloads vom Node drainen
kubectl drain f5a-01 --ignore-daemonsets --delete-emptydir-data

# 2. Warten bis CephFS-Mounts weg sind
talosctl read /proc/mounts | grep ceph
# (sollte leer sein)

# 3. Erst dann reboot
talosctl reboot
```

Falls `talosctl reboot` trotzdem hängt: ACPI Power-Off (kurzer Druck auf Powerknopf).
