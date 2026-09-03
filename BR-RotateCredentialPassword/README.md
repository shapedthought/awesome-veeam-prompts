# Rotate a VBR credentials record password across multiple servers

Generates a script that rotates the password on a Veeam Backup & Replication "Credentials Record" across many backup servers in one pass, using the VBR REST API. Useful when the same service account is registered on several VBR servers and its password has to change everywhere at once.

Two variants are provided:

* [Python](#python-prompt): processes servers sequentially. Simple to read and adapt.
* [Go](#go-prompt-parallel): processes servers concurrently with goroutines and a configurable worker limit. Better for large estates.

| | |
| --- | --- |
| **Product** | Veeam Backup & Replication 13 |
| **Interface** | REST API 1.3-rev2 |
| **Output** | Python script, or Go binary (parallel) |
| **Tested with** | Claude |
| **Author** | [@shapedthought](https://github.com/shapedthought) |
| **Origin** | [Original gist](https://gist.github.com/shapedthought/c4a58d593c09c6fd0416b641cd9e9936) |

## Python prompt

```text
I need a Python script that rotates the password on a Veeam Backup & Replication (VBR) "Credentials Record" across multiple VBR backup servers, using the VBR REST API (v13, REST API revision 1.3-rev2).

Reference docs: https://helpcenter.veeam.com/references/vbr/13/rest/1.3-rev2/tag/Credentials

Requirements:

1. Auth: Authenticate against each server's `POST /api/oauth2/token` endpoint (password grant) to get a bearer token. Include the `x-api-version: 1.3-rev2` header on all requests.
2. Server list: Read a list of target VBR servers (host, port, admin username, admin password) from a JSON config file, so the script can run against many servers in one pass.
3. Find the record: On each server, look up the existing credentials record by username via `GET /api/v1/credentials` (use the `nameFilter` query param), and handle the case where it isn't found.
4. Change password: Change the password via `POST /api/v1/credentials/{id}/password`.
5. Security: Take the new password from an environment variable, not a CLI argument, so it doesn't end up in shell history.
6. TLS: Support an `--insecure` flag to skip certificate verification, for self-signed VBR server certs.
7. Resilience: Process all servers in the list even if one fails (log the error, continue), then print a summary of which servers succeeded/failed at the end, and exit with a non-zero code if any failed.
8. Logging: Use Python's `logging` module with timestamps, prefixed per-server, rather than bare `print()`.

Use the `requests` library. Structure it with a small class wrapping the per-server session/auth/calls, and a `main()` with `argparse` for the CLI. Include a docstring at the top explaining usage and the config file format.
```

## Go prompt (parallel)

```text
I need a Go program that rotates the password on a Veeam Backup & Replication (VBR) "Credentials Record" across multiple VBR backup servers in parallel, using the VBR REST API (v13, REST API revision 1.3-rev2).

Reference docs: https://helpcenter.veeam.com/references/vbr/13/rest/1.3-rev2/tag/Credentials

Requirements:

1. Auth: Authenticate against each server's `POST /api/oauth2/token` endpoint (password grant, form-encoded body) to get a bearer token. Include the `x-api-version: 1.3-rev2` header on all requests.
2. Server list: Read a list of target VBR servers (host, port, admin username, admin password) from a JSON config file passed via a `-config` flag, so the program can run against many servers in one pass.
3. Find the record: On each server, look up the existing credentials record by username via `GET /api/v1/credentials` (use the `nameFilter` query param). Check for an exact username match in the results rather than taking the first item, and treat "not found" as a failure for that server.
4. Change password: Change the password via `POST /api/v1/credentials/{id}/password`.
5. Security: Take the new password from an environment variable (e.g. `VBR_NEW_PASSWORD`), not a CLI flag, so it doesn't end up in shell history. Never log the new password, admin passwords, or bearer tokens.
6. TLS: Support an `-insecure` flag that sets `InsecureSkipVerify` on the TLS config, for self-signed VBR server certs.
7. Concurrency: Process servers in parallel using goroutines. Add a `-concurrency` flag (default 5) that caps how many servers are worked on at once, using a buffered channel semaphore or `errgroup.SetLimit`. Use a `sync.WaitGroup` (or errgroup) to wait for all workers, and collect per-server results over a channel into a results slice. Do not use shared mutable state without a mutex.
8. Timeouts: Give each server a `context.Context` with a timeout (flag `-timeout`, default 60s) so one unreachable server cannot hang the run. Pass the context to every HTTP request.
9. Resilience: A failure on one server must not stop the others. Capture the error per server, continue, then print a summary table of which servers succeeded/failed at the end, and exit with a non-zero code if any failed.
10. Logging: Use the standard `log/slog` package with a text handler, timestamps, and a `server` attribute on every line so interleaved output from goroutines is attributable.

Use only the Go standard library plus `golang.org/x/sync/errgroup` if you choose that route. Structure it as a single `main` package with:
- a `Server` struct for config entries,
- a `VBRClient` struct wrapping the base URL, `*http.Client`, and token, with methods `Login`, `FindCredentialID`, and `ChangePassword`,
- a `rotate(ctx, server) error` function that runs the three steps for one server,
- a `main()` that parses flags, loads config, fans out the work, and prints the summary.

Include a doc comment at the top explaining usage, the flags, and the config file format, and a `go.mod` with the module name and Go version.
```

## Notes

* The config file will contain admin credentials for every VBR server. Store it somewhere with restricted permissions and do not commit it. Consider asking the model to add support for pulling those from a secret store instead.
* If you are on a different VBR version, change the version and API revision in the first line and the reference URL. Endpoint paths and the `x-api-version` header value differ between revisions.
* Check that the generated code handles token expiry if you have a long server list, and that it never logs the new password or the bearer token.
* Check whether `nameFilter` matches on substring in your API revision, and make sure the code checks for an exact username match rather than taking the first result.
* For the Go variant, start with a low `-concurrency` value. Every worker opens its own auth session on a different VBR server, so the limit is mostly about how many parallel logins you are comfortable with, but very high values can trip rate limits or lockout policies.
* The Go variant should compile into a single static binary, which is handy for running from a jump host that has no Python installed.

## Disclaimer

See the [read this first](../README.md#%EF%B8%8F-read-this-first) section in the root README. This prompt is a starting point, not a finished solution. Check with your security team before using LLM-generated code that handles credentials, and review and test all generated output before running it against production infrastructure.
