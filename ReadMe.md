# The Module Builder Project

This project is an attempt by a group of PowerShell MVPs and module authors to:

1. Build a common set of [tools for module authoring](#whats-in-the-module-so-far)
2. Encourage a common pattern for [organizing PowerShell module projects](#organizing-your-module)
3. Promote best practices for authoring functions and modules

In short, we want to make it easier for people to write great code and produce great modules.

In service of this goal, we intend to produce:

1. Guidance on using the best of the existing tools: dotnet, Pester, PSDepends, etc.
2. Module templates demonstrating best practices for organization
3. Function templates demonstrating best practices for common parameters and error handling
4. ModuleBuilder module - a set of tools for building modules following these best practices

## The ModuleBuilder module

This module is the first concrete step in the project (although it currently consists of only two commands). It represents the collaboration of several MVPs and module authors who had each written their own version of these tools for themselves, and have now decided to collaborate on creating a shared toolset. We are each using the patterns and tools that are represented here, and are committed to helping others to succeed at doing so.

### Building from source

[![Build Status](https://poshcode.visualstudio.com/ModuleBuilder/_apis/build/status/ModuleBuilder)](https://poshcode.visualstudio.com/ModuleBuilder/_build/latest?definitionId=1)

#### 1. Get the source, obviously

```powershell
git clone https://github.com/PoshCode/ModuleBuilder.git
cd Modulebuilder
```

#### 2. Install dependencies

We have a few modules which are required for building. They're listed in `RequiredModules.psd1` -- the `Install-RequiredModule` script installs them (it defaults to CurrentUser scope, but has a `-Scope` parameter if you're running elevated and want to install them for the `AllUsers`). They only change rarely, so you shouldn't need to run this repeatedly, but it does _import_ the modules too, so if you need to use newer versions of these modules (like Pester 5), you can use this to import the ones we need:

```powershell
./Install-RequiredModule.ps1
```

#### 3. Run the `build.ps1` script.

By default, the build script uses [gitversion](/gittols/gitversion) to calculate the version of the build automatically, and will put the build in a folder like "ModuleBuilder\2.0.0" where 2.0.0 is the current version number.

```powershell
./build.ps1
```

If you don't have `gitversion` handy, you can just specify a version for the `-Semver` parameter:

```powershell
./build.ps1 -Semver 2.1.0-beta
```

#### 4. Run tests with Pester

The `test.ps1` script runs Pester and ScriptAnalyzer. It finds the build output from our `build.ps1` script in the default build output location -- that is, right next to these scripts in the root of the repository. It actually removes the `ModuleBuilder` module and re-imports the highest version in that root:

```powershell
./test.ps1
```

If you want to test against a different version, you can import it manually and `Invoke-Pester` yourself.

##### You have a lot of other options here ...

You could import the module explicitly from the output path:

```powershell
./build.ps1 | Split-Path | Import-Module -Force
```

You could build into your personal Modules directory, instead:

```powershell
./build -Output ($Profile | Split-Path | Join-Path -ChildPath Modules)
```
### What's in the module, so far:

#### `Build-Module`

Builds a script module from a source project containing one file per function in `Public` and `Private` folders.

The `Build-Module` command is a build task for PowerShell script modules that supports [incremental builds](https://docs.microsoft.com/en-us/visualstudio/msbuild/incremental-builds).

#### `Convert-CodeCoverage`

Takes the output from `Invoke-Pester -Passthru` run against the build output, and converts the code coverage report to the source lines.

## A note on build tools

There are several PowerShell build frameworks available. The build task in ModuleBuilder doesn't particularly endorse or interoperate with any of them, but it does accomplish a particular task that is needed by all of them.

A good build framework needs to support [incremental builds](https://docs.microsoft.com/en-us/visualstudio/msbuild/incremental-builds) and have a way to define build targets which have dependencies on other targets, such that it can infer the [target build order](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-targets#target-build-order).

A good build framework should also include pre-defined tasks for most common build targets, including restoring dependencies, cleaning old output, building and assembling a module from source, testing that module, and publishing the module for public consumption.  Our `Build-Module` command, for instance, is just one task of several which would be needed for a build target for a PowerShell script module.


## Organizing Your Module

For best results, you need to organize your module project similarly to how this project is organized. It doesn't have to be exact, because nearly all of our conventions can be overriden, but the module *is* opinionated, so if you follow the conventions, it should feel wonderfully automatic.

1. Create a `source` folder with a `build.psd1` file and your module manifest in it
2. In the `build.psd1` specify the relative **Path** to your module's manifest, e.g. `@{ Path = "ModuleBuilder.psd1" }`
3. In your manifest, make sure a few values are not commented out. You can leave them empty, because they'll be overwritten:
    - `FunctionsToExport` will be updated with the _file names_ that match the `PublicFilter`
    - `AliasesToExport` will be updated with the values from `[Alias()]` attributes on commands
    - `Prerelease` and `ReleaseNotes` in the `PSData` hashtable in `PrivateData`

Once you start working on the module, you'll create sub-folders in source, and put script files in them with only **one** function in each file. You should name the files with _the same name_ as the function that's in them -- especially in the public folder, where we use the file name (without the extension) to determine the exported functions.

1. By convention, use folders named "Classes" (and/or "Enum"), "Private", and "Public"
2. By convention, the functions in "Public" will be exported from the module (you can override the `PublicFilter`)
3. To force classes to be in a certain order, you can prefix their file names with numbers, like `01-User.ps1`

There are a *lot* of conventions in `Build-Module`, expressed as default values for its parameters. These defaults are documented in the help for Build-Module. You can override any parameter to `Build-Module` by passing it, or by adding keys to the `build.psd1` file with your preferences.

## Changelog

### 3.0.0 - Now with better alias support

Starting with this release, ModuleBuilder will automatically export aliases from `New-Alias` and `Set-Alias` as well as the `[Alias()]` attributes on commands. This is (probably not) a breaking change, but because it can change the aliases exported by existing modules that use ModuleBuilder, I've bumped the major version number as a precaution (if you're reading this, mission accomplished).

Additionally, the `Build-Module` command now _explicitly sorts_ the source files into alphabetical order, to ensure consistent behavior regardless of the native order of the underlying file system. This is technically also a breaking change, but it's unlikely to affect anyone except the people whose builds didn't work on non-Windows systems because of the previous behavior.

