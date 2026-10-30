# Security Policy

## Supported versions

| Version | Supported |
| ------- | --------- |
| 1.x     | Yes       |

Only the latest published 1.x release receives security fixes.

## Reporting a vulnerability

Please do not open a public issue or pull request for a security problem.

Report it privately through **GitHub private vulnerability reporting**:

1. Open the repository's **Security** tab.
2. Choose **Report a vulnerability**.
3. Describe the issue and attach a reproduction if you have one.

Include:

- what the issue is and what an attacker could do with it,
- the steps to reproduce, ideally with a minimal build timeline or a screenshot,
- the RunLens version and the Azure DevOps area where you saw it.

## What to expect

- **Acknowledgement:** within 3 business days.
- **Status update:** within 10 business days, including whether the report is accepted.
- **Fix and release:** we aim to release a fix as soon as it is tested. We will agree a disclosure date with you when the fix is ready.
- **Credit:** reporters are credited in the changelog unless they ask not to be named.

RunLens is maintained by one person, so these are targets rather than guarantees. If you do not hear back within the stated time, you may post a reminder on the same advisory.

## Scope

In scope:

- Code in this repository, including the extension UI and the analysis rules.
- The extension manifest and the permissions it requests.
- The packaged `.vsix` built from this repository.

Out of scope:

- Azure DevOps itself. Report those issues to Microsoft through the Azure DevOps security process.
- Issues that need a user to already have the access they would gain.
- Findings from automated scanners without a demonstrated impact.

## Security design

RunLens is built to keep its attack surface small:

- It requests one scope only: **Build (read)** (`vso.build`). It does not write to pipelines, repositories, or work items.
- It sends data only to `dev.azure.com`, using the signed-in user's token from the Azure DevOps SDK. It does not send data to any other service, and it collects no telemetry.
- It does not store tokens. Tokens are used for the request and are not written to disk, logs, or the page.
- Build logs, task names, and error messages are rendered as plain text. They are never inserted as HTML.
- The page's Content Security Policy restricts network requests to `dev.azure.com`.
- The packaged extension contains the built `dist/` folder, the icon, the README, the license, and the standard manifest files. It contains no TypeScript sources, tests, or build configuration.
- CI installs dependencies with `npm ci`, which uses the committed lock file.
