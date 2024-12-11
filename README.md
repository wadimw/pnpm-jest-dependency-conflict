# pnpm-jest-dependency-conflict

This repository is a minimal reproduction of a dependency conflict between different `jest` versions within a `pnpm` workspace.

## Reproduction

This repository contains a minimal reproduction of the issue described in sections below.

There are two packages in the workspace:

-   `jest-26` - has a `devDependency` on `jest@26`
-   `jest-29` - has a `devDependency` on `jest@29`

These dependencies should not conflict, because `pnpm` should install them in separate directories (`packages/jest-26/node_modules` and `packages/jest-29/node_modules`).

Each package contains two scripts:

-   `test:bin` - runs `jest` through shell wrapper provided by `pnpm` in `node_modules/.bin/jest`
-   `test:node` - runs `jest` through `node` using `node_modules/jest/bin/jest.js`

In order to reproduce the issue, run the following commands:

```sh
pnpm install
pnpm -F jest-26 test:bin # passes
pnpm -F jest-26 test:node # passes
pnpm -F jest-29 test:bin # fails
pnpm -F jest-29 test:node # passes
```

Running `jest` through shell wrapper in `jest-29` package causes the following error:

```log
➜  pnpm-jest-dependency-conflict git:(main) ✗ pnpm -F jest-29 test:bin

> jest-29@1.0.0 test:bin /Users/wwawrzenczak/Desktop/pnpm-jest-dependency-conflict/packages/jest-29
> LANG=en_US.UTF8 NODE_OPTIONS="$NODE_OPTIONS --experimental-vm-modules" jest --testEnvironment node

 FAIL  test/index.test.js
  ● Test suite failed to run

    Must use import to load ES Module: /Users/wwawrzenczak/Desktop/pnpm-jest-dependency-conflict/node_modules/.pnpm/yaml@2.6.1/node_modules/yaml/browser/index.js

      at Runtime.requireModule (../../node_modules/.pnpm/jest-runtime@29.7.0/node_modules/jest-runtime/build/index.js:850:21)

Test Suites: 1 failed, 1 total
Tests:       0 total
Snapshots:   0 total
Time:        0.197 s
Ran all test suites.
/Users/wwawrzenczak/Desktop/pnpm-jest-dependency-conflict/packages/jest-29:
 ERR_PNPM_RECURSIVE_RUN_FIRST_FAIL  jest-29@1.0.0 test:bin: `LANG=en_US.UTF8 NODE_OPTIONS="$NODE_OPTIONS --experimental-vm-modules" jest --testEnvironment node`
Exit status 1
```

however, this error does not appear in `jest-26` package, nor when running `jest` directly (using `node_modules/jest/bin/jest.js` rather than `node_modules/.bin/jest`).

## Cause

We have discovered that this behaviour is caused by `jest` loading incorrect version of `jest-environment-node` (i.e. `jest@29.7.0` loads `jest-environment-node@26.6.2`) when using shell wrapper `node_modules/.bin/jest` provided by `pnpm`.

## Original issue description

Issue was encountered in [`ilib-mono#13`](https://github.com/iLib-js/ilib-mono/pull/13) during migration of package [`ilib-loctool-tap-i18n@7585e97`](https://github.com/iLib-js/ilib-loctool-tap-i18n/tree/7585e97497e16475bfce1fc034caf0c7716229e1) into the newly created monorepo.

The issue was encountered when running tests on `tap-i18n` package from within the monorepo managed by `pnpm`.

The following steps can be taken to reproduce the original issue:

```sh
git clone https://github.com/iLib-js/ilib-mono.git
cd ilib-mono
git checkout db750b1~1 # check out code before the workaround commit https://github.com/iLib-js/ilib-mono/pull/13/commits/db750b13424cf881ace9967fadc3fff9ea8f70d5
nvm use 20
corepack enable pnpm
pnpm install
pnpm -F ilib-loctool-tap-i18n exec jest --version # returns 29.7.0
pnpm -F ilib-loctool-tap-i18n test
```

This results in the following error:

```log
➜  ilib-mono git:(12ef72a0) ✗ pn -F ilib-loctool-tap-i18n test

> ilib-loctool-tap-i18n@1.1.1 test /Users/wwawrzenczak/Development/ilib-mono/packages/ilib-loctool-tap-i18n
> pnpm test:jest


> ilib-loctool-tap-i18n@1.1.1 test:jest /Users/wwawrzenczak/Development/ilib-mono/packages/ilib-loctool-tap-i18n
> LANG=en_US.UTF8 NODE_OPTIONS="$NODE_OPTIONS --experimental-vm-modules" jest --testEnvironment node

 FAIL  test/YamlFileType.test.js
  ● Test suite failed to run

    Must use import to load ES Module: /Users/wwawrzenczak/Development/ilib-mono/node_modules/.pnpm/yaml@2.6.1/node_modules/yaml/browser/index.js

      20 | var fs = require("fs");
      21 | var path = require("path");
    > 22 | var Yaml = require("ilib-yaml");
         |            ^
      23 | var Locale = require("ilib/lib/Locale.js");
      24 | var tools = require("ilib-tools-common");
      25 | var TranslationSet = require("loctool/lib/TranslationSet");

      at Runtime.requireModule (../../node_modules/.pnpm/jest-runtime@29.7.0/node_modules/jest-runtime/build/index.js:850:21)
      at Object.<anonymous> (../../node_modules/.pnpm/ilib-yaml@1.0.2/node_modules/ilib-yaml/src/index.js:21:83)
      at Object.require (YamlFile.js:22:12)
      at Object.require (YamlFileType.js:22:16)
      at Object.require (test/YamlFileType.test.js:20:24)

 FAIL  test/YamlFile.test.js
  ● Test suite failed to run

    Must use import to load ES Module: /Users/wwawrzenczak/Development/ilib-mono/node_modules/.pnpm/yaml@2.6.1/node_modules/yaml/browser/index.js

      20 | var fs = require("fs");
      21 | var path = require("path");
    > 22 | var Yaml = require("ilib-yaml");
         |            ^
      23 | var Locale = require("ilib/lib/Locale.js");
      24 | var tools = require("ilib-tools-common");
      25 | var TranslationSet = require("loctool/lib/TranslationSet");

      at Runtime.requireModule (../../node_modules/.pnpm/jest-runtime@29.7.0/node_modules/jest-runtime/build/index.js:850:21)
      at Object.<anonymous> (../../node_modules/.pnpm/ilib-yaml@1.0.2/node_modules/ilib-yaml/src/index.js:21:83)
      at Object.require (YamlFile.js:22:12)
      at Object.require (test/YamlFile.test.js:20:16)

Test Suites: 2 failed, 2 total
Tests:       0 total
Snapshots:   0 total
Time:        0.54 s
Ran all test suites.
 ELIFECYCLE  Command failed with exit code 1.
/Users/wwawrzenczak/Development/ilib-mono/packages/ilib-loctool-tap-i18n:
 ERR_PNPM_RECURSIVE_RUN_FIRST_FAIL  ilib-loctool-tap-i18n@1.1.1 test: `pnpm test:jest`
Exit status 1
```

This issue is not present when tests are run directly on `tap-i18n` package (standalone, prior to merging into the monorepo). Note that this package used `npm` and that with Node 20 `conditional-install` script resolved to `"jest": "^29.0.0"`.

```sh
git clone https://github.com/iLib-js/ilib-loctool-tap-i18n.git
cd ilib-loctool-tap-i18n
git checkout 7585e97 # commit that was merged into the monorepo https://github.com/iLib-js/ilib-loctool-tap-i18n/tree/7585e97497e16475bfce1fc034caf0c7716229e1
nvm use 20
npm install
node_modules/.bin/jest --version # returns 29.7.0
npm test # runs tests successfully
```

## Investigation

This seemed to be unrelated to our own package `ilib-yaml`, since the error was thrown during require of its dependency `yaml`.

Using a debugger to inspect method from stack trace `requireModule(from, moduleName, options, isRequireActual = false)` in `jest-runtime@29.7.0` package ([`./node_modules/.pnpm/jest-runtime@29.7.0/node_modules/jest-runtime/build/index.js`](./node_modules/.pnpm/jest-runtime@29.7.0/node_modules/jest-runtime/build/index.js)) we eventually found that a call to `this._resolveCjsModule(from, moduleName)` on line 846 returned different paths in `pnpm` and `npm` environments:

-   when run in `pnpm` (monorepo) it returned `/Users/wwawrzenczak/Development/ilib-mono/node_modules/.pnpm/yaml@2.6.1/node_modules/yaml/browser/index.js`
-   when run in `npm` (standalone) it returned `/Users/wwawrzenczak/Desktop/ilib-loctool-tap-i18n/node_modules/yaml/dist/index.js`

i.e. rather than using `.exports.['.'].node`, it used `.exports.['.'].default` when loading the `yaml` package ([yaml package.json reference](https://github.com/eemeli/yaml/blob/715e8eddf2611363cb16fa34fe04dbbc3280993e/package.json#L30-L31)).

This behaviour in Jest is controlled by setting `customExportConditions` which should be specified by [Jest environment options](https://jestjs.io/docs/configuration#testenvironmentoptions-object) - `jest-environment-node` should set this by default to `['node', 'node-addons']` in order for Jest to resolve using `.node`. Actual values passed at runtime to module resolver come from `this.cjsConditions` which can be seen in the definition of `_resolveCjsModule` in [`./node_modules/.pnpm/jest-runtime@29.7.0/node_modules/jest-runtime/build/index.js`](./node_modules/.pnpm/jest-runtime@29.7.0/node_modules/jest-runtime/build/index.js) line 1277. This value in turn originates from `this._environment.exportConditions()`. Method `exportConditions()` is defined in `jest-environment-node@29.7.0` package ([`./node_modules/.pnpm/jest-environment-node@29.7.0/node_modules/jest-environment-node/build/index.js`](./node_modules/.pnpm/jest-environment-node@29.7.0/node_modules/jest-environment-node/build/index.js)) line 203.

It turned out that method `this._environment.exportConditions()` was `undefined` when run from `pnpm`. Placing a breakpoint in jest Runtime constructor and going up the stack trace we found that the object passed to `environment` argument was an instance of class loaded dynamically in [`node_modules/.pnpm/jest-runner@29.7.0/node_modules/jest-runner/build/runTest.js`](./node_modules/.pnpm/jest-runner@29.7.0/node_modules/jest-runner/build/runTest.js) line 193 - however, `testEnvironment` had the following value:

```
/Users/wwawrzenczak/Development/ilib-mono/node_modules/.pnpm/jest-environment-node@26.6.2/node_modules/jest-environment-node/build/index.js
```

This was the cause of the issue - `jest-runtime@29.7.0` was using an instance of `jest-environment-node@26.6.2`, which did not define method `exportConditions()`, hence it was only using the default `this.cjsConditions = [ "require", "default" ]` and because of that it did not know that it should use export from `.node`.

Jest v26 was also installed in the monorepo at the time, because another monorepo package had a `devDependency` on it. However, note that `jest --version` correctly returned `29.7.0` (and debugger was showing that it's running from `node_modules/.pnpm/jest-runtime@29.7.0`), but another dependency of that package was resolved incorrectly to `node_modules/.pnpm/jest-environment-node@26.6.2`.

## Workaround

Google led us to the following comment with a workaround:

https://github.com/pnpm/pnpm/issues/5176#issuecomment-1262323891

i.e. ditching shell bin wrapper provided by `pnpm` and running `jest` through `node` directly from `node_modules`:

```sh
node --experimental-vm-modules node_modules/jest/bin/jest.js"
```

https://github.com/iLib-js/ilib-mono/pull/13/commits/db750b13424cf881ace9967fadc3fff9ea8f70d5

this resolved the issue and tests ran successfully.

Note that we did not want to change hoisting, because it would prevent us from having different versions of `jest` in different packages.
