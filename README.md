# GIFT

**Git Identity Forensics Tool ( GIFT )** is an OSINT tool for gathering intelligence on GitHub accounts. It comes with a web UI, a JSON API and a CLI, written in Go.

It ships as two binaries: **`gift-server`** (web UI + JSON API) and **`gift-cli`** (command-line tool that prints JSON to stdout).

> **Use responsibly.** GIFT is intended for lawful, authorized use only. See [Responsible Use](#responsible-use) before running it against anyone.

---

![GIFT WebUI](https://github.com/ashishpatil243/GIFT/blob/main/img/img-1.png?raw=true)

---

![GIFT CLI](https://github.com/ashishpatil243/GIFT/blob/main/img/img-2.png?raw=true)

---

## Features

- OSINT on GitHub accounts from a **username**, **profile URL**, or **`user/repo` URL**
- Web UI and a simple `POST /api/scan` JSON API (`gift-server`)
- Terminal-friendly CLI with JSON output that is easy to pipe into `jq` or other tools (`gift-cli`)
- Fast, concurrent scanning with a configurable number of workers
- Configurable via flags, environment variables or a `.env` file
- Lightweight single binaries

## Requirements

- **`git`** installed and available on your `PATH` (the server checks for it at scan time)
- Outbound internet access to `github.com` and `api.github.com`
- A `ui/` directory next to the binary (the frontend is served from `./ui`, configurable with `-ui` / `UI_DIR`)

## Installation

### Download a release

1. Download the binary for your platform from the [Releases](https://github.com/ashishpatil243/GIFT/releases) page.
2. Place it next to the `ui/` folder, so the layout looks like this:

   ```
   gift/
   ├── gift-server      
   ├── gift-cli
   ├── .env.example
   └── ui/
       └── index.html ...
   ```

3. On Linux/macOS, make it executable:

   ```bash
   chmod +x gift-server gift-cli
   ```
   
## Usage

### Web server

Run the server **from the directory that contains `ui/`**:

```bash
./gift-server
```

```
Server running at http://127.0.0.1:8080
```

Open <http://localhost:8080> in your browser and enter a target.

#### Server options

Run `./gift-server -h` to see the built-in help.

| Flag            | Env var  | Default     | Description                                                          |
| --------------- | -------- | ----------- | -------------------------------------------------------------------- |
| `-port <n>`     | `PORT`   | `8080`      | Port to listen on (1-65535)                                          |
| `-host <addr>`  | `HOST`   | `127.0.0.1` | Address to bind to. Use `0.0.0.0` to listen on all interfaces        |
| `-ui <dir>`     | `UI_DIR` | `./ui`      | Directory containing the web UI                                      |
| `-env <file>`   | -        | `.env`      | Path to an optional `.env` file                                      |
| -               | `WORKERS`| `10`        | Default worker count when an API request omits `workers`             |
| `-h`            | -        | -           | Show help and exit                                                   |

Examples:

```bash
./gift-server -port 9090
./gift-server -host 0.0.0.0 -port 3000
PORT=9090 ./gift-server
```

#### Configuration with `.env`

`gift-server` and `gift-cli` share the same `.env` format. Copy the example file and edit it. Each program loads `.env` from the current directory on startup (a missing file is fine):

```bash
cp .env.example .env
```

```env
PORT=8080
HOST=127.0.0.1
UI_DIR=./ui
WORKERS=10
```

Settings are applied in this order, highest priority first: **command-line flags**, **real environment variables**, **`.env` file**, **built-in defaults**.

> Add `.env` to your `.gitignore` so local settings are never committed.

### Accepted target formats

| Input                                    | Behaviour                         |
| ---------------------------------------- | --------------------------------- |
| `octocat`                                | Scans all non-fork public repos   |
| `https://github.com/octocat`             | Same as above                     |
| `https://github.com/octocat/Hello-World` | Scans only that single repository |

### CLI

`gift-cli` runs a single scan and writes the result as JSON to **stdout**. Errors are also reported as JSON on stdout (see [API](#api) for the shape), so the output is always parseable.

```bash
./gift-cli [options] <target>
```

| Flag           | Env var   | Default | Description                                 |
| -------------- | --------- | ------- | ------------------------------------------- |
| `-workers <n>` | `WORKERS` | `10`    | Number of parallel git workers              |
| `-env <file>`  | -         | `.env`  | Path to an optional `.env` file             |
| `-h`           | -         | -       | Show help and exit                          |

`<target>` accepts the same formats as the server (username, profile URL, or repository URL).

```bash
./gift-cli octocat
./gift-cli -workers 4 https://github.com/octocat/Hello-World
WORKERS=4 ./gift-cli octocat | jq '.target_identity.email'
```

The CLI needs only `git` and internet access. It does not need the `ui/` folder or a port.

## API

### `POST /api/scan`

**Request body**

```json
{
  "target": "octocat",
  "workers": 10
}
```

| Field     | Type   | Required | Description                                       |
| --------- | ------ | -------- | ------------------------------------------------- |
| `target`  | string | yes      | Username, profile URL, or repository URL          |
| `workers` | int    | no       | Number of concurrent clone workers (default `10`) |

**Example**

```bash
curl -X POST http://localhost:8080/api/scan \
  -H "Content-Type: application/json" \
  -d '{"target":"octocat","workers":5}'
```

**Success response (`200`)**

```json
{
  "target": "octocat",
  "scanned_at": "2026-01-01T12:00:00Z",
  "target_identity": {
    "email": "octocat@example.com",
    "associated_names": ["The Octocat"],
    "contributed_repositories": ["Hello-World"],
    "oldest_activity": {
      "repo_name": "Hello-World",
      "commit_hash": "abc123...",
      "timestamp": "2020-01-01T00:00:00Z"
    },
    "latest_activity": {
      "repo_name": "Hello-World",
      "commit_hash": "def456...",
      "timestamp": "2026-01-01T00:00:00Z"
    }
  },
  "collaborator_identities": []
}
```

**Error responses**

| Status | Meaning                                                  |
| ------ | -------------------------------------------------------- |
| `400`  | Missing or invalid request body / `target`               |
| `404`  | User not found, or no repositories discovered            |
| `405`  | Method other than `POST`                                 |
| `500`  | Required component missing (e.g. `git` is not installed) |

```json
{
  "target": "octocat",
  "scanned_at": "2026-01-01T12:00:00Z",
  "error": "Target user or organization context not found."
}
```

## How it works

1. **Resolve the target** into a username (and optionally a single repository).
2. **Discover repositories** via `api.github.com/users/<user>/repos`, falling back to the user's public Atom feed on rate limits or errors.
3. **Clone** each non-fork repository as a bare, tree-less clone into a temporary directory (up to 3 attempts with backoff).
4. **Read commit metadata** with `git log` and record the author name, email, hash, and timestamp.
5. **Cluster by email**, then pick the most likely "target identity" (email handle or name matching the username, otherwise the identity seen in the most repositories). All other identities are returned as `collaborator_identities`.
6. **Delete** all temporary clones.

## Limitations

- Uses the **unauthenticated** GitHub API, which is limited to roughly 60 requests per hour per IP. The Atom fallback is best-effort and may miss repositories.
- Only the first 100 repositories from the API are considered (`per_page=100`, no pagination).
- Only a repository's boundary commits are inspected, not its full author list, so the output is a sample, not a complete roster of contributors.
- Private repositories, and repositories that require authentication, are skipped.
- Developers who use GitHub's `noreply` addresses, or who rewrite their history, will not show a real email.
- Results are heuristic. A matching name or email is **not** proof of identity.

## Security notes

- The server binds to **`127.0.0.1` by default**, so it is only reachable from the same machine. It has **no authentication**, so if you set `HOST=0.0.0.0` put it behind a firewall or an authenticated reverse proxy. Do not expose it directly to the internet.
- Each scan makes the host clone repositories from GitHub, which consumes bandwidth, disk, and CPU. Keep `workers` at a sensible value.

## Responsible use

GIFT reads data that is publicly visible in git commit history, but collecting and aggregating it can still affect people's privacy. By using this tool you agree to:

- Use it only for legitimate purposes, such as auditing **your own** accounts and repositories, authorized security research, or checking what your own commits expose
- **Not** use it to harass, stalk, dox, or spam anyone, or to build contact lists for unsolicited messages (this also violates [GitHub's Acceptable Use Policies](https://docs.github.com/en/site-policy/acceptable-use-policies/github-acceptable-use-policies))
- Comply with applicable laws and regulations, including privacy laws such as GDPR where relevant
- Respect GitHub's Terms of Service and rate limits

> **Tip for developers:** if you don't want your email exposed in commits, enable *"Keep my email addresses private"* and *"Block command line pushes that expose my email"* in your GitHub email settings, and use your `noreply` address for `git config user.email`.

The authors are not responsible for misuse of this software.

## Troubleshooting

| Problem                                                  | Fix                                                            |
| -------------------------------------------------------- | -------------------------------------------------------------- |
| `Missing required component: 'git' must be installed...` | Install git and make sure it is on your `PATH`                 |
| Blank page or 404 at `/`                                 | Run the binary from the folder that contains `ui/`             |
| `No repositories discovered for target identifier.`      | Check the spelling, or you may be rate-limited. Wait and retry |
| Port already in use                                      | Pick another one: `./gift-server -port 9090` or set `PORT`     |
| Can't reach the server from another device               | Set `HOST=0.0.0.0` (and see the security notes)                |
| `Invalid port ...`                                       | `PORT` must be a number between 1 and 65535                    |
