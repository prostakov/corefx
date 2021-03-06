# How to service a library in CoreFx

This document provides the steps necessary after modifying a CoreFx library in a servicing branch (where "servicing branch" refers to any branch whose name begins with `release/`).

## Check for existence of a .pkgproj

Most CoreFx libraries are not packaged by default. Some libraries have their output packaged in `Microsoft.Private.CoreFx.NetCoreApp`, which is always built, while other libraries have their own specific packages, which are only built on-demand. Your first step is to determine whether or not your library has its own package. To do this, go into the root folder for the library you've made changes to. If there is a `pkg` folder there (which should have a `.pkgproj` file inside of it), your library does have its own package. If there is no `pkg` folder there, the library should be built as part of `Microsoft.Private.CoreFx.NetCoreApp` and shipped as part of `Microsoft.NetCore.App`. To confirm this, check for the `IsNETCoreApp` property being set to `true` in the library's `Directory.Build.props`. If it is, then there is nothing that needs to be done. If it's not, contact a member of the servicing team for guidance, as this situation goes against our convention.

For example, if you made changes to [System.Data.SqlClient](https://github.com/dotnet/corefx/tree/5da5fd3628253da1d2578ab5c6a437202ac76254/src/System.Data.SqlClient), then you have a .pkgproj, and will have to follow the steps in this document. However, if you made changes to [System.Collections](https://github.com/dotnet/corefx/tree/5da5fd3628253da1d2578ab5c6a437202ac76254/src/System.Collections), then you don't have a .pkgproj, and you do not need to do any further work for servicing.

## Determine PackageVersion

Each package has a property called `PackageVersion`. When you make a change to a library & ship it, the `PackageVersion` must be bumped. This property could either be in one of two places inside your library's source folder: `Directory.Build.Props` (called `dir.props` in 2.0.0 & earlier), or the `.pkgproj`. It's also possible that the property is in neither of those files, in which case we're using the default version from [Packaging.props](https://github.com/dotnet/corefx/blob/a10890f4ffe0fadf090c922578ba0e606ebdd16c/Packaging.props#L36) (IMPORTANT - make sure to check the default version from the branch that you're making changes to, not from master). You'll need to increment this package version. If the property is already present in your library's source folder, just increment the patch version by 1 (e.g. `4.6.0` -> `4.6.1`). If it's not present, add it to the  library's `Directory.Build.props`, where it is equal to the version in `Packaging.props`, with the patch version incremented by 1. 

Note that it's possible that somebody else has already incremented the package version since our last release. If this is the case, you don't need to increment it yourself. To confirm, check [Nuget.org](https://www.nuget.org/) to see if there is already a published version of your package with the same package version. If so, the `PackageVersion` must be incremented. If not, there's still a possibility that the version should be bumped, in the event that we've already built the package with its current version in an official build, but haven't released it publicly yet. If you see that the `PackageVersion` has been changed in the last 1-2 months, but don't see a matching package in Nuget.org, contact somebody on the servicing team for guidance.

## Determine AssemblyVersion

Much like `PackageVersion`, each library package has a property called `AssemblyVersion`. This will be in either the `.csproj` or in `Directory.Build.Props`/`dir.props`. If your library's `.csproj` (inside the library's `src` directory) references `.NETStandard` or `.NETFramework`, the `AssemblyVersion` should have its patch version incremented by 1 (e.g. `4.0.0.0` -> `4.0.1.0`). If we don't update the AssemblyVersion, a user could run into a situation where two different dependencies of their application pull a version of your library with the same `AssemblyVersion` (`4.0.0.0`), but different `PackageVersion`. If the older package is loaded first, then the application will be using the older version of your library.

## Add your package to packages.builds

In order to ensure that your package gets built, you need to add it to [packages.builds](https://github.com/dotnet/corefx/blob/588b431cf43cf9433e3c6c3e4367cbed8ac2dfa8/src/packages.builds#L27-L29). In the linked example, we were building `System.Drawing.Common`. All you have to do is add a `Project` block inside the linked ItemGroup that matches the form of the linked example, but with `System.Drawing.Common` replaced by your library's name. Again, make sure to do this in the right servicing branch.

## Test your changes

All that's left is to ensure that your changes have worked as expected. To do so, execute the following steps:

1. From a clean copy of your branch, run `build.cmd -allconfigurations`

2. Check in `bin\packages\Debug` for the existence of your package, with the appropriate package version.

3. Try installing the built package in a test application, testing that your changes to the library are present & working as expected.