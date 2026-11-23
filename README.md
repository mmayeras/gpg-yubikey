# Signed Git Commits with GPG, a YubiKey and GitLab

A step-by-step guide, from an empty keyring to a "Verified" commit on GitLab, with the signing key stored on a YubiKey.

---

## Overview

What you will end up with:

```
Primary key (ed25519)  [SC]  → certifies subkeys, kept OFFLINE in a backup
 ├─ Subkey (cv25519)   [E]   → encryption, on the YubiKey
 └─ Subkey (ed25519)   [S]   → signs Git commits, on the YubiKey
```

| Letter | Meaning |
|---|---|
| `S` | Sign |
| `C` | Certify (create or modify subkeys, sign other keys) |
| `E` | Encrypt |

> **Signing is not encryption.** A signed commit proves *who* wrote it and that it was not altered. The code itself stays readable. The encryption subkey is there for tools like `git-crypt`, `sops` or `pass`.

**Requirements**

- GnuPG 2.2+ (2.4 recommended)
- A YubiKey 5 with firmware **5.2.3 or newer** (needed for Curve25519 keys)
- `ykman` (YubiKey Manager CLI)
- A GitLab account with a **verified email** that you will put in the key

```bash
# Fedora / RHEL
sudo dnf install gnupg2 yubikey-manager pcsc-lite
sudo systemctl enable --now pcscd

# Debian / Ubuntu
sudo apt install gnupg scdaemon yubikey-manager pcscd

# macOS
brew install gnupg pinentry-mac ykman
```

---

## 1. Check the YubiKey

```bash
ykman info
```
Prints the model and firmware version. Check that firmware is **≥ 5.2.3**. If it is older, use RSA 4096 instead of ECC throughout this guide.

```bash
ykman openpgp info
```
Shows the state of the OpenPGP applet: PIN retry counters, which key slots are filled, and touch policies. Run this whenever you want to know what is on the card.

```bash
gpg --card-status
```
Same idea, seen from GnuPG's side. If this works, GPG can talk to the card.

---

## 2. Generate the primary key

```bash
gpg --expert --full-generate-key
```
Starts the interactive key generator. `--expert` unlocks all algorithm choices.

Answer the prompts:

| Prompt | Answer | Why |
|---|---|---|
| Kind of key | `(9) ECC (sign and encrypt)` | Creates an Ed25519 primary key (sign + certify) and a Cv25519 encryption subkey in one go |
| Curve | `(1) Curve 25519` | Modern, fast, small, well supported by GitLab and the YubiKey |
| Expiry | `2y` (recommended) or `0` | An expiry date limits damage if you lose everything. You can extend it later with your backup |
| Real name | Your name | Shown on the key |
| Email | **Your verified GitLab email** | GitLab only marks commits "Verified" if this matches |
| Passphrase | A strong one | Protects the key on disk and in your backup |

List the result:

```bash
gpg --list-secret-keys --keyid-format=long
```
`--keyid-format=long` shows 16-character key IDs. Example output:

```
sec   ed25519/5BD30DE0A0033B85 2026-11-23 [SC]
      7D698E1D7347B1411221F7B95BD30DE0A0033B85
uid                 [ultimate] Jane Doe <jane@example.com>
ssb   cv25519/B5E42894FCB08C5E 2026-11-23 [E]
```

- `sec` = your secret primary key. `5BD30DE0A0033B85` is its **key ID**; the long line under it is the full **fingerprint**.
- `ssb` = secret subkey.

In the rest of this guide, `PRIMARY_ID` is the primary key ID (e.g. `5BD30DE0A0033B85`).

---

## 3. Add a dedicated signing subkey

Git should sign with a subkey, not the primary key. That way the primary key can live offline.

```bash
gpg --expert --edit-key PRIMARY_ID
```
Opens the interactive key editor on your key. You get a `gpg>` prompt.

```
gpg> addkey
```
Creates a new subkey. Choose:
- `(10) ECC (sign only)`
- `(1) Curve 25519`
- Same expiry as the primary key

```
gpg> save
```
Writes the change and exits. **Always use `save`.** `quit` throws away your changes.

Check again:

```bash
gpg --list-secret-keys --keyid-format=long
```
```
sec   ed25519/5BD30DE0A0033B85 [SC]
ssb   cv25519/B5E42894FCB08C5E [E]
ssb   ed25519/6E04B72300A7DFAB [S]    ← new signing subkey
```

In the rest of this guide, `SIGNING_ID` is the `[S]` subkey ID (e.g. `6E04B72300A7DFAB`).

---

## 4. Back up everything (do not skip)

Moving keys to the YubiKey **deletes them from your computer**. There is no way to extract a key from the card afterwards. Without a backup, a lost or broken YubiKey means the key is gone for good.

```bash
gpg --armor --export-secret-keys PRIMARY_ID > master-private.asc
```
Exports the primary key **and** all subkeys (private parts) as text. `--armor` makes it ASCII instead of binary. This is your master backup.

```bash
gpg --armor --export-secret-subkeys PRIMARY_ID > subkeys-private.asc
```
Exports only the subkeys, with a stub in place of the primary key. Handy if you later want a computer that can sign but not certify.

```bash
gpg --armor --export PRIMARY_ID > public.asc
```
Exports the **public** key. This is safe to share and is what you upload to GitLab.

```bash
gpg --output revoke.asc --gen-revoke PRIMARY_ID
```
Creates a revocation certificate. If the key is ever compromised, publishing this tells everyone to stop trusting it.

Copy all four files to **offline, encrypted** storage (an encrypted USB stick, kept in a safe place). Then delete them from your working folder.

---

## 5. Set the YubiKey PINs

The OpenPGP applet has its own PINs:

| PIN | Default | Used for | Lock behaviour |
|---|---|---|---|
| User PIN | `123456` | Every signature / decryption | Locks after 3 wrong tries. The admin PIN can unlock it |
| Admin PIN | `12345678` | Managing the card (moving keys, touch policy, unblocking) | **3 wrong tries wipes the OpenPGP applet** |

```bash
gpg --card-edit
```
Opens the interactive card editor (`gpg/card>` prompt).

```
gpg/card> admin
```
Enables admin commands.

```
gpg/card> passwd
```
Opens the PIN menu:
- `1` → change the user PIN (6+ digits)
- `3` → change the admin PIN (8+ digits)
- `2` → **unblock** the user PIN if you have locked it (asks for the admin PIN)
- `q` → back

```
gpg/card> quit
```
Leaves the card editor. Card changes are written immediately, so `quit` is fine here.

Check the counters:

```bash
ykman openpgp info
```
`PIN tries remaining` should be `3`. If it is lower, use `passwd` → `2` to reset it before going further.

---

## 6. Move the subkeys to the YubiKey

```bash
gpg --edit-key PRIMARY_ID
```

Subkeys are numbered in the order they are listed: **key 1 = encryption `[E]`**, **key 2 = signing `[S]`**. A `*` next to `ssb` means "selected".

**Encryption subkey:**

```
gpg> key 1
```
Selects subkey 1 (the `[E]` one). A `*` appears next to it.

```
gpg> keytocard
Your selection? 2
```
Moves the selected subkey into the card's **encryption** slot. GPG only offers the slots that match the key's usage, so for an `[E]` key you will only see `(2)`.

```
gpg> key 1
```
Deselects subkey 1 (the `*` disappears). Important: `keytocard` refuses to work with more than one key selected.

**Signing subkey:**

```
gpg> key 2
```
Selects subkey 2 (the `[S]` one).

```
gpg> keytocard
Your selection? 1
```
Moves it into the card's **signature** slot.

```
gpg> save
```
Commits the change: the private subkeys are removed from disk and replaced by stubs that point to the card.

Verify:

```bash
gpg --list-secret-keys --keyid-format=long
```
```
sec   ed25519/5BD30DE0A0033B85 [SC]
ssb>  cv25519/B5E42894FCB08C5E [E]
ssb>  ed25519/6E04B72300A7DFAB [S]
```
`ssb>` means "this subkey lives on a smartcard". Both subkeys should have the `>`.

> **Two YubiKeys?** Re-import the backup (`gpg --import master-private.asc`), plug in the second key, and repeat this step. Both cards will hold the same subkeys.

---

## 7. Require a physical touch to sign

Without a touch policy, any program running as you can sign silently while the YubiKey is plugged in. With it, every signature needs a tap on the key.

GPG's smartcard daemon can hold the card and block `ykman`, so release it first:

```bash
gpgconf --kill scdaemon
```
Stops `scdaemon`, the GnuPG process that talks to the card. GPG restarts it automatically the next time it needs the card.

```bash
 ykman openpgp keys set-touch --admin-pin 'YOUR_ADMIN_PIN' --force sig on
```
- `set-touch sig on` → require a touch for the **signature** slot
- `--admin-pin` → passes the PIN directly, so there is no hidden prompt that looks frozen
- `--force` → skips the "are you sure?" confirmation
- The **leading space** keeps the command (and your PIN) out of shell history. Bash needs `HISTCONTROL=ignorespace`; zsh needs `setopt HIST_IGNORE_SPACE`.

Optional, for the encryption slot too:

```bash
 ykman openpgp keys set-touch --admin-pin 'YOUR_ADMIN_PIN' --force enc on
```

Verify:

```bash
ykman openpgp info
```
The signature key should show `Touch policy: On`.

> **Policy options:** `on` (can be changed later), `fixed` (can only be removed by resetting the applet), `cached` (one touch is valid for 15 seconds, useful for rebases that sign many commits).

---

## 8. Configure Git

```bash
git config --global user.name "Jane Doe"
git config --global user.email "jane@example.com"
```
Sets the author identity. The email must match the key's email **and** a verified email on GitLab.

```bash
git config --global user.signingkey SIGNING_ID!
```
Tells Git which key to sign with. The trailing `!` forces GPG to use **exactly this subkey** instead of choosing one itself. Without it, GPG may try the primary key.

```bash
git config --global commit.gpgsign true
```
Signs every commit automatically, so you don't need `-S` each time.

```bash
git config --global tag.gpgSign true
```
Signs annotated tags too (optional).

If Git can't find GPG (common on macOS):

```bash
git config --global gpg.program "$(which gpg)"
```
Points Git at the GPG binary explicitly.

---

## 9. Make the PIN prompt work

GPG asks for your user PIN through `pinentry`. In a terminal, it needs to know which terminal it is in:

```bash
echo 'export GPG_TTY=$(tty)' >> ~/.bashrc   # or ~/.zshrc
source ~/.bashrc
```
`GPG_TTY` tells `gpg-agent` where to show the PIN prompt. Without it, signing fails with `gpg failed to sign the data`.

On macOS, use the graphical prompt instead:

```bash
echo "pinentry-program $(brew --prefix)/bin/pinentry-mac" >> ~/.gnupg/gpg-agent.conf
gpgconf --kill gpg-agent
```
Sets the PIN dialog to a native macOS window and restarts the agent to apply it.

---

## 10. Upload the public key to GitLab

Export the public key **now**, after all subkeys exist:

```bash
gpg --armor --export PRIMARY_ID
```
Prints the public key, which contains the primary key and every subkey. Moving keys to the card does not change the public key, but **adding a subkey does**. If you uploaded an older export, it won't include the signing subkey.

In GitLab:
1. **Avatar → Edit profile → GPG Keys**
2. If an old version of this key is there, click **Remove** (not *Revoke*: revoking marks commits signed with it as unverified).
3. **Add new key** → paste everything from `-----BEGIN PGP PUBLIC KEY BLOCK-----` to `-----END PGP PUBLIC KEY BLOCK-----` → **Add key**.

Check that GitLab lists:
- The primary fingerprint
- **Both subkeys**, including your `SIGNING_ID`
- Your email with a **Verified** badge

---

## 11. Test

```bash
git commit --allow-empty -m "Test signed commit from YubiKey"
```
Creates a commit with no file changes (`--allow-empty`), signed automatically because of `commit.gpgsign`. Expect:
1. A PIN prompt (once per session, because of "Require PIN for signature: Once")
2. The YubiKey blinking: **touch it**

```bash
git log --show-signature -1
```
Shows the last commit with its signature check. Look for `Good signature from "Jane Doe <jane@example.com>"`.

```bash
git push
```
In GitLab, the commit should now show a green **Verified** badge.

---

## 12. Optional: take the primary key offline

The primary key is still on your computer (`sec` without `#`). It can create new subkeys, so it is the most valuable key. After you have **tested restoring** `master-private.asc` on another machine or in a temporary `GNUPGHOME`:

```bash
gpg --delete-secret-keys PRIMARY_ID
```
Deletes all secret material for this key from the keyring, including the card stubs.

```bash
gpg --card-status
```
Reading the card recreates the stubs for the subkeys on the YubiKey.

```bash
gpg --list-secret-keys --keyid-format=long
```
You should now see `sec#` (primary key offline) and `ssb>` (subkeys on card).

When you need the primary key again (extend expiry, add a subkey), import `master-private.asc` on a trusted, ideally offline, machine.

---

## Using the YubiKey on another computer

```bash
gpg --import public.asc
```
Imports your public key (or use `gpg --recv-keys` if you published it to a keyserver).

```bash
gpg --card-status
```
Detects the card and links its keys to the imported public key.

Then repeat steps 8 and 9 on that computer.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `keytocard` → "Invalid selection" | The selected subkey doesn't match the slot you picked (e.g. `[E]` key into slot 1) | Check which key has the `*`. Key 1 = `[E]` → slot 2, key 2 = `[S]` → slot 1 |
| `ykman` hangs at "Enter Admin PIN" | Input is hidden, or `scdaemon` is holding the card | `gpgconf --kill scdaemon` and pass `--admin-pin` / `--force` as in step 7 |
| `PIN tries remaining: 1` | Wrong user PINs entered | `gpg --card-edit` → `admin` → `passwd` → `2` (unblock) with the admin PIN |
| `gpg failed to sign the data` | No PIN prompt possible | Set `GPG_TTY` (step 9); run `echo test \| gpg --clearsign` to see the real error |
| Commit shows **Unverified** on GitLab | Commit email ≠ key email ≠ verified GitLab email, or GitLab's key doesn't have the signing subkey | Align the emails; re-export and re-upload the public key (step 10) |
| `No secret key` when signing | Git is using the wrong key ID, or the card isn't detected | Check `user.signingkey` has the `!`; run `gpg --card-status` |
| Card not detected on Linux | `pcscd` not running | `sudo systemctl enable --now pcscd` |
| Commit waits forever | YubiKey is waiting for a touch | Tap the key |

---

## Command cheat sheet

| Command | Purpose |
|---|---|
| `gpg --list-secret-keys --keyid-format=long` | List your keys (`>` = on card, `#` = offline) |
| `gpg --edit-key ID` | Edit a key (add subkeys, move to card, change expiry) |
| `gpg --card-status` | Show the card and link it to local keys |
| `gpg --card-edit` | Manage the card (PINs, name, etc.) |
| `gpg --armor --export ID` | Export the public key (for GitLab) |
| `gpg --armor --export-secret-keys ID` | Back up private keys |
| `ykman openpgp info` | PIN counters, key slots, touch policies |
| `ykman openpgp keys set-touch SLOT POLICY` | Require a physical touch |
| `gpgconf --kill scdaemon` | Release the card (fixes most hangs) |
| `git log --show-signature` | Verify commit signatures locally |
