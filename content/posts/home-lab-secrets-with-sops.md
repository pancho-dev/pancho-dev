---
title: "Home Lab Secrets with SOPS"
date: 2026-09-16T11:00:00-03:00
draft: false
tags: ["security", "automation", "secrets", "sops"]
---

In a past blog post I covered using Bitwarden for home lab automation secrets ([Secrets Manager for Home Lab Automation]({{< relref "home-lab-automation-secrets.md" >}})). That approach works well if you already use a password manager and want a CLI to pull credentials into environment variables. This follow-up covers [SOPS](https://github.com/getsops/sops) as an alternative when you want encrypted secrets in the repo without relying on a password manager or paying for a subscription.

The goal is the same: get secrets into environment variables for Terraform, Ansible, and cloud CLIs at runtime. The difference is where those secrets live. With SOPS you keep them versioned next to your infrastructure code as ciphertext. That means you are putting secrets in git — encrypted, but still in the repo — which is a tradeoff worth accepting for many solo home lab setups because it removes login/session/sync choreography and avoids another hosted service.

This post is opinionated toward SOPS with [age](https://github.com/FiloSottile/age) keys. Age is simpler than GPG for a personal lab: generate a keypair, keep the private key off the repo, and commit a small config that points at your public key.


# Why SOPS for a home lab

Hashicorp Vault and cloud secret managers are great when you need fine-grained access control, audit logs, and team workflows. For a personal home lab they often feel like operational burden. A password manager CLI (like Bitwarden) is a good middle ground if you already pay for it or use the free tier and sync across phone and laptop.

SOPS sits in a different spot:

- No subscription and no vault service to keep running
- Secrets travel with the repo, so cloning the project on a new machine is enough once your age private key is in place
- Encryption happens locally; only ciphertext is committed
- Editing secrets is a normal file workflow (`sops secrets.yaml`) instead of app UI + sync

The cost of that simplicity is the threat model. Encrypted secrets live forever in git history. If someone steals your age private key, they can decrypt every revision that was encrypted to that key. Treat the private key like a master password: back it up carefully, never commit it, and do not leave it on shared or untrusted machines.


# Setup

You need two tools: `sops` and `age`. Install them with your package manager of choice (examples below; versions change, so check current docs if needed).

```bash
# macOS with Homebrew
$ brew install sops age

# Or on a Debian/Ubuntu-like system (package names may vary by distro)
$ sudo apt install age
# Install sops from the GitHub release for your arch if the distro package is old
```

### Generate an age keypair

The private key decrypts secrets. The public key encrypts them. Only the public key belongs in the repo.

```bash
# Create the default directory SOPS looks for age keys
$ mkdir -p ~/.config/sops/age

# Generate a keypair and write the private key file (do NOT commit this file)
$ age-keygen -o ~/.config/sops/age/keys.txt
Public key: age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

SOPS will find the private key automatically when that path exists. You can also set `SOPS_AGE_KEY_FILE` if you store the key somewhere else.

Keep a backup of `keys.txt` somewhere safe (encrypted USB, password manager note for the key file itself, etc.). Losing the private key means losing the ability to decrypt your secrets file.

### Configure SOPS for the repo

In the root of your automation repo, create a `.sops.yaml` that tells SOPS which age public key to use and which files to encrypt. Only the public key goes here.

```yaml
# .sops.yaml — safe to commit
creation_rules:
  - path_regex: secrets\.yaml$
    age: age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

You can widen the regex later (`.*secrets\.ya?ml$`, `secrets/.*`, etc.). For this post we keep a single `secrets.yaml`.


# Encrypt and decrypt workflow

Create a plain YAML file with the same kind of credentials used in the Bitwarden post: cloud provider API keys and S3-compatible backend credentials for Terraform state.

```bash
$ cat > secrets.yaml <<'EOF'
vultr:
  api_key: "vultr-api-key-example-1234567890"
cloudflare:
  api_token: "cloudflare-api-token-example-abcdef"
aws:
  access_key_id: "AKIAEXAMPLEKEYID1234"
  secret_access_key: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
EOF
```

Encrypt the file in place. After this, `secrets.yaml` contains ciphertext and is safe to commit.

```bash
$ sops -e -i secrets.yaml
```

If you open the file you will see encrypted values and a `sops` metadata block. Keys stay readable; values are encrypted. That is intentional so diffs stay somewhat reviewable.

To edit secrets later, open them through SOPS so they decrypt into your editor and re-encrypt on save:

```bash
$ sops secrets.yaml
```

To decrypt to stdout (useful for scripting):

```bash
$ sops -d secrets.yaml
vultr:
  api_key: vultr-api-key-example-1234567890
cloudflare:
  api_token: cloudflare-api-token-example-abcdef
aws:
  access_key_id: AKIAEXAMPLEKEYID1234
  secret_access_key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

Commit `.sops.yaml` and the encrypted `secrets.yaml`. Do not commit plaintext copies, backup `*.dec.yaml` files, or your age private key.


# Fetching secrets into environment variables

Same pattern as the Bitwarden post: load credentials into environment variables so Terraform providers, cloud CLIs, and Ansible lookups pick them up from the process environment. Variables live for the shell session and disappear when the shell exits.

You can parse decrypted YAML with `yq`, or use SOPS helpers that avoid dumping secrets to intermediate files.

### Using yq

```bash
# Decrypt and assign fields to the env vars providers expect
$ export VULTR_API_KEY=$(sops -d secrets.yaml | yq -r '.vultr.api_key')
$ export CLOUDFLARE_API_TOKEN=$(sops -d secrets.yaml | yq -r '.cloudflare.api_token')
$ export AWS_ACCESS_KEY_ID=$(sops -d secrets.yaml | yq -r '.aws.access_key_id')
$ export AWS_SECRET_ACCESS_KEY=$(sops -d secrets.yaml | yq -r '.aws.secret_access_key')
```

### Using sops exec-env

If you shape the file as a flat map of environment variable names to values, `sops exec-env` can inject them for a single command without exporting them in your interactive shell history the same way.

```yaml
# Alternative shape for exec-env (still encrypted with SOPS)
VULTR_API_KEY: "vultr-api-key-example-1234567890"
CLOUDFLARE_API_TOKEN: "cloudflare-api-token-example-abcdef"
AWS_ACCESS_KEY_ID: "AKIAEXAMPLEKEYID1234"
AWS_SECRET_ACCESS_KEY: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
```

```bash
# Run terraform with secrets injected only for that process tree
$ sops exec-env secrets.yaml 'terraform apply'
```

I prefer nested YAML for readability when many services share one file, and `yq` exports when I need the same env vars for several tools in one shell. Use `exec-env` when you want a tighter scope around a single command.

### Ansible

Export first, then look up from the environment:

```bash
$ export SSH_PUB_KEY=$(sops -d secrets.yaml | yq -r '.ssh.public_key')
```

```yaml
- name: Print SSH Pub Key
  ansible.builtin.debug:
    msg: "'{{ lookup('ansible.builtin.env', 'SSH_PUB_KEY') }}'"
```

### Terraform providers

Same credential chain idea as in the Bitwarden post. After exporting, the AWS, Cloudflare, Vultr, and similar providers read from the usual environment variables. No secrets need to live in `.tf` files.


# Example

Context: Terraform creates a VM on Vultr, DNS records on Cloudflare, and uses an S3-compatible backend for state. Terraform code itself is out of scope; the point is loading credentials the same way as the Bitwarden example, but from SOPS.

Because those secrets live encrypted in git, the ideal scenario is to use credentials that are purpose-built for this automation and narrowly scoped to the function they serve. That limits blast radius if a key leaks and makes revocation easy later — you rotate or delete one automation-specific credential instead of a full personal or admin key.

In this example that would mean:

- A Cloudflare API token scoped to the DNS zone you manage, with only the permissions needed for that purpose (for example DNS edit on that domain), and used only by this automation
- An S3-compatible access key tied to that state bucket, with constrained permissions (read/write on that bucket only, not account-wide)
- A Vultr API key limited to what the stack needs — for example create VMs or manage VIPs — rather than a full-account key

How far you can go depends on each platform's IAM or token model. Not every provider exposes the same granularity, but aiming for least privilege per automation is still the right default when secrets are stored in the repo.

```bash
# Private age key must already be available, e.g. ~/.config/sops/age/keys.txt

# Get the cloud provider api key
$ export VULTR_API_KEY=$(sops -d secrets.yaml | yq -r '.vultr.api_key')

# Get S3 storage backend keys
$ export AWS_ACCESS_KEY_ID=$(sops -d secrets.yaml | yq -r '.aws.access_key_id')
$ export AWS_SECRET_ACCESS_KEY=$(sops -d secrets.yaml | yq -r '.aws.secret_access_key')

# Get cloudflare api token
$ export CLOUDFLARE_API_TOKEN=$(sops -d secrets.yaml | yq -r '.cloudflare.api_token')
```

After those exports you can run the Terraform stack. No `bw login`, no session token, no vault sync — just decrypt with the local age key.


# Bitwarden vs SOPS

Both solve “do not hardcode secrets in plaintext next to the code.” They differ in where secrets live and how much day-to-day friction you accept.

**Where secrets live.** Bitwarden keeps them in a vault (cloud or self-hosted). SOPS keeps encrypted files in git. With SOPS you are deliberately putting secrets in the repository; with Bitwarden the repo stays free of secret material.

**Cost and operations.** Bitwarden has a solid free tier, paid plans for more features, and optional self-hosting. SOPS is free and local: age keys on your machine, no subscription. For a home lab that wants to avoid another bill or another service to patch, that matters.

**Multi-device UX.** Bitwarden wins if you need phone apps, browser extensions, and easy sync across laptops. SOPS is CLI-first. Sharing an age private key across machines is manual and easy to do wrong; it is fine for one or two trusted lab hosts, awkward as a general password manager replacement.

**Automation friction.** Bitwarden needs login, unlock, `BW_SESSION`, and often `bw sync` before reads. SOPS needs the age private key present and a decrypt command. For repo-bound automation, SOPS is usually less ceremony once the key is installed.

**History and audit.** Bitwarden has vault history and a product UX around items. SOPS leans on git: every encrypted change is in history. That is useful for IaC review and painful if you ever need to rotate after a private-key leak, because old ciphertext may still decrypt.

**Risk model.** Bitwarden risks are vault account access, session material left on disk, and sync/trust of the service. SOPS risks are private-key compromise and the long life of secrets-in-git. Encrypted-in-git is not the same as “not in git.”

**Best fit.**

- Choose Bitwarden (or another password manager CLI) when you already use it for personal secrets, need multi-device access, or want secrets out of the repository entirely.
- Choose SOPS when the secrets belong to an IaC/home-lab repo, you want zero subscription and minimal moving parts, and you accept encrypted secrets living in git next to the code.


# Conclusion

Using SOPS with age is a practical way to handle home lab automation secrets without a password manager or a self-hosted Vault. You encrypt a YAML file, commit the ciphertext, keep the age private key off the repo, and decrypt into environment variables when you run Terraform, Ansible, or cloud CLIs.

Compared to the Bitwarden approach in my earlier post, SOPS is simpler for solo, repo-centric labs and costs nothing beyond caring for a private key. The tradeoff is real: secrets live in git forever as ciphertext, and key hygiene becomes your whole security story. For enterprise or team environments with shared access needs, purpose-built secret managers are still a better fit.

For my own home lab automations that live in git, SOPS is often the path of least resistance. When I already need Bitwarden on every device for personal passwords, reusing it for automation remains a good option. Pick the model that matches where you want the secrets to live and how much operational surface you want to maintain.
