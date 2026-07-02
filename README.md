# TLM Plugins SDK — Distribution

Public distribution repository for the **DigiCert Trust Lifecycle Manager (TLM) Plugins SDK**.

This repository is the **package host**, not the source. It publishes the SDK's Maven
artifacts (JARs and POMs) so that you can add the SDK as a dependency and build your own
TLM plugins. Use it to pull a released SDK version into your plugin project, then package
and upload the result to Trust Lifecycle Manager.

- **You are here:** the SDK distribution (Maven packages).
- **Where you build:** your own plugin project, typically started from an
  [official example](#examples).
- **Where you deploy:** Trust Lifecycle Manager → **Integrations → Plugins**.

> For product concepts and the full plugin walkthrough, see the official docs:
> [Build your plugin](https://docs.digicert.com/en/trust-lifecycle-manager/build-your-inventory-and-ecosystem/plugins/build-your-plugin.html).

---

## Contents

- [How it fits together](#how-it-fits-together)
- [Supported plugin workflows](#supported-plugin-workflows)
- [Prerequisites](#prerequisites)
- [Consume the SDK](#consume-the-sdk)
  - [1. Authenticate to GitHub Packages](#1-authenticate-to-github-packages)
  - [2. Configure `settings.xml`](#2-configure-settingsxml)
  - [3. Declare the repository and dependency](#3-declare-the-repository-and-dependency)
- [Build and package a plugin](#build-and-package-a-plugin)
- [Deploy to Trust Lifecycle Manager](#deploy-to-trust-lifecycle-manager)
- [Examples](#examples)
- [Versioning](#versioning)
- [Support](#support)
- [License](#license)

---

## How it fits together

The SDK is published here once and consumed by every plugin project. Each plugin
implements one workflow type, builds to a distributable ZIP, and is uploaded to TLM.

```mermaid
flowchart TD
    SDK["tlm-plugins-sdk-dist<br/><i>(this repo — SDK Maven packages)</i>"]

    subgraph Examples["Plugin projects (start from tlm-plugin-examples)"]
        CD["Certificate delivery plugin<br/><code>AbstractAutomationWorkflow</code>"]
        AU["Automation plugin<br/><code>AbstractAutomationWorkflow</code>"]
        DI["Discovery plugin<br/><code>AbstractDiscoveryWorkflow</code>"]
    end

    ZIP["plugin-dist/*.zip<br/><i>(JAR + metadata JSON + SHA-256)</i>"]
    TLM["Trust Lifecycle Manager<br/>Integrations → Plugins"]

    SDK -->|Maven dependency| CD
    SDK -->|Maven dependency| AU
    SDK -->|Maven dependency| DI

    CD --> ZIP
    AU --> ZIP
    DI --> ZIP
    ZIP -->|upload + verify| TLM
```

**Flow:** add the SDK dependency → implement your workflow logic → `./build.sh` produces a
signed ZIP in `plugin-dist/` → upload the ZIP + configuration JSON to TLM → verify and activate.

---

## Supported plugin workflows

A plugin implements exactly one of the three workflow types that Trust Lifecycle Manager
recognizes. Each maps to an SDK base class you extend and a small set of methods you override.

| Workflow | What it does | SDK base class | Typical methods to implement |
| --- | --- | --- | --- |
| **Certificate delivery** | Request and deliver certificates to a target system type. | `AbstractAutomationWorkflow` | `testConnection`, `generateCsr`, `installCertificate`, `validateCertificate` |
| **Automation** | Manage and automate certificate lifecycles (renewal, rotation, inventory) on network appliances and cloud services. | `AbstractAutomationWorkflow` | `testConnection`, `generateCsr`, `installCertificate`, `validateCertificate`, `refreshConfiguration` |
| **Discovery** | Import certificate/endpoint inventory from a third-party scan provider. | `AbstractDiscoveryWorkflow` | `testConnection`, `getDiscoveryData`, `getUserDetails` |

Configuration properties (credentials, URLs, network settings) are declared in your
plugin's configuration class and mirrored in the configuration JSON you upload to TLM.

References:
[Automation plugins](https://docs.digicert.com/en/trust-lifecycle-manager/build-your-inventory-and-ecosystem/plugins/build-your-plugin/build-automation-plugins.html) ·
[Discovery plugins](https://docs.digicert.com/en/trust-lifecycle-manager/build-your-inventory-and-ecosystem/plugins/build-your-plugin/build-discovery-plugins.html) ·
[Add a plugin](https://docs.digicert.com/en/trust-lifecycle-manager/build-your-inventory-and-ecosystem/plugins/add-a-plugin.html)

---

## Prerequisites

- **Java** 11 or later (JDK) and **Apache Maven** 3.6+.
- A **GitHub account** with a **personal access token (classic)** that has the
  `read:packages` scope — required to download packages from GitHub Packages.
- Access to a **Trust Lifecycle Manager** account to upload and activate the finished plugin.

---

## Consume the SDK

The SDK is distributed via **GitHub Packages** (Apache Maven registry). Consuming it takes
three steps: authenticate, point Maven at the registry, and declare the dependency.

### 1. Authenticate to GitHub Packages

GitHub Packages requires authentication even for public packages. Create a
[personal access token (classic)](https://github.com/settings/tokens) with the
`read:packages` scope and keep it handy for the next step.

> Never commit your token. Prefer an environment variable and reference it from
> `settings.xml`, or store the token in an encrypted CI secret.

### 2. Configure `settings.xml`

Add a matching `<server>` entry to your Maven `settings.xml` (usually `~/.m2/settings.xml`).
The `<id>` must match the repository `<id>` you declare in the next step.

```xml
<settings>
  <servers>
    <server>
      <id>github-tlm-sdk</id>
      <username>YOUR_GITHUB_USERNAME</username>
      <password>${env.GITHUB_TOKEN}</password>
    </server>
  </servers>
</settings>
```

Set the token in your environment before building:

```bash
export GITHUB_TOKEN=ghp_your_read_packages_token   # macOS/Linux
```

```powershell
$env:GITHUB_TOKEN = "ghp_your_read_packages_token"  # Windows PowerShell
```

### 3. Declare the repository and dependency

In your plugin project's `pom.xml`, add the GitHub Packages repository and the SDK dependency.

```xml
<repositories>
  <repository>
    <id>github-tlm-sdk</id>
    <url>https://maven.pkg.github.com/digicert/tlm-plugins-sdk-dist</url>
  </repository>
</repositories>

<dependencies>
  <dependency>
    <groupId>com.digicert.tlm.plugins</groupId>
    <artifactId>tlm-plugins-sdk</artifactId>
    <version>REPLACE_WITH_LATEST</version>
  </dependency>
</dependencies>
```

> **Authoritative coordinates:** the exact `groupId`, `artifactId`, and available
> `version` values are published on this repository's **[Packages](https://github.com/digicert/tlm-plugins-sdk-dist/packages)**
> tab. Always pin to a specific released version rather than a snapshot. The example
> projects already reference the correct coordinates — the fastest path is to start from one.

---

## Build and package a plugin

The example projects are Maven projects that bundle a build script. From the project root:

```bash
./build.sh
```

The build compiles your plugin, resolves the SDK from GitHub Packages, and produces a
distributable archive in the **`plugin-dist/`** subdirectory. The archive contains:

- the plugin **JAR**,
- the **metadata JSON** required by Trust Lifecycle Manager, and
- a **SHA-256** checksum for integrity verification.

Keep the plugin ZIP under **100 MB** — that is the upload limit enforced by TLM.

---

## Deploy to Trust Lifecycle Manager

1. In Trust Lifecycle Manager, go to **Integrations → Plugins** and select **Add plugin**.
2. Upload the **plugin ZIP** (from `plugin-dist/`) and its **configuration JSON**.
3. Choose the **workflow type** — Certificate delivery, Automation, or Discovery — and the
   target OS (Linux, Windows, or Cross platform). Provide name, vendor, version (`x.x.x`),
   and an optional description.
4. Select **Verify and create**. TLM scans the package for malicious content; verification
   can take up to ~10 minutes before the plugin becomes **Active** and usable.

Full steps: [Add a plugin in Trust Lifecycle Manager](https://docs.digicert.com/en/trust-lifecycle-manager/build-your-inventory-and-ecosystem/plugins/add-a-plugin.html).

---

## Examples

Working, buildable starting points for each workflow type are published in the consolidated
examples repository:

**➡️ [digicert/tlm-plugin-examples](https://github.com/digicert/tlm-plugin-examples)**

It contains one example per workflow — certificate delivery, automation, and discovery.
Clone the example that matches your use case, adapt the workflow class and configuration,
then build and upload as described above.

---

## Versioning

- SDK artifacts follow **semantic versioning** (`MAJOR.MINOR.PATCH`).
- Released versions are listed on the **[Packages](https://github.com/digicert/tlm-plugins-sdk-dist/packages)**
  tab; always depend on a fixed released version.
- Pin the SDK version in your `pom.xml` so plugin builds stay reproducible.

---

## Support

For access, integration guidance, or help scoping a custom plugin, contact your **DigiCert
account representative or solutions engineer**, or open an issue in the relevant public
repository. Product documentation:
[Trust Lifecycle Manager plugins](https://docs.digicert.com/en/trust-lifecycle-manager/build-your-inventory-and-ecosystem/plugins.html).

---

## License

Licensed under the [Apache License 2.0](./LICENSE).
