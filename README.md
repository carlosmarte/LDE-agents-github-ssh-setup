# AI-Agent-LDE-GITHUB-SSH

Instructions for an LLM agent (Claude Code or similar) that walks a user through
setting up GitHub SSH access with the GitHub CLI (`gh`).

## How an agent should use this repo

1. Detect the user's operating system (`uname -s` on macOS/Linux, `$env:OS` or
   `[System.Environment]::OSVersion` in PowerShell on Windows). Do not ask the
   user if the shell already tells you.
2. Open the matching instruction file and follow it top to bottom:

   | OS | File |
   |---|---|
   | macOS | [INSTRUCTIONS-MACOS.md](INSTRUCTIONS-MACOS.md) |
   | Windows | [INSTRUCTIONS-WINDOWS.md](INSTRUCTIONS-WINDOWS.md) |
   | Linux | [INSTRUCTIONS-LINUX.md](INSTRUCTIONS-LINUX.md) |

3. Every file follows the same four steps:

   | Step | What happens |
   |---|---|
   | 0 | Check `git` and `gh` are installed. If not, tell the user how to install them and stop until they have. |
   | 1 | Authenticate with `gh auth login` using the SSH protocol, then run `gh auth setup-git`. |
   | 2 | Load the key into the SSH agent so the passphrase is not asked for on every push. |
   | 3 | Verify with `gh auth status` and `ssh -T git@github.com`. |

## Guides for people

- [docs/WINDOWS-GH-SSH-GUIDE.md](docs/WINDOWS-GH-SSH-GUIDE.md): GitHub CLI +
  SSH on Windows, including the Git-for-Windows `ssh.exe` problem, multiple keys
  and troubleshooting.

## Rules that apply on every platform

- **Never generate, read, print, or move a private key yourself.** `gh auth login`
  generates the key for the user. Only ever reference the `.pub` file.
- **Never run `gh auth login` from the agent's own shell.** It is interactive and
  opens a browser. Hand the command to the user. In Claude Code, tell them to
  type it with the `!` prefix (for example `! gh auth login`) so it runs in
  their session and the output lands back in the conversation.
- **Same for `ssh-add`** when the key has a passphrase. The prompt must reach the
  user's terminal, so give it to them as a `!` command.
- **Do not install anything on the user's behalf** in Step 0. Suggest the install
  command, explain what it does, and wait for them to confirm it finished.
- Prefer `ed25519` keys. Only fall back to RSA if the user has a hard requirement.
- If the user already has a working SSH key and `ssh -T git@github.com` succeeds,
  skip key generation and only run `gh auth login` plus `gh auth setup-git`.
