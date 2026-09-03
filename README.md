# Awesome Veeam Prompts

A community collection of reusable LLM prompts for working with Veeam products. Think of it as the prompt equivalent of [VeeamHub/powershell](https://github.com/VeeamHub/powershell): each entry is a prompt you can paste into your LLM of choice (Claude, ChatGPT, Copilot, a local model, etc.) to generate scripts, configs, reports, or explanations for a specific Veeam task.

Prompts here are community driven and are **not** created by Veeam R&D or validated by Veeam QA. They are maintained by community members who may or may not be Veeam employees.

## ⚠️ Read this first

* A prompt is a starting point, not a finished solution. **You** are responsible for reviewing, testing, and securing anything an LLM produces from it.
* Check with your security team before using LLM-generated code that handles credentials or connects to production systems.
* Manually review and test all generated code before running it, particularly against production infrastructure.
* LLM output may contain errors or inaccuracies. Verify technical details independently against the [Veeam Help Center](https://helpcenter.veeam.com/).
* Everything here is provided under CC0 1.0, "AS IS", with no warranty of any kind and no liability accepted for any outcome arising from its use.

## 📗 How to use a prompt

1. Open the folder for the prompt you want. Each one has a `README.md` with the prompt, what it targets, and any notes.
2. Copy the prompt block and paste it into your LLM.
3. Adjust the requirements list to fit your environment (product version, API revision, language, auth method).
4. Review, test, and harden the output before you use it.

## 📚 Prompt index

Folders are prefixed by product, following the VeeamHub convention:

| Prefix | Product |
| ------ | ------- |
| `BR-` | Veeam Backup & Replication |
| `VB365-` | Veeam Backup for Microsoft 365 |
| `VONE-` | Veeam ONE |
| `VSPC-` | Veeam Service Provider Console |
| `VAW-` / `VAL-` | Veeam Agent for Windows / Linux |
| `VBAz-` / `VBAWS-` / `VBGCP-` | Veeam Backup for Azure / AWS / Google Cloud |
| `VDC-` | Veeam Data Cloud |
| `GEN-` | General / cross-product |

### Veeam Backup & Replication

| Prompt | Description | Output |
| ------ | ----------- | ------ |
| [BR-RotateCredentialPassword](BR-RotateCredentialPassword/) | Rotate the password on a VBR credentials record across multiple backup servers via the v13 REST API | Python script |

## ✍ Contributions

Contributions are very welcome. Open an [issue](../../issues/new/choose) to suggest or report a prompt, or submit a pull request to add or update one. See the [Contributing Guide](CONTRIBUTING.md) for the prompt format and folder conventions.

## 🤝🏾 License

* [CC0 1.0 Universal](LICENSE). All prompts in this repository are dedicated to the public domain.

## 🤔 Questions

If something is unclear, please [create an issue](../../issues/new/choose) and let us know.
