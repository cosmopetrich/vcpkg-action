# vcpkg-manifest-install-action

Installs packages from a [vcpkg](https://github.com/microsoft/vcpkg) manifest.

There are at least a couple of other Actions which you should consider before using this one.

- [lukka/run-vcpkg](https://github.com/lukka/run-vcpkg)
- [johnwason/vcpkg-action](https://github.com/johnwason/vcpkg-action)

They are likely a better suit to your use case, as well as being more better supported and tested.
This action was designed for how I tend to use vcpkg. It's almost certainly not idomatic and may even adopt some antipatterns.

Specifcally, this action was designed with the following goals in mind.

- Use an arbitrary path as the manifest, not just $PWD/vcpkg.json.
- Place installed packages in user-specified directory, not just $PWD/vcpkg_installed.
- Operate outside the default Actions workspace. No installing to $PWD/vcpkg.
- Allow setting a toolset version without creating an overlay triplet.
- Upload buildtree logs as artifacts if there's an error.
- Provide an easy to parse file with information on the build.
- Just YAML and a bit of bash.

To that end there are a few caveats and intentional non-features.

- Only supports the `install` operation. No `integrate` etc.
- Only operates in manifest mode. No installing packages by name.
- No default value for triplet. The naming scheme used by vspkg is strange (compare settings in `x64-windows` and `x64-linux`).
- Not tested on MacOS runners at this stage. Might still work there?

Possible future improvements include.

- Support for arbitrary overlay triplets.
  - Will add this if/when I need it.

## Usage

Minimal usage requires only `output-directory` and `triplet`.

```yaml
- name: vcpkg
  uses: cosmopetrich/vcpkg-action@v1
  with:
    output-directory: packages
    triplet: x64-linux
```

This will use the `vcpkg.json` in the root of the Actions workspace and place the results into `packages/` in the same directory.

To [enable caching](https://learn.microsoft.com/en-us/vcpkg/consume/binary-caching-github-actions-cache) you will need to set `${{ caching-auth-token }}`.
This is required due to to [actions/runner#3046](https://github.com/actions/runner/issues/3046).

```yaml
- name: vcpkg
  uses: cosmopetrich/vcpkg-action@v1
  with:
    caching-auth-token: ${{ github.token }}
    output-directory: packages
    triplet: x64-linux
```

To use the `ACTIONS_RUNTIME_TOKEN` instead of the full `github.token` then you'll need to use an `actions/github-script` action to re-export it.

If your vcpkg manifest isn't named `vcpkg.json` or isn't in the root of the Actions workspace then you can specify its location.

```yaml
- name: vcpkg
  uses: cosmopetrich/vcpkg-action@v1
  with:
    output-directory: packages
    triplet: x64-linux
    vcpkg-manifest: dependencies/vcpkg-production.json
```

See below for details on the `output-directory` and `vcpkg-manifest` inputs.

## Advanced usage

### Setting a toolset version

Ordinarily, vcpkg expects that users will construct their own triplets if they wish to build with a specific toolset when multiple are available.
Since there are almost always multiple versions of MSVC available this can get quite annoying. The action can take care of this for you if you specify a `vcpkg-platform-toolset-version`.

```yaml
- name: vcpkg
  uses: cosmopetrich/vcpkg-action@v1
  with:
    output-directory: packages
    triplet: x64-linux
    vcpkg-platform-toolset-version: "1.2.3.4"
```

This will cause the action to find the specified `triplet`, copy it, and ammend it with the necessary instructions.

Note that it will not actually ensure that the necessary version is available.

### Overlays

You can provide a directory containing your [overlay ports](https://learn.microsoft.com/en-us/vcpkg/concepts/overlay-ports).

```yaml
- name: vcpkg
  uses: cosmopetrich/vcpkg-action@v1
  with:
    output-directory: packages
    triplet: x64-linux
    overlay-ports-directory: .vcpkg/ports
```

Note that [overlay triplets](https://learn.microsoft.com/en-us/vcpkg/users/examples/overlay-triplets-linux-dynamic)
are not currently supported, aside from the `vcpkg-platform-toolset-version` feature described above.

## Details

### Behaviour of output-directory and vcpkg-manifest

The vcpkg executable expects to be run from a directory containing a manifest and will place installed packages in that same directory.
This necessitates either an annoying process of shuffling files around during CI or a repository structure which may not suite some projects.
It is only possible to override this behaviour to a limited extent using vcpkg options which are flagged as experimental.

As a workaround, this action provides the `${{output-directory}}` and `${{vcpkg-manifest}}` inputs.
When the action runs, the file specified in `${{vcpkg-manifest}}` will be copied to a temporary directory as `vcpkg.json`
and then `vcpkg install` will be run against this temporary directory.
Once the installation has completed, the contents of the temporary directory's `vcpkg_installed/${{triplet}}` will be moved to `${{output-directory}}`.

- The `${{ output-directory}}` will be created if it does not exist.
- By daafult there will not be a `${{triplet}}` subdirectory under `${{output-directory}}`.
  - If vcpkg installed `boost` with `x64-windows` then after this action runs `${{output-directory}}/include/boost/` will exist as opposed to `${{output-directory}}/x64-windows/include/boost`.

### Build information

Information on the build is saved to a pair of JSON files in the root of the output directory: `vcpkg-build-info.json` and `vcpkg-installed-files.txt`. This information was previously provided via outputs. However, it turns out accessing those when they're run in a matrix build was a real pain. See [community discussion #17245](https://github.com/orgs/community/discussions/17245) and [actions/runner#2477](https://github.com/actions/runner/pull/2477).

The JSON file is formatted as follows.

```json5
{
  // Short and long git SHAs used
  // The SHA of HEAD if no baseline was specified
  "baseline_long": "4f746bc66438fce2b900c3ba6094a483b871b045",
  "baseline_short" "4f746bc",
  // MSVC or GNU
  "compiler_id": "MSVC",
  // On GNU these will be the same
  // On MSVC a the version is a mostly meaningless value
  "compiler_version": "19.36.32546.0",
  "compiler_toolset": "14.36.32532",
}
```

The txt file will be formatted like the example below. It is generated using [vcpkg list](https://learn.microsoft.com/en-us/vcpkg/commands/list) command with feature entries stripped out. The results should be lexically ordered.

```
cppgraphqlgen 4.5.7#1
pegtl 3.2.8
rapidjson 2023-07-17#1
```

The `#` part of the version number refers to which version of the vcpkg port configuration was used.
