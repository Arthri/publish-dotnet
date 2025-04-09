# Publish .NET
Reusable workflows for publishing .NET projects to various targets.

> [!IMPORTANT]
> The workflows only support publishing one project per run. Attempting to publish multiple projects simultaneously may lead to corrupted releases.

## Installation
> [!NOTE]
> Additional installation steps may be required for some workflows.

All workflows require that a [global.json](https://learn.microsoft.com/en-us/dotnet/core/tools/global-json) file is present at the root of the repository.

To upload built binaries to GitHub releases when they are published, add a new workflow under `.github/workflows/` with the following contents,
```yml
name: Publish

on:
  release:
    types:
      - published

jobs:
  publish-release:
    uses: Arthri/publish-dotnet/.github/workflows/publish-release.yml@v2
    permissions:
      contents: write
```

To push a NuGet package when GitHub releases are published, add a new workflow under `.github/workflows/` with the following contents,
```yml
name: Publish

on:
  release:
    types:
      - published

jobs:
  publish-nuget:
    uses: Arthri/publish-dotnet/.github/workflows/publish-nuget.yml@v2
    secrets:
      NUGET_API_KEY: ${{ secrets.NUGET_API_KEY }}
```

To do both, combine the `jobs` key as follows,
```yml
name: Publish

on:
  release:
    types:
      - published

jobs:
  publish-release:
    uses: Arthri/publish-dotnet/.github/workflows/publish-release.yml@v2
    permissions:
      contents: write

  publish-nuget:
    uses: Arthri/publish-dotnet/.github/workflows/publish-nuget.yml@v2
    secrets:
      NUGET_API_KEY: ${{ secrets.NUGET_API_KEY }}
```

## Common Input Parameters
Parameters available on all or most workflows provided by the repository.

### Machine
The workflows, by default, run on Ubuntu 24.04 (as of April 09, 2025). The machine that the workflows run on can be changed by using the `runs-on` input parameter.

### Specify Project
The workflows, by default, builds the singular solution or project at the root of the repository. If there are multiple solutions and/or projects at the root of the repository, a solution or a project must be specified explicitly using the `project-path` input parameter.
```yml
jobs:
  publish-nuget:
    uses: Arthri/publish-dotnet/.github/workflows/publish-nuget.yml@v2
    with:
      project-path: ./src/Test.App/Test.App.csproj
```

### Override Version
By default, the project is built using the version defined in the project file. A different version can be set by using the `version` input parameter.
```yml
jobs:
  publish-nuget:
    uses: Arthri/publish-dotnet/.github/workflows/publish-nuget.yml@v2
    with:
      version: v1.0.0

      # Alternatively, use the tag associated with the GitHub release.
      version: ${{ github.event.release.tag_name }}
```

### Disable Version Sanitization
If a version is specified explicitly, then it will be sanitized to remove the `v` prefix and any directories. For example, `a/b/c/v1.0.0` becomes `1.0.0`, and `v1.2.3` becomes `1.2.3`. To disable version sanitization, set the `sanitize-version` input parameter to `false`.
```yml
jobs:
  publish-nuget:
    uses: Arthri/publish-dotnet/.github/workflows/publish-nuget.yml@v2
    with:
      sanitize-version: false
```

### Additional Build Arguments
```yml
jobs:
  publish-nuget:
    uses: Arthri/publish-dotnet/.github/workflows/publish-nuget.yml@v2
    with:
      build-arguments: -p:PublishAot=true
```

## GitHub Releases
Builds the solution or the project and uploads the resulting build artifacts inside an archive. For solutions, each project's build artifacts are uploaded in a separate archive.

### Filter Projects to be Uploaded
If no project is specified, then all projects' outputs are uploaded. To upload only select projects, specify a pattern using the `project-filter` input parameter. See https://www.gnu.org/software/bash/manual/html_node/Pattern-Matching.html for details on patterns.
```yml
jobs:
  publish-release:
    uses: Arthri/publish-dotnet/.github/workflows/publish-release.yml@v2
    permissions:
      contents: write
    with:
      # Upload all projects.
      project-filter: *

      # Upload all projects starting with "Belp."
      project-filter: Belp.*

      # Only upload the "Belp.Build.Packinf" and the "Belp.Build.Testing" projects.
      # "@" is a reserved character in YAML, so the pattern must appear in quotes.
      project-filter: '@(Belp.Build.Packinf|Belp.Build.Testing)'

      # Upload all projects starting with "Belp.Build." or "Belp.SDK."
      project-filter: '@(Belp.Build.*|Belp.SDK.*)'
```

### Configure Archive Names
The default name of the archive uploaded to GitHub Releases is `release`, and the extension is `.tar.gz`. The name of the archive can be configured using the `filename-template` input parameter. The specified name is passed through [envsubst](https://www.gnu.org/software/gettext/manual/html_node/envsubst-Invocation.html); consequently, it may utilize environment variables available to the workflow. Furthermore, three additional environment variables, `PROJECT_NAME`, `PROJECT_VERSION`, `PROJECT_INFORMATIONAL_VERSION`, are provided and usable inside the template.
```yml
jobs:
  publish-release:
    uses: Arthri/publish-dotnet/.github/workflows/publish-release.yml@v2
    permissions:
      contents: write
    with:
      # Sets the archive's name to archive.tar.gz.
      filename-template: archive

      # Sets the archive's name to the project's name.
      # If, for example, the project's name is Belp.Build.Packinf,
      # then the resulting archive's name will be Belp.Build.Packinf.tar.gz.
      filename-template: $PROJECT_NAME

      # Sets the archive's name to the project's name, followed by a hyphen,
      # then by the project's information version.
      # If, for example, the project's name is Belp.Build.Packinf and the
      # project's informational version is 0.0.1-alpha, then the resulting
      # archive's name will be Belp.Build.Packinf-0.0.1-alpha.tar.gz.
      filename-template: $PROJECT_NAME-$PROJECT_INFORMATIONAL_VERSION
```

### Specify Release Tag
The workflow assumes that it is triggered by a release event. If that is not the case, then a release tag must be explicitly specified through the `release-tag` input parameter.

## NuGet

### Installation
The workflow requires an environment named `NuGet (Stable)` with a secret named `NUGET_API_KEY` containing the API key used to publish packages to NuGet. The following is a list of steps to create the environment.

> [!NOTE]
> Environments are not available on private repositories (as of April 09, 2025) under the free plan. https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-deployments/managing-environments-for-deployment

1. Go to the repository's settings tab.
1. Navigate to `Environments` under the `Code and automation` section.
1. Create a new environment named `NuGet (Stable)`.
1. Obtain a NuGet API key.
1. Add a secret named `NUGET_API_KEY` containing the NuGet API key.
1. Configure environment as appropriate.

### Set Release Notes or Changelog
NuGet supports release notes (for an example, see [Belp.Build.Packinf@0.6.0](https://www.nuget.org/packages/Belp.Build.Packinf/0.6.0#releasenotes-body-tab)), a tab dedicated to the changes introduced in a given version. By default, the workflow doesn't build packages with release notes, but it can be configured to.
```yml
jobs:
  publish-nuget:
    uses: Arthri/publish-dotnet/.github/workflows/publish-nuget.yml@v2
    with:
      changelog: |
        - Added this
        - Removed that
```

Optionally, to source the release notes from the GitHub release,
```yml
jobs:
  publish-nuget:
    uses: Arthri/publish-dotnet/.github/workflows/publish-nuget.yml@v2
    with:
      changelog: ${{ github.event.release.body }}
```

### Change Environment Name
The workflow acquires the NuGet API key from an environment named `NuGet (Stable)`, by default. The environment name can be changed using the `environment-name` input parameter.
```yml
jobs:
  publish-nuget:
    uses: Arthri/publish-dotnet/.github/workflows/publish-nuget.yml@v2
    with:
      environment-name: Custom Environment Name
```
