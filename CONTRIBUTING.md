# Contributing

Thanks for helping build this collection. Contributions can be:

* A new prompt
* An improvement to an existing prompt (better requirements, updated API revision, fixes)
* A bug report or suggestion via the [issue tracker](../../issues/new/choose)

## License

By contributing, you agree that your contribution is dedicated to the public domain under [CC0 1.0](LICENSE). Only submit prompts you wrote yourself or have the right to release this way.

## Adding a prompt

1. Create a folder named `<PREFIX>-<ShortName>` using PascalCase, for example `BR-RotateCredentialPassword` or `VB365-AuditRestoreOperations`. Product prefixes are listed in the [README](README.md#-prompt-index).
2. Add a `README.md` inside the folder using the template below.
3. Add a row to the relevant product table in the root `README.md`. Create the product section if it does not exist yet.
4. Open a pull request. The PR template will ask which LLM you tested the prompt with.

Optional extras in the folder:

* `example-output/` with a sanitised sample of what the prompt produced. Strip hostnames, credentials, and anything environment specific.
* Screenshots if the prompt produces something visual.

## Prompt README template

Copy the block below into your new folder's `README.md`.

````markdown
# <Title>

<One or two sentences on what the prompt does and why you would use it.>

| | |
| --- | --- |
| **Product** | Veeam Backup & Replication 13 |
| **Interface** | REST API 1.3-rev2 / PowerShell / UI |
| **Output** | Python script / PowerShell script / report / explanation |
| **Tested with** | Claude Opus 4.1, GPT-5 (list what you actually used) |
| **Author** | @your-github-handle |

## Prompt

```text
<The prompt. Write it as a numbered list of concrete requirements. Link the
relevant Help Center page so the model has the right reference.>
```

## Notes

* <Anything the user must change before running: version, API revision, paths.>
* <Known gaps in the output, or things the model tends to get wrong.>

## Disclaimer

See the [read this first](../README.md#%EF%B8%8F-read-this-first) section in the root README. This prompt is a starting point, not a finished solution. Review and test all generated output before use.
````

## Writing a good prompt

* **Be specific about versions.** Name the product version and the REST API revision or PowerShell module version. LLMs will happily invent endpoints that do not exist in your version.
* **Link the docs.** Point at the exact Help Center page for the API or cmdlet.
* **List requirements, not vibes.** Numbered, testable requirements produce far better output than "make it robust".
* **Call out security.** If the output handles credentials, say how they should be supplied (environment variable, secret store) and what must never be logged.
* **Ask for structure.** Specify the language, libraries, CLI shape, logging, and error handling you expect.

## Code of Conduct

Be respectful and constructive. Harassment, personal attacks, and publishing other people's private information are not acceptable. Maintainers may remove contributions or contributors that do not follow this.
