---
id: pnpmfile
title: .pnpmfile.cjs
---

pnpm lets you hook directly into the installation process via special functions
(hooks). Hooks can be declared in a file called `.pnpmfile.cjs`.

By default, `.pnpmfile.cjs` should be located in the same directory as the
lockfile. For instance, in a [workspace](workspaces.md) with a shared lockfile,
`.pnpmfile.cjs` should be in the root of the monorepo.

## Hooks

### TL;DR

| Hook Function                                         | Process                                                    | Uses                                               |
|-------------------------------------------------------|------------------------------------------------------------|----------------------------------------------------|
| `hooks.readPackage(pkg, context): pkg`                | Called after pnpm parses the dependency's package manifest | Allows you to mutate a dependency's `package.json`. |
| `hooks.afterAllResolved(lockfile, context): lockfile` | Called after the dependencies have been resolved.          | Allows you to mutate the lockfile.                 |
| `hooks.adapters`                                      | Called throughout package resolution.              | Allows you to register custom resolvers and fetchers.       |

### `hooks.readPackage(pkg, context): pkg | Promise<pkg>`

Allows you to mutate a dependency's `package.json` after parsing and prior to
resolution. These mutations are not saved to the filesystem, however, they will
affect what gets resolved in the lockfile and therefore what gets installed.

Note that you will need to delete the `pnpm-lock.yaml` if you have already
resolved the dependency you want to modify.

:::tip

If you need changes to `package.json` saved to the filesystem, you need to use the [`pnpm patch`] command and patch the `package.json` file.
This might be useful if you want to remove the `bin` field of a dependency for instance.

:::

#### Arguments

* `pkg` - The manifest of the package. Either the response from the registry or
the `package.json` content.
* `context` - Context object for the step. Method `#log(msg)` allows you to use
a debug log for the step.

#### Usage

Example `.pnpmfile.cjs` (changes the dependencies of a dependency):

```js
function readPackage(pkg, context) {
  // Override the manifest of foo@1.x after downloading it from the registry
  if (pkg.name === 'foo' && pkg.version.startsWith('1.')) {
    // Replace bar@x.x.x with bar@2.0.0
    pkg.dependencies = {
      ...pkg.dependencies,
      bar: '^2.0.0'
    }
    context.log('bar@1 => bar@2 in dependencies of foo')
  }

  // This will change any packages using baz@x.x.x to use baz@1.2.3
  if (pkg.dependencies.baz) {
    pkg.dependencies.baz = '1.2.3';
  }

  return pkg
}

module.exports = {
  hooks: {
    readPackage
  }
}
```

#### Known limitations

Removing the `scripts` field from a dependency's manifest via `readPackage` will
not prevent pnpm from building the dependency. When building a dependency, pnpm
reads the `package.json` of the package from the package's archive, which is not
affected by the hook. In order to ignore a package's build, use the
[neverBuiltDependencies](settings.md#neverbuiltdependencies) field.

### `hooks.updateConfig(config): config | Promise<config>`

Added in: v10.8.0

Allows you to modify the configuration settings used by pnpm. This hook is most useful when paired with [configDependencies](config-dependencies), allowing you to share and reuse settings across different Git repositories.

For example, [@pnpm/plugin-better-defaults](https://github.com/pnpm/plugin-better-defaults) uses the `updateConfig` hook to apply a curated set of recommended settings.

#### Usage example

```js title=".pnpmfile.cjs"
module.exports = {
  hooks: {
    updateConfig (config) {
      return Object.assign(config, {
        enablePrePostScripts: false,
        optimisticRepeatInstall: true,
        resolutionMode: 'lowest-direct',
        verifyDepsBeforeRun: 'install',
      })
    }
  }
}
```

### `hooks.afterAllResolved(lockfile, context): lockfile | Promise<lockfile>`

Allows you to mutate the lockfile output before it is serialized.

#### Arguments

* `lockfile` - The lockfile resolutions object that is serialized to
`pnpm-lock.yaml`.
* `context` - Context object for the step. Method `#log(msg)` allows you to use
a debug log for the step.

#### Usage example

```js title=".pnpmfile.cjs"
function afterAllResolved(lockfile, context) {
  // ...
  return lockfile
}

module.exports = {
  hooks: {
    afterAllResolved
  }
}
```

#### Known Limitations

There are none - anything that can be done with the lockfile can be modified via
this function, and you can even extend the lockfile's functionality.

### `hooks.preResolution(options): Promise<void>`

This hook is executed after reading and parsing the lockfiles of the project, but before resolving dependencies. It allows modifications to the lockfile objects.

#### Arguments

* `options.existsCurrentLockfile` - A boolean that is true if the lockfile at `node_modules/.pnpm/lock.yaml` exists.
* `options.currentLockfile` - The lockfile object from `node_modules/.pnpm/lock.yaml`.
* `options.existsNonEmptyWantedLockfile` - A boolean that is true if the lockfile at `pnpm-lock.yaml` exists.
* `options.wantedLockfile` - The lockfile object from `pnpm-lock.yaml`.
* `options.lockfileDir` - The directory where the wanted lockfile is found.
* `options.storeDir` - The location of the store directory.
* `options.registries` - A map of scopes to registry URLs.

### `hooks.importPackage(destinationDir, options): Promise<string | undefined>`

This hook allows to change how packages are written to `node_modules`. The return value is optional and states what method was used for importing the dependency, e.g.: clone, hardlink.

#### Arguments

* `destinationDir` - The destination directory where the package should be written.
* `options.disableRelinkLocalDirDeps`
* `options.filesMap`
* `options.force`
* `options.resolvedFrom`
* `options.keepModulesDir`

### `hooks.fetchers`

This hook allows to override the fetchers that are used for different types of dependencies. It is an object that may have the following fields:

* `localTarball`
* `remoteTarball`
* `gitHostedTarball`
* `directory`
* `git`

### `hooks.adapters`

Added in: v11.0.0

Adapters extend pnpm's resolution and fetching logic to support custom package sources, protocols, or package management systems.

#### Adapter Interface

An adapter is an object that can implement any combination of the following methods:

##### `canResolve(wantedDependency): boolean | Promise<boolean>`

Determines whether this adapter can resolve a given wanted dependency.

**Arguments:**
- `wantedDependency` - Object with:
  - `alias` - The package name or alias as it appears in package.json
  - `bareSpecifier` - The version range, git URL, file path, or other specifier

**Returns:** `true` if this adapter can resolve the package, `false` otherwise. This determines whether `resolve` will be called.

##### `resolve(wantedDependency, opts): ResolveResult | Promise<ResolveResult>`

Resolves a wanted dependency to specific package metadata and resolution information.

**Arguments:**
- `wantedDependency` - The wanted dependency (same as `canResolve`)
- `opts` - Object with:
  - `lockfileDir` - Directory containing the lockfile
  - `projectDir` - The project root directory
  - `preferredVersions` - Map of package names to preferred versions

**Returns:** Object with:
- `id` - Unique package identifier (e.g., `'custom-pkg@1.0.0'`)
- `resolution` - Resolution metadata. This can be:
  - Standard resolution (e.g., `{ tarball: 'https://...' }`)
  - Custom resolution with scoped type (e.g., `{ type: '@company/cdn', url: '...' }`)

Custom resolutions must be handled with `canFetch`/`fetch`.

:::warning Custom Resolution Types

Custom resolutions must use an `@`-scoped type (e.g., `@company/custom-type`) to avoid conflicts with pnpm's built-in resolution types (`tarball`, `directory`, `git`, `binary`).

:::

##### `shouldForceResolve(wantedDependency): boolean | Promise<boolean>`

Determines whether packages matching this wanted dependency should be re-resolved even during headless installs.

**Arguments:**
- `wantedDependency` - The wanted dependency (same as `canResolve`)

**Returns:** `true` to force re-resolution, `false` otherwise.

This is useful when you want to update a package with your `resolve` function even if the lockfile is up-to-date.

:::note

`shouldForceResolve` is skipped during frozen lockfile installs, as no resolution is allowed in that mode.

:::

##### `canFetch(pkgId, resolution): boolean | Promise<boolean>`

Determines whether this adapter can fetch a package with the given resolution.

**Arguments:**
- `pkgId` - The unique package identifier from the resolution phase
- `resolution` - The resolution object from the `resolve` method

**Returns:** `true` if this adapter can fetch the package, `false` otherwise.

##### `fetch(cafs, resolution, opts, fetchers): FetchResult | Promise<FetchResult>`

Fetches package files and returns metadata about the fetched package.

**Arguments:**
- `cafs` - Content-addressable file system interface for storing files
- `resolution` - The resolution object (same as passed to `canFetch`)
- `opts` - Fetch options including:
  - `lockfileDir` - Directory containing the lockfile
  - `filesIndexFile` - Path for the files index
  - `onStart` - Optional callback when fetch starts
  - `onProgress` - Optional progress callback
- `fetchers` - Object containing pnpm's standard fetchers for delegation:
  - `remoteTarball` - Fetcher for remote tarballs
  - `localTarball` - Fetcher for local tarballs
  - `gitHostedTarball` - Fetcher for GitHub/GitLab/Bitbucket tarballs
  - `directory` - Fetcher for local directories
  - `git` - Fetcher for git repositories

**Returns:** Object with:
- `filesIndex` - Map of relative file paths to their physical locations. For remote packages, these are paths in pnpm's content-addressable store (CAFS). For local packages (when `local: true`), these are absolute paths to files on disk.
- `manifest` - Optional. The package.json from the fetched package. If not provided, pnpm will read it from disk when needed. Providing it avoids an extra file I/O operation and is recommended when you have the manifest data readily available (e.g., already parsed during fetch).
- `requiresBuild` - Boolean indicating whether the package has build scripts that need to be executed. Set to `true` if the package has `preinstall`, `install`, or `postinstall` scripts, or contains `binding.gyp` or `.hooks/` files. Standard fetchers determine this automatically using the manifest and file list.
- `local` - Optional. Set to `true` to load the package directly from disk without copying to pnpm's store. When `true`, `filesIndex` should contain absolute paths to files on disk, and pnpm will hardlink them to `node_modules` instead of copying. This is how the directory fetcher handles local dependencies (e.g., `file:../my-package`).

:::tip Delegating to Standard Fetchers

Custom fetchers can delegate to pnpm's built-in fetchers using the `fetchers` parameter.

:::

#### Usage Examples

##### Basic Custom Resolver

This example shows an adapter that resolves packages from a custom registry:

```js title=".pnpmfile.cjs"
module.exports = {
  hooks: {
    adapters: [
      {
        // Only handle packages with @company scope
        canResolve: (wantedDependency) => {
          return wantedDependency.alias.startsWith('@company/')
        },

        resolve: async (wantedDependency, opts) => {
          // Fetch metadata from custom registry
          const response = await fetch(
            `https://custom-registry.company.com/${wantedDependency.alias}/${wantedDependency.bareSpecifier}`
          )
          const metadata = await response.json()

          return {
            id: `${metadata.name}@${metadata.version}`,
            resolution: {
              tarball: metadata.tarballUrl,
              integrity: metadata.integrity
            }
          }
        }
      }
    ]
  }
}
```

##### Basic Custom Fetcher

This example shows an adapter that tells pnpm to fetch certain packages from a different source than the one specified in the package resolution.

```js title=".pnpmfile.cjs"
module.exports = {
  hooks: {
    adapters: [
      {
        canFetch: (pkgId, resolution) => {
          return pkgId.startsWith('@company/')
        },

        fetch: async (cafs, resolution, opts, fetchers) => {
          // Delegate to pnpm's tarball fetcher
          const tarballResolution = {
            tarball: resolution.tarballUrl.replace(
              `https://registry.npmjs.org/`,
              `https://custom-registry.company.com/`
            ),
            integrity: resolution.integrity
          }

          return fetchers.remoteTarball(cafs, tarballResolution, opts)
        }
      }
    ]
  }
}
```

##### Custom Resolution Type

This example shows an adapter with both a custom resolver and a custom fetcher, providing full control over lockfile entries.

```js title=".pnpmfile.cjs"
module.exports = {
  hooks: {
    adapters: [
      {
        canResolve: (wantedDependency) => {
          return wantedDependency.alias.startsWith('@internal/')
        },

        resolve: async (wantedDependency) => {
          return {
            id: `${wantedDependency.alias}@${wantedDependency.bareSpecifier}`,
            resolution: {
              type: '@company/internal',
              directory: `/packages/${wantedDependency.alias}/${wantedDependency.bareSpecifier}`
            }
          }
        },

        canFetch: (pkgId, resolution) => {
          return resolution.type === '@company/internal'
        },

        fetch: async (cafs, resolution, opts, fetchers) => {
          // Delegate to pnpm's directory fetcher for local packages
          // Transform custom resolution to standard directory resolution
          const directoryResolution = {
            type: 'directory',
            directory: resolution.directory
          }

          return fetchers.directory(cafs, directoryResolution, opts)
        }
      }
    ]
  }
}
```

#### Adapter Priority

When multiple adapters are provided, they are checked in order. The first adapter where `canResolve` returns `true` will be used for resolution. The same applies for `canFetch` during the fetch phase.

## Finders

Added in: v10.16.0

Finder functions are used with `pnpm list` and `pnpm why` via the `--find-by` flag.

Example:

```js title=".pnpmfile.cjs"
module.exports = {
  finders: {
    react17: (ctx) => {
      return ctx.readManifest().peerDependencies?.react === "^17.0.0"
    }
  }
}
```

Usage:

```
pnpm why --find-by=react17
```

See [Finders] for more details.

[Finders]: ./finders.md

## Related Configuration

import IgnorePnpmfile from './settings/_ignorePnpmfile.mdx'

<IgnorePnpmfile />

import Pnpmfile from './settings/_pnpmfile.mdx'

<Pnpmfile />

import GlobalPnpmfile from './settings/_globalPnpmfile.mdx'

<GlobalPnpmfile />

[`pnpm patch`]: ./cli/patch.md
