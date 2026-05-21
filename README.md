# runway-secrets

Encrypted per-user secrets (wandb API keys, HF tokens, etc.) for the
Runway agents. **Private repo.** Inside the repo, every file under
`secrets/` is encrypted with [age](https://age-encryption.org).

```
recipients/                  age public keys, one per person/host
  <github-login>.age.pub     a contributor (intern, admin)
  <agent-host>.age.pub       an agent host (4vita, gcp, ...)

secrets/
  <github-login>.env.enc     env-file format, age-encrypted to:
                             { owner, all agent hosts, admin }
```

## Trust model

- Each user holds their **own age private key** locally
  (`~/.config/age/runway.key`, mode 600).
- The private key **never** enters this repo.
- A secret is encrypted with multiple recipient public keys: the owner,
  every agent host, and the admin (for recovery). Anyone holding any of
  those private keys can decrypt that file.
- A user cannot decrypt another user's secret (their private key is not
  a recipient).

## Onboarding a new user (manual today; CLI later)

1. User generates an age keypair locally:
   ```bash
   mkdir -p ~/.config/age && chmod 700 ~/.config/age
   age-keygen -o ~/.config/age/runway.key
   ```
2. User PRs the **public key** (only) to `recipients/<github-login>.age.pub`.
3. Admin reviews + merges the PR.
4. User can now encrypt their secrets against all current recipient keys:
   ```bash
   for f in recipients/*.age.pub; do args+=( -R "$f" ); done
   age "${args[@]}" -o secrets/<login>.env.enc <<EOF
   WANDB_API_KEY=...
   WANDB_ENTITY=...
   HF_TOKEN=...
   EOF
   ```
5. PR the encrypted file.

## How the agent uses this

On boot, the agent clones this repo to `~/.rwy/secrets-repo/`. Before
each poll iteration it runs `git pull`. When a spec is claimed, the
agent reads `secrets/<spec.metadata.owner>.env.enc`, decrypts with its
own age private key, and injects the resulting `KEY=value` pairs into
the subprocess environment.

A missing or undecryptable secret file is **non-fatal**: the run still
executes, just without wandb / HF credentials. The training script can
detect the missing env var and skip those integrations.
