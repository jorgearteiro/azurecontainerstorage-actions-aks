# Copilot instructions

## What this repository is

This is a **tutorial / reference deployment**, not a product. It shows how to run
GitHub Actions self-hosted runners on AKS using [ARC (Actions Runner Controller)](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners-with-actions-runner-controller/about-actions-runner-controller)
with two Azure storage layers:

- **Azure Container Storage** — replicated ephemeral NVMe volumes for the runner
  `_work` directory (fast, disposable per-job disk). Storage class
  `acstor-ephemeraldisk-nvme`.
- **Azure Files Premium (SMB)** — a `ReadWriteMany` persistent share mounted into
  every runner pod at `/home/runner/.nuget/` as a shared NuGet package cache.
  Storage class `azurefile-csi-premium`, share name `metadatacaching`.

The two "moving parts" that matter are therefore **`install/*.yaml`** (Helm values
+ Kubernetes manifests) and **`.github/workflows/*.yml`**. The `README.md` is the
authoritative end-to-end setup guide — read it before changing infra.

## The .NET app is only a sample workload

`Program.cs`, `Components/`, and `azurecontainerstorage-actions-aks.csproj` are the
default .NET 8 Blazor Server template (Home / Counter / Weather pages). The `.csproj`
lists ~50 unrelated NuGet packages **on purpose** — a comment in the file states they
exist "only to test Azure Files as caching layer." Their job is to make `dotnet restore`
heavy so the Azure Files NuGet cache is exercised. Do not treat this list as a real
dependency set, and don't "clean up" or refactor the Blazor app unless explicitly asked.

Build/run the sample locally with:

```bash
dotnet restore
dotnet build --configuration Release --no-restore
dotnet publish --configuration Release --no-restore --output ./publish
```

There are no unit tests despite `xunit`/`Moq` being referenced — they are part of the
package-bloat, not an actual test project.

## Workflows

All three workflows in `.github/workflows/` are `workflow_dispatch`-only and take an
`arc_runner_set_name` input (default `arc-runner-set-storage`) that feeds `runs-on:`.
They are meant to be triggered manually against the self-hosted runners:

- `dotnet-using-container.yml` — restore/build/publish inside a
  `mcr.microsoft.com/dotnet/sdk:8.0` job container (uses the workflow `container:` feature).
- `dotnet-without-container.yml` — installs the SDK on the runner itself via `setup-dotnet`.
- `container-service-test.yml` — exercises the `container:` + `services:` (redis) features.

When editing them, keep `runs-on: ${{ inputs.arc_runner_set_name }}` so the runner set
stays selectable at dispatch time.

## Conventions and gotchas

- **Names must stay in sync across files.** The Azure Files share name (`metadatacaching`),
  the namespace (`arc-runners-storage`), the resource group (`aks-storage-actions`), and
  the secret name (`azure-storage-secret`) are hardcoded in `install/*.yaml` and referenced
  by `README.md` commands. Change one, change all matching references.
- **`fsGroup: 123`** is required on runner pod specs — it is the GID of the GitHub
  runner image's default user. Preserve it when editing pod templates.
- **`install/arc-runners-permissions.yaml` is reference-only** — its header says not to
  apply it; ARC's service account creates the real Role/RoleBinding.
- **`containerMode.type: kubernetes`** is the chosen mode; the runner values file has a
  customized `template.spec` mounting both the NuGet Azure Files PVC and the
  `hook-extension` ConfigMap (`arc-runners-set-container-pod-spec.yaml`), which injects
  the same cache into job containers spawned by the `container:` feature.
- Helm chart versions are pinned (e.g. `0.9.3`) and the controller and runner-set
  **must use the same version**.
- `Components/App.razor` still references a legacy stylesheet name
  (`azurefiles-actions-aks.styles.css`); the generated file follows `AssemblyName`, so
  leave it unless renaming the assembly.

## Security note

`README.md` contains an example `-----BEGIN RSA PRIVATE KEY-----` block for the GitHub
App secret. It is a placeholder/dummy, not a live credential. Never commit real storage
keys, PATs, or GitHub App private keys — they are created at deploy time as Kubernetes
secrets (`azure-storage-secret`, `arc-runner-github-secret`).
