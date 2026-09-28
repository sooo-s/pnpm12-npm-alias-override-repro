# pnpm12-npm-alias-override-repro

This repo is a minimal reproduction of a pnpm 12 regression. When pnpm re-resolves a locked dependency, it drops the `npm:` alias of an override and resolves the real package instead.

- Fails: pnpm 12.0.0, 12.4.2, 12.5.1, 12.6.0 and 12.7.0
- Works: pnpm 11.27.1 and 11.28.0

## Setup

`pnpm-workspace.yaml`:

```yaml
overrides:
  vite@*: npm:@voidzero-dev/vite-plus-core@0.3.3
```

`vite-node@6.0.0` depends on `vite`. The committed `pnpm-lock.yaml` resolves that dependency to the alias:

```text
  vite-node@6.0.0(@types/node@25.9.7):
    dependencies:
      vite: '@voidzero-dev/vite-plus-core@0.3.3(@types/node@25.9.7)'
```

`devEngines` pins pnpm 12.7.0, and `registries` pins registry.npmjs.org. Neither is needed to trigger the bug.

## Reproduction

```sh
pnpm install --frozen-lockfile                  # OK
pnpm add -D @types/node@25.9.8 --lockfile-only  # fails
```

`@types/node` is an optional peer of `@voidzero-dev/vite-plus-core`. Bumping it forces pnpm to re-resolve the locked `vite-node` snapshot. The same error occurs without `--lockfile-only`.

### Actual

```
Error: ERR_PNPM_NO_MATCHING_VERSION

  × adding a new package
  ╰─▶ Failed to resolve dependency tree: No matching version found for
      vite@0.3.3 while fetching it from https://registry.npmjs.org/
  help: The latest release of vite is "8.3.1".

        Other releases are:
          * alpha: 6.0.0-alpha.24
          * beta: 8.3.0-beta.1
          * previous: 7.3.6

        If you need the full list of all 753 published versions run "pnpm view
        vite versions".
```

### Expected

`vite-node`'s `vite` dependency keeps resolving to `@voidzero-dev/vite-plus-core@0.3.3`. That is what happens on pnpm 11, and on pnpm 12 when `pnpm-lock.yaml` is deleted first.

## Trying another pnpm version

Because of `onFail: download`, pnpm switches to the version in `devEngines.packageManager.version`, whatever pnpm you have installed. To try another version, edit that field. Then run `pnpm install` instead of `pnpm install --frozen-lockfile`, because the committed lockfile records pnpm 12.7.0. Then run the bump.

## When the version also exists under the real name

`vite@0.3.2` exists on the registry as a 2020 release of the real `vite`, but `vite@0.3.3` does not. With 0.3.2 the install succeeds silently and locks the wrong package:

```sh
sed -i.bak 's/0\.3\.3/0.3.2/' pnpm-workspace.yaml
pnpm install                                    # locks the alias
pnpm add -D @types/node@25.9.8 --lockfile-only  # succeeds
```

After the bump, `pnpm-lock.yaml` contains:

```text
  vite-node@6.0.0:
    dependencies:
      cac: 7.0.0
      es-module-lexer: 2.3.2
      obug: 2.2.1
      pathe: 2.0.3
      vite: 0.3.2
```
