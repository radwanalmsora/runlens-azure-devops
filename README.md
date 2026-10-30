# RunLens for Azure Pipelines

RunLens adds a **RunLens** tab to every build result in Azure Pipelines. It explains why a build failed, which task was responsible, and whether the build became slower than recent builds of the same pipeline.

## What it shows

**Failure analysis**
- The first failed task in the build.
- The error messages recorded on that task, de-duplicated and capped.
- A likely category: environment (agent, disk, memory, network), dependency or package resolution, or code (compile or test failure).
- The rule that matched, so you can check the reasoning against the log.
- Suggested next steps for that category.

**Performance**
- Total wall-clock time of the build.
- The slowest tasks, with their share of the build time.
- A comparison with the median of up to five recent completed builds of the same pipeline.
- Tasks that became noticeably slower than their recent median.

## How it decides

RunLens uses transparent rules, not machine learning, and is not an AI product. Thresholds are constants in the source and are documented in [docs/DESIGN.md](docs/DESIGN.md):

- A build is flagged as slower only when it is at least 20% and at least 60 seconds above the median.
- A task is flagged only when it is at least 25% and at least 30 seconds above its median.
- Comparisons need at least two comparable builds.

## Permissions

RunLens requests a single scope: **Build (read)** (`vso.build`). It does not write to your pipelines, repositories, or work items.

## Privacy

RunLens runs inside Azure DevOps and calls the Azure DevOps REST API with the signed-in user's access token. It sends no data to any external service and collects no telemetry. See [SECURITY.md](SECURITY.md) to report a vulnerability.

## Requirements

- Azure DevOps Services (cloud). Azure DevOps Server is not supported yet.
- The user must have permission to view the build.

## Installation

1. Open the extension page on the Visual Studio Marketplace and choose **Get it free**.
2. Select your Azure DevOps organization and confirm.
3. Open any build in **Pipelines → Runs**, then select the **RunLens** tab.

Administrators can also install from **Organization settings → Extensions → Browse marketplace**.

## Documentation

- [Design notes and rule thresholds](https://github.com/radwanalmsora/runlens-azure-devops/blob/main/docs/DESIGN.md)
- [Security policy](https://github.com/radwanalmsora/runlens-azure-devops/blob/main/SECURITY.md)
- [Changelog](https://github.com/radwanalmsora/runlens-azure-devops/blob/main/CHANGELOG.md)
- [Contributing](https://github.com/radwanalmsora/runlens-azure-devops/blob/main/CONTRIBUTING.md)

## Support and maintenance

- **Questions and bugs:** open an issue at [github.com/radwanalmsora/runlens-azure-devops/issues](https://github.com/radwanalmsora/runlens-azure-devops/issues).
- **Security issues:** use private reporting, as described in [SECURITY.md](https://github.com/radwanalmsora/runlens-azure-devops/blob/main/SECURITY.md).
- **Maintainer:** RadwanAlmsora. The project is maintained by one person.
- **Commitment:** the extension is actively maintained. Issues receive a first response within 7 days, and security fixes follow the timelines in SECURITY.md. If the project is ever discontinued, the README will say so at least 90 days before the extension is delisted.

## Development

```bash
npm install
npm test          # compile and run the unit tests
npm run package   # build dist/ and create the .vsix
```

Project layout:

- `src/core/`: pure analysis rules, with no browser or API calls.
- `src/app.ts`: the tab UI, which loads build data through the Azure DevOps REST API.
- `src/test/`: unit tests for the rules.
- `vss-extension.json`: the extension manifest.
- `scripts/`: asset and packaging helpers.

## Limitations

- Classification is heuristic. Unusual error messages may fall back to "unknown", and the log remains the source of truth.
- Parallel jobs make "share of wall clock" percentages add up to more than 100%.
- Baselines use the last completed builds of the same pipeline definition. A change in pipeline structure can make the comparison less meaningful.

## License

[MIT](LICENSE) © RadwanAlmsora
