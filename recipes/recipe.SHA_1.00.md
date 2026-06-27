# Recipe: SHA/main → v1.0.0

One-time migration for a card project that consumed the shared toolchain at a bare SHA or `@main`
under its old name `ha-shared`. v1.0.0 renames the package to `ha-card-shared`, so this migration
also swaps every reference. Run it once per consumer; afterwards dependabot keeps it updated.

Run from the consumer repo root, on a branch.

```bash
# 1. Swap the dependency: drop the old name, add the renamed package pinned to the tag
npm uninstall ha-shared
npm install github:marcintk/ha-card-shared#v1.0.0 --save-dev

# 2. Rename every tracked ha-shared reference to ha-card-shared in one pass — configs, source,
#    workflows, the git hook shim, CLAUDE.md (git grep skips node_modules and binaries)
git grep -lIz 'ha-shared' | xargs -0 sed -i 's|ha-shared|ha-card-shared|g'

# 3. Rename + pin the workflow refs (the shared workflows gained a shared- prefix; @main is gone)
sed -i \
  -e 's|ha-card-shared/.github/workflows/build-and-test.yml@[^ "]*|ha-card-shared/.github/workflows/shared-build-and-test.yml@v1.0.0|g' \
  -e 's|ha-card-shared/.github/workflows/hacs-validation.yml@[^ "]*|ha-card-shared/.github/workflows/shared-hacs-validation.yml@v1.0.0|g' \
  -e 's|ha-card-shared/.github/workflows/publish-release.yml@[^ "]*|ha-card-shared/.github/workflows/shared-publish-release.yml@v1.0.0|g' \
  -e 's|ha-card-shared/.github/workflows/deploy-demo-page.yml@[^ "]*|ha-card-shared/.github/workflows/shared-deploy-demo-page.yml@v1.0.0|g' \
  .github/workflows/*.yml

# 4. Adopt the shared prettier config; drop the local one
npm pkg set prettier="ha-card-shared/prettier.config.json"
rm -f .prettierrc .prettierrc.json .prettierrc.yaml

# 5. Adopt the shared ambient globals; drop the local declaration file
rm -f src/global.d.ts src/globals.d.ts
#    Add this as the first line of your entry module (src/index.ts):
#      /// <reference path="../node_modules/ha-card-shared/globals.d.ts" />

# 6. Verify, then commit
npm run check:ci && npm run test:coverage
git add -A
git commit -m "chore: migrate to ha-card-shared v1.0.0"
```

Step 5's triple-slash reference must go in the file that uses `__CARD_VERSION__` — it pulls the
ambient type into the bundler's module graph, which a bare `tsconfig` include does not do.

This is a one-time migration. Once it's merged, keep `ha-card-shared` current automatically with
[Dependabot](dependabot.md).

> Future migrations get their own recipe, named `recipe.<from>_<to>.md`
> (e.g. `recipe.1.00_1.10.md`).
