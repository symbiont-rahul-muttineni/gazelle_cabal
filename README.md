## A gazelle extension for Cabal files

[![Build status](https://badge.buildkite.com/4529c8b05b4bb481bd50b509acdfb3e5f862b308c8ddbe441d.svg?branch=main)](https://buildkite.com/tweag-1/gazelle-cabal)

This is a [gazelle][gazelle] extension that generates and
updates [Haskell rules][rules_haskell] for [Bazel][bazel]
from [Cabal][cabal] files.

For each Cabal file found, rules are generated in a sibling
`BUILD` file for every `buildable` component.

Additionally, it generates a `stack_snapshot` rule in the
`WORKSPACE` file with all of the necessary dependencies.

This [example repo][example] shows it in action.

## Configuration

Firstly, setup [gazelle][gazelle] and [rules_haskell][rules_haskell].
Then import `gazelle_cabal`.

```python
http_archive(
    name = "io_tweag_cabal_gazelle",
    strip_prefix = "gazelle_cabal-main",
    url = "https://github.com/tweag/gazelle_cabal/archive/main.zip",
)
```

Additionally, some Haskell packages are needed to build
`gazelle_cabal`. The simplest way to bring them is to use the
`stack_snapshot` rule in the `WORKSPACE` file as follows.

```python
load("@rules_haskell//haskell:cabal.bzl", "stack_snapshot")
load("@io_tweag_gazelle_cabal//:defs.bzl", "gazelle_cabal_dependencies")
gazelle_cabal_dependencies()

stack_snapshot(
    name = "stackage",
    packages = [
        "aeson", # keep
        "optparse-applicative", # keep
        "path", # keep
        "path-io", # keep
    ],
    # Most snapshots of your choice might do
    snapshot = "lts-18.1",
)
```
Should Haskell packages need to be grabbed from elsewhere, alternative
labels can be provided to [gazelle_cabal_dependencies][gazelle_cabal_dependencies].

You can generate or update build rules by adding the following to
one of your `BUILD` files.

```python
load(
    "@bazel_gazelle//:def.bzl",
    "DEFAULT_LANGUAGES",
    "gazelle",
    "gazelle_binary",
)

gazelle(
    name = "gazelle",
    data = ["@io_tweag_gazelle_cabal//cabalscan"],
    gazelle = "//:gazelle_binary",
)

gazelle_binary(
    name = "gazelle_binary",
    languages = DEFAULT_LANGUAGES + ["@io_tweag_gazelle_cabal//gazelle_cabal"],
)
```

## Running

Build and run gazelle with
```bash
bazel run //:gazelle
```

Gazelle's [fix command][fix-command] can be used to delete rules when
components are removed from the cabal file.

In order to generate or update the `stack_snapshot` rule, add
the following to one of your build files

```python
gazelle(
    name = "gazelle-update-repos",
    command = "update-repos",
    data = ["@io_tweag_gazelle_cabal//cabalscan"],
    extra_args = [
        "-lang",
        "gazelle_cabal",
        "stackage",
    ],
    gazelle = "//:gazelle_binary",
)
```
Then build and run gazelle with
```bash
bazel run //:gazelle-update-repos
```
`gazelle_cabal` only fills the `packages` attribute of `stack_snapshot`.
Either before or after the update, the other mandatory attributes of the
rule need to be written by the user.

## Directives

```python
# gazelle:cabal_extra_libraries sodium=@libsodium//:libsodium
```
Maps names in the Cabal file's `extra-libraries` field to Bazel
labels. The labels are added to the `deps` attribute of the
corresponding rule. Names not mentioned in this directive are
added to the `compiler_flags` attribute as `-l<name>`.

```python
# gazelle:cabal_haskell_package_repo stackage
```
Specifies the name of the repository from where Cabal packages
are provided. The default value is `stackage`. The user is
responsible for setting up the repository. The repository for
tools is constructed automatically by appending `-exe` to this.

## Dependency resolution

In general, package names in the `build-depends` field are mapped to
`@stackage//:<package_name>`, except if there is a `haskell_library`
rule in the current repository with the same name, in which
case such a target is added to the `deps` attribute instead.

If there is a `ghc_plugin` rule named as `<package name>-plugin` and
`<package name>` is listed in the `build-depends` field, the
corresponding label is added to the `plugins` attribute and omitted
in `deps`.

Tools in the `build-tool-depends` field are mapped to
`@stackage-exe//<package_name>:<executable_name>` and added to the
`tools` attribute, unless there is a `haskell_binary` rule in the
current repository named as the executable, in which case such a
target is added to the `tools` attribute instead. The path to the
tool is defined as a CPP macro via the compiler flags of the
corresponding rule as `PACKAGE_NAME_EXECUTABLE_NAME_PATH`
(e.g. `TASTY_DISCOVER_TASTY_DISCOVER_PATH`).

Currently, `gazelle` is unable to merge the contents of the
`components` [attribute][gazelle-937] of the `stack_snapshot` rule if
the attribute is already present. But it is still possible to have the
attribute automatically generated when it is not present yet.
Thus, refreshing the attribute requires to manually remove it first.

Files in the `data-files` field are placed verbatim in the `data`
attribute of the `haskell_library` rule if a library component is
present in the Cabal file. Otherwise, the `data-files` field is
ignored.

## Implementation

`gazelle_cabal` extracts data from Cabal files using [cabalscan][cabalscan],
a command line tool written in Haskell, which presents the extracted data
in json format to the `go` part.

The [go part][go-part] consists of a set of functions written in the
[go][go] programming language, which `gazelle` will invoke to
generate Haskell rules. The most important functions are:

* `GenerateRules`: calls the `cabalscan` command-line tool and
  produces rules that contain no information about dependencies.

* `Resolve`: adds to the previously generated rules all the
  information about dependencies (`deps`, `plugins`, and `tools`).

* `UpdateRepos`: scans the `BUILD` files for Haskell rules, extracts
  their dependencies, and puts them in a `stack_snapshot` rule in the
  `WORKSPACE` file.

## Limitations

Despite that `gazelle_cabal` can produce most of the build configuration
from Cabal files, Haskell dependencies brought with `stack_snapshot`
might fail to build if their Cabal files use sub-libraries or some particular
custom `Setup.hs` files. In these cases, the simpler route to adoption could
be to patch the problematic dependencies and add them to a local `stack`
snapshot (see the [local_snapshot][local_snapshot] attribute of
`stack_snapshot`).

Also, support for fields in Cabal files is added in as-needed fashion,
which means that when `gazelle_cabal` is tried on new Cabal files it could
require patches to deal with some unsupported features. The Cabal files in
the [example repo][example] rehearse the range of features currently
supported.

* `.hs-boot` and `.lhs-boot` files are unsupported.
* If Cabal components use different dependencies depending on Cabal
  flags, `gazelle_cabal` will only generate the rules for the
  configuration with default flag values.

## Sponsors

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
[![Symbiont](https://imgur.com/KPV3lTY.png)](https://symbiont.io)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
[![Tweag I/O](https://i.imgur.com/KPEy44T.png)](http://tweag.io)

`gazelle_cabal` was funded by [Symbiont](https://www.symbiont.io/)
and is maintained by [Tweag I/O](http://tweag.io/).

Have questions? Need help? Tweet at
[@tweagio](http://twitter.com/tweagio).

[gazelle-937]: https://github.com/bazelbuild/bazel-gazelle/issues/937
[bazel]: https://bazel.build
[cabal]: https://www.haskell.org/cabal
[cabalscan]: cabalscan/exe/Main.hs
[gazelle_cabal_dependencies]: defs.bzl
[example]: example
[fix-command]: https://github.com/bazelbuild/bazel-gazelle#fix-and-update
[gazelle]: https://github.com/bazelbuild/bazel-gazelle
[go-part]: gazelle_cabal/lang.go
[go]: https://golang.org
[rules_haskell]: https://github.com/tweag/rules_haskell
[local_snapshot]: https://api.haskell.build/haskell/cabal.html#stack_snapshot-local_snapshot
