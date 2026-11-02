---
name: scaffold-test-project
description: Scaffold a new .NET 9 NUnit + Playwright API test automation project, pre-loaded with reusable authentication infrastructure (Keycloak client-credentials token fetch), the HTML/JSON/console TestReporter, and a single AuthenticationTests smoke test — everything else (Models, Payloads, other Tests) left empty and ready for a new API. Use when asked to create/scaffold/bootstrap a new test automation project, spin up a new API test repo from the reusable template, or start a new Playwright NUnit project that reuses the existing auth+reporting framework.
---

# Scaffold a new test-automation project from the reusable template

This skill creates a brand-new .NET 9 NUnit + Playwright API test project. It reuses,
byte-for-byte (only the `__PROJECT_NAME__` placeholder is substituted — this covers the
namespace as well as the HTML report's `<title>` and header text), the authentication
infrastructure and test reporter from the reference project (originally `PaypalTests`).
Everything else
(`Models/`, `Payloads/`, other test classes) is intentionally left empty scaffolding for
the new project to fill in.

All the reusable source files live as `.template` files under this skill's `assets/`
directory, plus `assets/mukuru-card.jpg` (the HTML report's header background image).
Do not regenerate these files from memory — copy them from `assets/` and substitute the
placeholder, so the copies stay byte-identical to the source of truth.

## 1. Get the inputs

Determine from `args` (or ask the user if not given):
- **Project name** (PascalCase, e.g. `MyServiceTests`) — becomes the `.csproj`/`.sln`
  name and the C# namespace for every reused file.
- **Target directory** — where to create the project (default: a new folder named after
  the project, in the current working directory).

## 2. Scaffold the .NET project

```
dotnet new nunit -n {ProjectName}
cd {ProjectName}
dotnet new sln
dotnet sln add {ProjectName}.csproj

dotnet add package coverlet.collector --version 6.0.2
dotnet add package dotenv.net --version 3.2.1
dotnet add package JunitXml.TestLogger --version 8.0.0
dotnet add package Microsoft.Playwright --version 1.52.0
dotnet add package NUnit.Analyzers --version 4.4.0
```

Delete the default sample test file `dotnet new nunit` generated (`UnitTest1.cs` /
`Tests.cs`).

Edit `{ProjectName}.csproj`: set `<PropertyGroup>` to

```xml
<TargetFramework>net9.0</TargetFramework>
<LangVersion>latest</LangVersion>
<ImplicitUsings>enable</ImplicitUsings>
<Nullable>enable</Nullable>
<IsPackable>false</IsPackable>
```

and add:

```xml
<ItemGroup>
  <Using Include="NUnit.Framework" />
</ItemGroup>

<ItemGroup>
  <Folder Include="TestReports\" />
</ItemGroup>
```

## 3. Create the folder structure

```
{ProjectName}/
  Infrastructure/
  Models/            (empty — add .gitkeep)
  Payloads/          (empty — add .gitkeep)
  Tests/             (only AuthenticationTests.cs)
  TestReports/       (empty — add .gitkeep; reports generate here at runtime)
```

`TestResults/` (dotnet test/coverage output) is not created up front — it appears on
first test run and must never be committed (see `.gitignore` below).

## 4. Copy the reusable files, substituting the namespace

For every `*.template` file below, copy it to the destination path (stripping the
`.template` suffix) and replace every occurrence of `__PROJECT_NAME__` with the actual
project name:

| Source (this skill's `assets/`) | Destination in new project |
|---|---|
| `JsonDefaults.cs.template` | `JsonDefaults.cs` |
| `Endpoints.cs.template` | `Endpoints.cs` |
| `TestReporter.cs.template` | `TestReporter.cs` |
| `Infrastructure/BaseTestFixture.cs.template` | `Infrastructure/BaseTestFixture.cs` |
| `Infrastructure/GlobalTestSetup.cs.template` | `Infrastructure/GlobalTestSetup.cs` |
| `Infrastructure/PaymentTestHelpers.cs.template` | `Infrastructure/PaymentTestHelpers.cs` |
| `Infrastructure/StringToDecimalConverter.cs.template` | `Infrastructure/StringToDecimalConverter.cs` |
| `Tests/AuthenticationTests.cs.template` | `Tests/AuthenticationTests.cs` |

Do this with `sed`, e.g.:

```
sed 's/__PROJECT_NAME__/{ProjectName}/g' <skill-dir>/assets/JsonDefaults.cs.template > JsonDefaults.cs
```

`Endpoints.cs` ends up empty (just the namespace + class skeleton) — that's intentional,
it's populated per-project as new endpoints are tested.

`JsonDefaults.cs` is a required dependency of `BaseTestFixture.cs` and
`PaymentTestHelpers.cs` (both reference `JsonDefaults.Options`) even though it isn't one
of the four files under `Infrastructure/` — don't skip it.

## 5. Copy non-template assets

```
cp <skill-dir>/assets/mukuru-card.jpg ./mukuru-card.jpg
cp <skill-dir>/assets/.env.example ./.env.example
cp <skill-dir>/assets/gitignore.template ./.gitignore
```

`mukuru-card.jpg` is referenced by `TestReporter.cs`'s HTML report as
`background-image: url('../mukuru-card.jpg')` — keep it at the project root, one level
above `TestReports/`, and keep the filename unchanged.

`.env.example` is structure-only — placeholder values, no live secrets. Tell the user to
copy it to `.env` and fill in real credentials themselves; never invent or embed real
secret values.

## 6. Verify

```
dotnet build
dotnet test --filter "AuthenticationTests"
```

The build must succeed. Running the filtered test will fail at runtime without a real
`.env` (expected — token endpoints aren't reachable) — that's fine for a structural
verification; don't try to fabricate credentials to make it pass.

Confirm `TestReports/` received a fresh JSON + HTML report after the run, then delete
those generated reports before committing (the folder itself stays, tracked via
`.gitkeep`).

## 7. Report back to the user

Summarize what was created, and explicitly flag:
- Where `.env.example` is and that they need to fill in `.env` manually with real
  credentials for their new backend.
- That `Models/`, `Payloads/`, `Endpoints.cs`, and `Tests/` (besides
  `AuthenticationTests.cs`) are intentionally empty for them to build out.
