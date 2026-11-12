# Dev setup

Linux, macOS, or Windows with WSL2. On Windows, keep the repo inside the WSL filesystem (e.g. `~/dev/invoice-relay`), not under `/mnt/c` or in a synced folder (OneDrive, Dropbox). That keeps Docker/Testcontainers I/O fast and the line endings right.

## Prerequisites
| Tool | Version | Notes |
|---|---|---|
| Git | recent | `git config --global core.autocrlf input` on WSL/Linux |
| Docker | Docker Desktop (WSL integration on) or Docker Engine | Testcontainers needs a running daemon |
| JDK | 25 | e.g. Temurin via SDKMAN |
| Node.js | active LTS | e.g. via fnm |

```bash
# JDK 25 (SDKMAN): pick the Temurin 25 build listed
curl -s "https://get.sdkman.io" | bash && source ~/.sdkman/bin/sdkman-init.sh
sdk list java | grep -E '25(\.[0-9]+)*-tem'
sdk install java <25.x.y-tem>

# Node active LTS (fnm)
curl -fsSL https://fnm.vercel.app/install | bash && exec $SHELL
fnm install --lts && fnm default lts-latest
```

## Check
```bash
java -version                              # 25
node -v                                    # LTS
docker info >/dev/null && echo docker-ok
```

## Run
Backend and frontend commands (`./mvnw -B verify`, `npm run check`, `docker compose up`) arrive with the first v1 milestone.
