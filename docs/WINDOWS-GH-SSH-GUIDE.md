# GitHub CLI + SSH on Windows

A guide for people setting up GitHub SSH access on Windows 10/11 with the
GitHub CLI (`gh`). The step-by-step file for AI agents is
[../INSTRUCTIONS-WINDOWS.md](../INSTRUCTIONS-WINDOWS.md). This page covers the
same setup and explains the Windows-specific parts.

All commands are for **PowerShell** unless noted.

---

## How the pieces fit

| Piece | Where it lives on Windows | Job |
|---|---|---|
| `gh` (GitHub CLI) | `C:\Program Files\GitHub CLI\gh.exe` | Signs you in, creates the SSH key and uploads it to GitHub |
| OpenSSH client | `C:\Windows\System32\OpenSSH\` | `ssh`, `ssh-add`, `ssh-keygen` |
| `ssh-agent` | A Windows **service** | Holds your unlocked key so you type the passphrase once |
| Git for Windows | `C:\Program Files\Git\` | Includes **its own** `ssh.exe`, which does *not* use the Windows agent service |
| Your keys | `%USERPROFILE%\.ssh\` | `id_ed25519` (private, never share it) and `id_ed25519.pub` (public) |

Most SSH problems on Windows come from the last two rows of tooling: Git for
Windows has its own SSH, and by default it ignores the key you loaded into the
Windows agent. Step 4 fixes that.

---

## 1. Install the tools

Check what you already have:

```powershell
git --version
gh --version
ssh -V
```

Install anything missing with `winget`, which comes with Windows 10 (1809+) and Windows 11:

```powershell
winget install --id Git.Git   -e --source winget
winget install --id GitHub.cli -e --source winget
```

If `ssh -V` fails, enable the built-in OpenSSH client from an **administrator**
PowerShell:

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
```

> **Close and reopen your terminal after installing.** The installers update
> `PATH`, but windows that are already open don't see the change. If `gh` is still
> "not recognized", run it by its full path to confirm it installed:
> `& "C:\Program Files\GitHub CLI\gh.exe" --version`.

Other ways to install: `scoop install git gh`, `choco install git gh` (admin
shell), or the `.msi` from <https://cli.github.com>.

---

## 2. Sign in with `gh` and choose SSH

Check for existing keys first:

```powershell
Get-ChildItem "$env:USERPROFILE\.ssh\*.pub" -ErrorAction SilentlyContinue
```

Then sign in:

```powershell
gh auth login
```

| Prompt | Choose |
|---|---|
| Where do you use GitHub? | `GitHub.com` |
| Preferred protocol for Git operations? | **`SSH`** |
| Upload your SSH public key / generate a new one? | Pick an existing `.pub` if you have one, otherwise generate a new key |
| Passphrase | Use one. Step 3 means you type it only once |
| Title for your SSH key | e.g. `DESKTOP-ABC123 (gh)` |
| How would you like to authenticate? | `Login with a web browser` |

`gh` shows a one-time code and opens your browser. After you approve, it uploads
the public key to your GitHub account. A new key is saved as
`%USERPROFILE%\.ssh\id_ed25519`.

### Sign in without the questions (HTTPS, browser)

To skip the questions, or if `gh` isn't on `PATH` yet because the terminal
was opened before the install, call it by its full path with the answers given
as flags:

```powershell
& "C:\Program Files\GitHub CLI\gh.exe" auth login --hostname github.com --git-protocol https --web
```

It prints a one-time code and opens the browser. **Check which GitHub account
the browser is signed in to before you approve.** If it's a different account
(for example a bot or work account), open a private window, go to
<https://github.com/login/device>, sign in as the account you want, and enter
the code there. Answer **Yes** to "Authenticate Git with your GitHub
credentials?" so `git push` uses this sign-in.

Confirm the account afterwards with
`& "C:\Program Files\GitHub CLI\gh.exe" auth status`.

Then let `gh` configure git for any HTTPS remotes:

```powershell
gh auth setup-git
```

If you already signed in with HTTPS, switch the protocol without signing in again:

```powershell
gh config set git_protocol ssh
```

---

## 3. Start the Windows `ssh-agent` and add your key

The agent ships **disabled** on many machines. Enable it once from an
**administrator** PowerShell:

```powershell
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent
```

Then, in a normal PowerShell, add your key. It asks for the passphrase:

```powershell
ssh-add $env:USERPROFILE\.ssh\id_ed25519
ssh-add -l   # should list the key's fingerprint
```

Unlike macOS, you don't need a Keychain step. The Windows agent service keeps
added keys across reboots.

---

## 4. Make Git use Windows OpenSSH

Git for Windows uses its own `ssh.exe` by default. That one can't see the
Windows agent, so `git push` keeps asking for the passphrase or fails with
`Permission denied (publickey)`. Point Git at the system OpenSSH:

```powershell
git config --global core.sshCommand "C:/Windows/System32/OpenSSH/ssh.exe"
```

Check for an existing value first with `git config --global core.sshCommand`.
If something is already set on purpose, don't overwrite it.

> **Git Bash note:** inside Git Bash, `ssh` and `ssh-add` run Git's bundled
> copies, which also can't reach the Windows agent. Run `ssh-add` from
> PowerShell, or call `/c/Windows/System32/OpenSSH/ssh-add.exe` explicitly.

---

## 5. Verify

```powershell
gh auth status          # shows your account and "Git operations protocol: ssh"
ssh-add -l              # lists your key
ssh -T git@github.com   # "Hi <user>! You've successfully authenticated..."
```

`ssh -T` exits with code 1 even when it works. That's normal, because GitHub
doesn't offer a shell. The first time, it asks you to trust GitHub's host key:
type `yes`.

Try a real clone:

```powershell
gh repo clone <owner>/<repo>
# or
git clone git@github.com:<owner>/<repo>.git
```

---

## Pushing a branch from any directory

`git -C <path>` runs git as if you were inside `<path>`, so you don't need to
`cd` first. For example, to push this repo's docs branch and set it to track
`origin`:

```powershell
git -C "C:\Users\Public\github.com\carlosmarte\LDE-agents-github-ssh-setup" push -u origin docs/windows-gh-ssh-guide
```

The general form is `git -C "<repo path>" push -u origin <branch>`. If the
push fails with `Permission ... denied to <user>`, the credential git used
belongs to an account without write access. Check with `gh auth status` and
`ssh -T git@github.com`.

---

## More than one key

If `%USERPROFILE%\.ssh\` holds several keys (for example a personal key and a
deploy key), tell SSH which one to use for GitHub. Create or edit
`%USERPROFILE%\.ssh\config` (no file extension):

```
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
```

For a second identity, such as a deploy key for one repo, add a host alias:

```
Host github-deploy
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_deploy
  IdentitiesOnly yes
```

Then clone with the alias: `git clone git@github-deploy:<owner>/<repo>.git`.

To see which keys GitHub knows about: `gh ssh-key list`. To upload another
public key: `gh ssh-key add $env:USERPROFILE\.ssh\<key>.pub --title "<name>"`.

---

## Switching an existing repo from HTTPS to SSH

```powershell
git remote -v
git remote set-url origin git@github.com:<owner>/<repo>.git
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `gh : The term 'gh' is not recognized` right after installing | Terminal opened before the install | Open a new terminal, or use the full path `C:\Program Files\GitHub CLI\gh.exe` |
| `Error connecting to agent: No such file or directory` | `ssh-agent` service is stopped or disabled | Step 3 (admin PowerShell) |
| `Start-Service` fails with "access denied" | Not an elevated shell | Right-click PowerShell, choose **Run as administrator**, run it once |
| `git push` asks for the passphrase every time | Git uses its bundled `ssh.exe` | Step 4 (`core.sshCommand`) |
| `Permission denied (publickey)` | Key not uploaded, or the wrong key is offered | `gh ssh-key list`, then `ssh -vT git@github.com` to see which key is tried. Use `IdentityFile` + `IdentitiesOnly` in `~/.ssh/config` |
| `WARNING: UNPROTECTED PRIVATE KEY FILE!` | Key file permissions are too open | `icacls "$env:USERPROFILE\.ssh\id_ed25519" /inheritance:r /grant:r "$($env:USERNAME):F"` |
| `Host key verification failed` | GitHub's host key isn't trusted yet, or has changed | Run `ssh -T git@github.com` and answer `yes`. If it reports a *changed* key, check against <https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints> before removing the old line from `~/.ssh/known_hosts` |
| Port 22 blocked on a corporate network | Firewall | Use SSH over 443: in `~/.ssh/config` set `HostName ssh.github.com` and `Port 443` under `Host github.com` |

---

## Safety rules

- Never share, paste or upload the private key (the file **without** `.pub`).
- Use a passphrase. The agent means you only type it once per login.
- Prefer `ed25519` keys. Use RSA only if something strictly needs it.
- To revoke a key, delete it from <https://github.com/settings/keys> or run
  `gh ssh-key delete <id>`.
