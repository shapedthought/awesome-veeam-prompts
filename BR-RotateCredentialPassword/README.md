# Rotate a VBR credentials record password across multiple servers

Generates a script that rotates the password on a Veeam Backup & Replication "Credentials Record" across many backup servers in one pass, using the VBR REST API. Useful when the same service account is registered on several VBR servers and its password has to change everywhere at once.

Two variants are provided, plus a Docker add-on:

* [Python](#python-prompt): processes servers sequentially. Simple to read and adapt.
* [Python in Docker](#docker-add-on-for-the-python-variant): an add-on to the Python prompt that also produces a Dockerfile, for hosts with no Python or a locked-down environment.
* [Go](#go-prompt-parallel): processes servers concurrently with goroutines and a configurable worker limit. Better for large estates.

Both prompts are designed to be pasted together with the [API contract](#api-contract) block, which pins the endpoints and schemas the model must use.

| | |
| --- | --- |
| **Product** | Veeam Backup & Replication 13 |
| **Interface** | REST API 1.3-rev2 |
| **Output** | Python script, or Go binary (parallel) |
| **Tested with** | Claude (Go variant generated, built, and run with `go vet` clean; the API contract, lookup logic, and `/changepassword` endpoint were exercised against a live server). The `-dry-run` / `--dry-run` flag, `-ignore-case` flag, `passwordEnv` config handling, and the mock-server smoke test were added afterwards and have not yet been exercised. |
| **Endpoints verified** | 2026-09-03, against a live VBR 13 server and its OpenAPI document |
| **Author** | [@shapedthought](https://github.com/shapedthought) |
| **Origin** | [Original gist](https://gist.github.com/shapedthought/c4a58d593c09c6fd0416b641cd9e9936) |

## API contract

Paste this block **together with** either prompt below. It exists because the model cannot reach your VBR server to check anything: without it, a wrong endpoint is only discovered when the script runs against production.

Everything here was confirmed against a live VBR 13 server on 2026-09-03 — the endpoints by live call, the schemas and query semantics from that server's OpenAPI document.

```text
## API contract -- treat as ground truth

The endpoints, headers and schemas below were verified against VBR 13, REST API
revision 1.3-rev2. Implement exactly these. Do NOT substitute similar-looking
paths, and do NOT probe for alternatives.

Auth
  POST /api/oauth2/token
  Header: x-api-version: 1.3-rev2   (required on EVERY request)
  Body:   application/x-www-form-urlencoded
          grant_type=password&username=<u>&password=<p>
  200 ->  {"access_token", "token_type", "expires_in", "refresh_token",
           ".issued", ".expires"}
  Subsequent calls: Authorization: Bearer <access_token>

Lookup
  GET /api/v1/credentials?nameFilter=<username>
  Query params: skip, limit, orderColumn, orderAsc, nameFilter, typeFilter,
                createdAfterFilter, createdBeforeFilter,
                includeDefaultApplianceCreds
  200 ->  {"data": [ ... ], "pagination": { ... }}
  Record fields: id, username, type, description, creationTime, uniqueId,
                 isDefaultAppliance  (+ SSHPort, authenticationType,
                 elevateToRoot, useSu, addToSudoers on Linux records)

  nameFilter semantics: EXACT match unless you use * wildcards at the start
  and/or end -- BUT it matches any field of a record, not just the username.
  A record whose *description* equals the search term also comes back. The
  filter is a narrowing device, not a match: re-check the username yourself.

Change password
  POST /api/v1/credentials/{id}/changepassword     <-- note: NOT /password
  Body: application/json  {"password": "<new>"}    (schema
        CredentialsPasswordChangeSpec; `password` is the only field, required)
  Responses: 200, 400, 401, 403, 404, 500

Related paths that exist and must NOT be substituted:
  GET|PUT|DELETE /api/v1/credentials/{id}
  POST /api/v1/credentials/{id}/changeprivatekey
  POST /api/v1/credentials/{id}/changerootpassword

If a request returns 404 or 400 during testing, STOP and report it to me with
the request and the response. Do not try alternative endpoint paths -- this
tool writes to production backup infrastructure, and guessing is not
acceptable. You will not have access to a live server or its OpenAPI document.
```

## Example config file

Paste this alongside either prompt so the model has the exact shape. Note that there is no password anywhere in it. The first server names its own variable, the second uses the shared fallback.

```json
{
  "servers": [
    {
      "host": "vbr01.lab.local",
      "port": 9419,
      "username": "LAB\\Administrator",
      "passwordEnv": "VBR01_ADMIN_PASSWORD",
      "credentialUsername": "LAB\\svc_backup"
    },
    {
      "host": "vbr02.lab.local",
      "username": "DMZ\\Administrator",
      "credentialUsername": "DMZ\\svc_backup"
    }
  ]
}
```

With that config the tool reads `VBR01_ADMIN_PASSWORD` for the first server, `VBR_ADMIN_PASSWORD` for the second, and `VBR_NEW_PASSWORD` for the value being set.

## Python prompt

```text
I need a Python script that rotates the password on a Veeam Backup & Replication (VBR) "Credentials Record" across multiple VBR backup servers, using the VBR REST API (v13, REST API revision 1.3-rev2). Use the API contract block I have pasted alongside this prompt as ground truth for all endpoints and schemas.

Reference docs: https://helpcenter.veeam.com/references/vbr/13/rest/1.3-rev2/tag/Credentials

Requirements:

1. Auth: Authenticate against each server's token endpoint (password grant) to get a bearer token. Include the `x-api-version: 1.3-rev2` header on all requests.
2. Server list: Read a list of target VBR servers from a JSON config file. Each entry has `host`, `port` (default 9419), the admin `username` used to authenticate to that server, an optional `passwordEnv` naming the environment variable that holds that server's admin password, and a `credentialUsername` naming the credentials record to rotate on that server (it may differ per server). If `passwordEnv` is omitted, fall back to the shared `VBR_ADMIN_PASSWORD` variable. The config file must never contain a password: do not accept a plaintext `password` field, and fail with a clear message if one is present. Resolve every referenced environment variable at startup and exit with a list of the missing or empty ones before contacting any server, so a typo cannot produce a half-rotated estate.
3. Find the record: On each server, look up the credentials record with the `nameFilter` query param, then re-check the username yourself with an exact, case-sensitive comparison — `nameFilter` matches any field of a record, so it narrows, it does not match. Linux credential records are case-sensitive, so do not relax this by default. Provide an `--ignore-case` flag for estates that are all Windows accounts. Zero matches and multiple matches are both errors: never guess which record to modify. Duplicate usernames do occur on real servers, and the multiple-match rule applies even with `--ignore-case`. When there is no exact match, make the error list any case-insensitive near-misses from the filter results (for example "no exact match for `lab\svc_backup`, did you mean `LAB\svc_backup`?") so the operator can fix the config.
4. Change password: Use the change-password endpoint from the API contract block.
5. Dry run: Support a `--dry-run` flag that performs the login and the lookup, reports the record that would be modified, and makes no write. Make the summary state clearly that it was a dry run.
6. Security: Take the new password from an environment variable (`VBR_NEW_PASSWORD`), not a CLI argument, so it doesn't end up in shell history. Never log the new password, any resolved admin password, or bearer tokens, and redact all of them if they appear in an API error body — servers do echo submitted values back in error messages.
7. TLS: Support an `--insecure` flag to skip certificate verification, for self-signed VBR server certs. Log a warning when it is enabled.
8. Resilience: Process all servers in the list even if one fails (log the error, continue), then print a summary of which servers succeeded/failed at the end, and exit with a non-zero code if any failed.
9. Logging: Use Python's `logging` module with timestamps, prefixed per-server, rather than bare `print()`.

Use the `requests` library. Structure it with a small class wrapping the per-server session/auth/calls, and a `main()` with `argparse` for the CLI. Include a docstring at the top explaining usage, the config file format, and the environment variables the script reads, with an example of loading them from a `.env` file using `set -a; source .env; set +a` and a note that the `.env` file, not the config, is the one that must be mode 0600 and never committed.

Before showing me the result, smoke-test the script against a throwaway mock HTTP server implementing the API contract, covering: success; username not found; two records sharing the target username; failed admin login; and a server that is unreachable. Confirm the summary and the exit code in each case. Do not test against a real VBR server — I will run `--dry-run` against one myself.

(If your model cannot execute code, delete the paragraph above and run those cases yourself.)
```

## Docker add-on for the Python variant

If the target host has no Python, or you cannot install packages on it, paste the Python prompt above and then append the block below. It asks the model to package the same script as a container image without changing the script itself.

```text
Also package the script as a container image:

1. Dockerfile: Use a pinned slim official Python base image (for example `python:3.12-slim`). Copy in a `requirements.txt` containing `requests` with a pinned version, install with `pip install --no-cache-dir`, and copy the script last so dependency layers are cached.
2. Non-root: Create an unprivileged user and switch to it before the `ENTRYPOINT`.
3. Entrypoint: Set `ENTRYPOINT ["python", "/app/rotate_vbr_password.py"]` so any CLI flags passed to `docker run` go straight to the script.
4. Config outside the image: The JSON config file must never be copied into the image. Document mounting it read-only at run time, for example `-v /path/to/servers.json:/config/servers.json:ro`, and have the script default its `--config` path to `/config/servers.json`.
5. Secret handling: The script reads several variables (`VBR_NEW_PASSWORD`, `VBR_ADMIN_PASSWORD`, and any per-server `passwordEnv` names). Document passing them with `--env-file .env` as the primary method, and `-e NAME` with no value (so Docker inherits it from the caller's environment) as the alternative for one or two variables. Never show `-e NAME=<value>` anywhere, since that lands in shell history and `docker inspect` output.
6. Usage: Add a section to the top-of-file docstring and a short `README` snippet showing `docker build` and a full `docker run` example that includes the config mount, the environment variable, and the `--insecure` flag pass-through.
7. Keep the script unchanged apart from the default config path. It must still run directly with `python` outside the container.
```

Example of the `docker run` shape the model should produce:

```bash
# .env holds VBR_NEW_PASSWORD, VBR_ADMIN_PASSWORD, VBR01_ADMIN_PASSWORD, ...
chmod 600 .env
docker run --rm \
  -v "$PWD/servers.json:/config/servers.json:ro" \
  --env-file .env \
  vbr-rotate-password:latest --insecure --dry-run
```

## Go prompt (parallel)

```text
I need a Go program that rotates the password on a Veeam Backup & Replication (VBR) "Credentials Record" across multiple VBR backup servers in parallel, using the VBR REST API (v13, REST API revision 1.3-rev2). Use the API contract block I have pasted alongside this prompt as ground truth for all endpoints and schemas.

Reference docs: https://helpcenter.veeam.com/references/vbr/13/rest/1.3-rev2/tag/Credentials

Requirements:

1. Auth: Authenticate against each server's token endpoint (password grant, form-encoded body) to get a bearer token. Include the `x-api-version: 1.3-rev2` header on all requests.
2. Server list: Read a list of target VBR servers from a JSON config file passed via a `-config` flag. Each entry has `host`, `port` (default 9419), the admin `username` used to authenticate to that server, an optional `passwordEnv` naming the environment variable that holds that server's admin password, and a `credentialUsername` naming the credentials record to rotate on that server (it may differ per server). If `passwordEnv` is omitted, fall back to the shared `VBR_ADMIN_PASSWORD` variable. The config file must never contain a password: do not accept a plaintext `password` field, and fail with a clear message if one is present. Resolve every referenced environment variable at startup and exit with a list of the missing or empty ones before any goroutine is started, so a typo cannot produce a half-rotated estate.
3. Find the record: On each server, look up the credentials record with the `nameFilter` query param, then re-check the username in the results with an exact, case-sensitive comparison — `nameFilter` matches any field of a record, so it narrows, it does not match. Linux credential records are case-sensitive, so do not relax this by default. Provide an `-ignore-case` flag (using `strings.EqualFold`) for estates that are all Windows accounts. Zero matches and multiple matches are both errors for that server: never guess which record to modify. Duplicate usernames do occur on real servers, and the multiple-match rule applies even with `-ignore-case`. When there is no exact match, make the error list any case-insensitive near-misses from the filter results (for example "no exact match for `lab\svc_backup`, did you mean `LAB\svc_backup`?") so the operator can fix the config.
4. Change password: Use the change-password endpoint from the API contract block.
5. Dry run: Add a `-dry-run` flag that performs the login and the lookup, logs the record that would be modified, and makes no write. The summary table must state clearly that it was a dry run.
6. Security: Take the new password from an environment variable (`VBR_NEW_PASSWORD`), not a CLI flag, so it doesn't end up in shell history. Never log the new password, any resolved admin password, or bearer tokens, and redact all of them if they appear in an API error body — servers do echo submitted values back in error messages.
7. TLS: Support an `-insecure` flag that sets `InsecureSkipVerify` on the TLS config, for self-signed VBR server certs, and log a warning when it is enabled.
8. Concurrency: Process servers in parallel using goroutines. Add a `-concurrency` flag (default 5) that caps how many servers are worked on at once, using a buffered channel semaphore or `errgroup.SetLimit`. Use a `sync.WaitGroup` (or errgroup) to wait for all workers, and collect per-server results over a channel into a results slice. Do not use shared mutable state without a mutex.
9. Timeouts: Give each server a `context.Context` with a timeout (flag `-timeout`, default 60s) so one unreachable server cannot hang the run. Pass the context to every HTTP request.
10. Resilience: A failure on one server must not stop the others. Capture the error per server, continue, then print a summary table (server, credential, status, detail) in config-file order at the end, and exit with a non-zero code if any failed.
11. Logging: Use the standard `log/slog` package with a text handler, timestamps, and a `server` attribute on every line so interleaved output from goroutines is attributable.

Use only the Go standard library plus `golang.org/x/sync/errgroup` if you choose that route. Structure it as a single `main` package with:
- a `Server` struct for config entries,
- a `VBRClient` struct wrapping the base URL, `*http.Client`, and token, with methods `Login`, `FindCredentialID`, and `ChangePassword`,
- a `rotate(ctx, server) error` function that runs the three steps for one server,
- a `main()` that parses flags, loads config, fans out the work, and prints the summary.

Include a doc comment at the top explaining usage, the flags, the config file format, and the environment variables the program reads, with an example of loading them from a `.env` file using `set -a; source .env; set +a` and a note that the `.env` file, not the config, is the one that must be mode 0600 and never committed. Also include a `go.mod` with the module name and Go version.

Before showing me the result, smoke-test the program against a throwaway mock HTTP server implementing the API contract, covering: success; username not found; two records sharing the target username; failed admin login; and a server that hangs until the timeout fires. Confirm the summary table and the exit code in each case. Do not test against a real VBR server — I will run `-dry-run` against one myself.

(If your model cannot execute code, delete the paragraph above and run those cases yourself.)
```

## Notes

* **Run `--dry-run` / `-dry-run` against one server first, then a small subset, before a fleet-wide run.** The model cannot verify anything against your environment, so this is the check that replaces it.
* `nameFilter` is an exact match unless you use `*` wildcards, but it matches **any** field of a record, not just the username — a record whose *description* equals the search term also comes back. The generated code must re-check the username itself. Confirmed against 1.3-rev2; re-check if you are on another revision.
* Duplicate usernames are normal. An ordinary lab server used to verify this prompt had two separate records named `administrator`. Make sure the generated code treats multiple matches as an error rather than picking one.
* Username matching is case-sensitive by default. Windows accounts are case-insensitive in practice, but VBR also stores Linux credentials where `Admin` and `admin` are different users, and a loose match could rotate the wrong record silently. Copy the username exactly as the VBR console shows it, or pass `--ignore-case` / `-ignore-case` if every record on the target servers is a Windows account. On a miss, the tool should tell you the near-miss so the fix is a one-line config edit.
* Earlier versions of this prompt gave the change endpoint as `POST /api/v1/credentials/{id}/password`. That path does not exist and returns 404 — it is `/changepassword`. This is exactly the failure the API contract block is there to prevent, and a good illustration of why an LLM will faithfully implement a wrong detail you hand it.
* The config file holds no secrets by design. Admin passwords come from environment variables named per server via `passwordEnv`, with `VBR_ADMIN_PASSWORD` as the shared fallback, and the tool refuses a config that contains a plaintext `password` field. The file that does need protecting is whatever you load those variables from, typically a `.env` file: keep it mode 0600, add it to `.gitignore`, and load it with `set -a; source .env; set +a` rather than pasting values on the command line. If you have a secret store, ask the model to add a fetch step for it instead of the `.env` file.
* Missing variables should fail the run before any server is contacted. If you see a half-rotated estate after a run, that check is missing from the generated code.
* If you are on a different VBR version, change the version and API revision in the first line, the reference URL, **and the API contract block** — endpoint paths and the `x-api-version` header value differ between revisions. Do not carry the contract block across versions unverified.
* Check that the generated code handles token expiry if you have a long server list, and that it never logs the new password or the bearer token.
* If the problem is a messy Python environment rather than no Python at all, `uv run --with requests rotate_vbr_password.py` or `pipx run` is far less ceremony than building an image. Use Docker when the host has no Python or you cannot install anything on it.
* The container needs network access to every VBR server in the config. This is normally fine, but Docker Desktop on a VPN-connected laptop can route container traffic differently from the host, so test against one server first.
* For the Go variant, start with a low `-concurrency` value. Every worker opens its own auth session on a different VBR server, so the limit is mostly about how many parallel logins you are comfortable with, but very high values can trip rate limits or lockout policies.
* The Go variant should compile into a single static binary, which is handy for running from a jump host that has no Python installed.

## Disclaimer

See the [read this first](../README.md#%EF%B8%8F-read-this-first) section in the root README. This prompt is a starting point, not a finished solution. Check with your security team before using LLM-generated code that handles credentials, and review and test all generated output before running it against production infrastructure.
