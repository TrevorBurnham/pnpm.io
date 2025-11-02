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

#### Lifecycle Hooks

| Hook Function                                         | Process                                                    | Uses                                               |
|-------------------------------------------------------|------------------------------------------------------------|----------------------------------------------------|
| `hooks.readPackage(pkg, context): pkg`                | Called after pnpm parses the dependency's package manifest | Allows you to mutate a dependency's `package.json` |
| `hooks.preResolution(options, logger): result`        | Called after reading lockfiles, before resolving dependencies | Allows you to modify lockfiles and force full resolution |
| `hooks.afterAllResolved(lockfile, context): lockfile` | Called after the dependencies have been resolved.          | Allows you to mutate the lockfile.                 |

#### Custom Resolvers

Custom resolvers allow you to override how specific packages are resolved and installed:

| Resolver Method                                  | Process                                                                      | Uses                                                        |
|--------------------------------------------------|------------------------------------------------------------------------------|-------------------------------------------------------------|
| `supportsDescriptor(descriptor): boolean`        | Called during resolution phase to check if resolver handles this package    | Match packages by name, scope, or other criteria            |
| `resolve(descriptor, opts): ResolveResult`       | Called during resolution phase to resolve matched package                   | Return custom resolution (tarball location, integrity, etc) |
| `supportsLockfileResolution(pkgId, resolution): boolean` | Called during headless install to check if resolver handles this lockfile entry | Match custom resolution types in existing lockfiles         |
| `fromLockfileResolution(pkgId, resolution, opts): Resolution` | Called during headless install to convert lockfile entry to fetchable resolution | Convert custom lockfile format back to tarball/directory/git resolution |

### `hooks.readPackage(pkg, context): pkg | Promise<pkg>`

Allows you to mutate a dependency's `package.json` after parsing and prior to
resolution. These mutations are not saved to the filesystem, however, they will
affect what gets resolved in the lockfile and therefore what gets installed.

:::tip TypeScript

If you're using TypeScript or want IntelliSense in your editor, you can import types:

```typescript
import type { ReadPackageHook } from '@pnpm/hooks.types'
```

:::

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

:::tip TypeScript

If you're using TypeScript or want IntelliSense in your editor, you can import types:

```typescript
import type { AfterAllResolvedHook } from '@pnpm/hooks.types'
```

:::

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

### `hooks.preResolution(options, logger): Promise<{ forceFullResolution?: boolean } | undefined>`

This hook is executed after reading and parsing the lockfiles of the project, but before resolving dependencies. It allows modifications to the lockfile objects.

:::tip TypeScript

If you're using TypeScript or want IntelliSense in your editor, you can import types:

```typescript
import type { PreResolutionHook } from '@pnpm/hooks.types'
```

:::


#### Arguments

* `options.existsCurrentLockfile` - A boolean that is true if the lockfile at `node_modules/.pnpm/lock.yaml` exists.
* `options.currentLockfile` - The lockfile object from `node_modules/.pnpm/lock.yaml`.
* `options.existsNonEmptyWantedLockfile` - A boolean that is true if the lockfile at `pnpm-lock.yaml` exists.
* `options.wantedLockfile` - The lockfile object from `pnpm-lock.yaml`.
* `options.lockfileDir` - The directory where the wanted lockfile is found.
* `options.storeDir` - The location of the store directory.
* `options.registries` - A map of scopes to registry URLs.
* `logger.info(message)` - Log an informational message.
* `logger.warn(message)` - Log a warning message.

#### Return Value

The hook can return an object with the following optional properties:

* `forceFullResolution` - When set to `true`, pnpm will re-resolve all dependencies, even if they exist in the lockfile. This is useful when the hook modifies the lockfile and needs to ensure all dependencies are refreshed.

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

### `hooks.resolvers`

Added in: v11.0.0

Custom resolvers allow you to integrate pnpm with internal package registries that use non-standard versioning systems or custom resolution strategies. This is useful for:

- Internal registries with custom versioning (hashes, build IDs, etc.)
- Redirecting packages to local caches or mirrors
- Integrating with proprietary package management systems
- Supporting custom resolution protocols

Each resolver is an object that implements the following interface. All methods are optional, allowing you to implement only the functionality you need:

```typescript
// TypeScript users can import types from @pnpm/hooks.types:
// import type { ResolverPlugin, PackageDescriptor, ResolveOptions, ResolveResult } from '@pnpm/hooks.types'

interface ResolverPlugin {
  // Resolution phase: resolve package descriptors
  supportsDescriptor?: (descriptor: PackageDescriptor) => boolean | Promise<boolean>
  resolve?: (descriptor: PackageDescriptor, opts: ResolveOptions) => ResolveResult | Promise<ResolveResult>

  // Headless install phase: convert lockfile entry back to fetchable resolution
  supportsLockfileResolution?: (pkgId: string, resolution: unknown) => boolean | Promise<boolean>
  fromLockfileResolution?: (pkgId: string, resolution: unknown, opts: ResolveOptions) => unknown | Promise<unknown>
}
```

**Note**: You must implement either:
- Both `supportsDescriptor` and `resolve` (for resolution phase), or
- Both `supportsLockfileResolution` and `fromLockfileResolution` (for headless install phase only), or
- All four methods (for full control over both phases)

#### Resolution Flow

1. **Resolution phase** (fresh install): When pnpm encounters a package, it calls `supportsDescriptor()` on each resolver. The first resolver that returns `true` handles the package via `resolve()`.
2. **Headless install phase** (frozen lockfile): When reading from the lockfile, pnpm calls `supportsLockfileResolution()` to find the appropriate resolver, then calls `fromLockfileResolution()` to convert the lockfile entry back to a fetchable resolution.

#### Arguments

**PackageDescriptor:**
- `name` - Package name (e.g., `"@internal/foo"`)
- `range` - Version range or specifier (e.g., `"1.2.3"`, `"^2.0.0"`)
- `type` - Optional dependency type: `"prod"`, `"dev"`, or `"optional"`

**ResolveOptions:**
- `lockfileDir` - Directory containing the lockfile
- `projectDir` - Directory of the project being installed
- `preferredVersions` - Map of preferred versions for packages

**ResolveResult:**
- `id` - Unique identifier for the resolved package
- `resolution` - Resolution object used for fetching (e.g., `{ tarball: "...", integrity: "..." }`)
- `manifest` - Optional package.json content
- `resolvedVia` - String identifying which resolver was used
- `getLockfileResolution` - Optional function to transform resolution for lockfile storage (receives resolution, returns lockfile-compatible format)

#### Usage Examples

##### Normal resolution

```js title=".pnpmfile.cjs"
module.exports = {
  hooks: {
    resolvers: [
      {
        // Match the package you want to redirect
        supportsDescriptor(descriptor) {
          return descriptor.name === 'lodash'
        },

        // Resolve to a local tarball
        resolve(descriptor, opts) {
          return {
            id: 'file:./local-packages/lodash-custom.tgz',
            resolution: {
              tarball: 'file:./local-packages/lodash-custom.tgz',
              integrity: 'sha512-abc123...'
            },
            resolvedVia: 'local-tarball'
          }
        }
      }
    ]
  }
}
```

With this change, `lodash` will always be installed from `./local-packages/lodash-custom.tgz` (relative to the project root).

##### Custom lockfile format

If your resolver needs to store resolution information differently in the lockfile than it's used for fetching, use `getLockfileResolution`:

```js title=".pnpmfile.cjs"
module.exports = {
  hooks: {
    resolvers: [
      {
        supportsDescriptor(descriptor) {
          return descriptor.name.startsWith('@internal/')
        },
        resolve(descriptor, opts) {
          // Fetch from build server with hash
          const buildHash = lookupLatestBuild(descriptor.name, descriptor.range)
          return {
            id: `@internal/${descriptor.name}@${buildHash}`,
            resolution: {
              tarball: `https://builds.example.com/${descriptor.name}/${buildHash}.tgz`,
              integrity: `sha512-${buildHash}`
            },
            // Store portable version info in lockfile instead of build-specific URLs
            getLockfileResolution: (resolution) => ({
              type: 'internal-build',
              package: descriptor.name,
              version: descriptor.range,
              hash: buildHash
            }),
            resolvedVia: 'internal-resolver'
          }
        },
        // When reading from lockfile, reconstruct the resolution
        supportsLockfileResolution: (pkgId, resolution) => {
          return resolution.type === 'internal-build'
        },
        fromLockfileResolution: (pkgId, resolution, opts) => {
          return {
            tarball: `https://builds.example.com/${resolution.package}/${resolution.hash}.tgz`,
            integrity: `sha512-${resolution.hash}`
          }
        }
      }
    ]
  }
}
```

This allows you to store a portable format in the lockfile (like a semantic version) while fetching from environment-specific locations (like different build servers).

##### Lockfile resolution

```js title=".pnpmfile.cjs"
module.exports = {
  hooks: {
    resolvers: [
      {
        // Only implement lockfile resolution - no descriptor resolution
        supportsLockfileResolution: (pkgId, resolution) => {
          // Intercept all tarball resolutions
          return 'tarball' in resolution
        },
        fromLockfileResolution: (pkgId, resolution, opts) => {
          // Redirect to local cache
          return {
            tarball: `file://local-cache/${pkgId}.tgz`,
            integrity: resolution.integrity
          }
        }
      }
    ]
  }
}
```

With this change, all packages that have a tarball specified in the lockfile will be resolved from the tarball cache in `file://local-cache/`.

#### Multiple Resolvers

You can register multiple resolvers. They are tried in order, and the first resolver that returns `true` from `supportsDescriptor()` handles the package:

```js title=".pnpmfile.cjs"
module.exports = {
  hooks: {
    resolvers: [
      {
        supportsDescriptor: (desc) => desc.name === 'lodash',
        resolve: (desc) => ({ /* custom resolution for lodash */ })
      },
      {
        supportsDescriptor: (desc) => desc.name.startsWith('@mycompany/'),
        resolve: (desc) => ({ /* custom resolution for @mycompany/* packages */ })
      }
    ]
  }
}
```

#### Synchronous vs Asynchronous

All resolver methods support both synchronous and asynchronous implementations:

```js
// Synchronous (faster for simple checks)
supportsDescriptor(descriptor) {
  return descriptor.name.startsWith('@mycompany/')
}

// Asynchronous (when you need to check remote sources)
async supportsDescriptor(descriptor) {
  return await checkIfPackageExistsInLocalCache(descriptor.name)
}
```

For best performance, use synchronous methods when possible, especially for `supportsDescriptor` and `supportsLockfileResolution`, as they're called frequently.

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
