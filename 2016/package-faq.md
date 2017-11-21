# Package FAQ

Here's are some issues that I commonly see while curating hackage and stackage.

## Test files can't be found

```
runtests: [...]/tests/skein_golden_kat.txt: openFile: does not exist (No such file or directory)
```

A test file is missing but the tests run fine in your checkout of the
project.

This happens for others because they don't have a source checkout,
instead they are running tests from the tarball that was uploaded to
hackage.

### Reproducing

Using cabal-install:
```
$ cabal sdist
$ tar xf ./dist/aeson-1.0.0.0.tar.gz
$ cd aeson-1.0.0.0/
$ cabal sandbox init
$ cabal install --enable-tests
$ cabal test
```

Using stack:
```
$ stack sdist
$ tar xf .stack-work/**/aeson-1.0.0.0.tar.gz
$ cd aeson-1.0.0.0/
$ stack init
$ stack test
```

### Solution

Add the missing files as `extra-source-files`.

Using `extra-source-files` the files will be added to the tarball that's created when you run `cabal sdist`, `stack sdist`, or `stack upload`. You can then use file paths relative to the project's root as you probably already are.
```
extra-source-files:
  tests/*.txt
  tests/animage.jpg
```

### Tips

* Using wildcards makes it less likely that you forget to add new files.

* Stack warns about files it thinks should be part of the distribution.

* To test if the tarball contains everything it needs you need to install everything in a directory separate from your source checkout.


## Ambiguous modules when using doctest

Upstream issue for doctest: https://github.com/sol/doctest/issues/119

```
src/Language/Nix/PrettyPrinting.hs:23:1: error:
    Ambiguous interface for ‘Text.PrettyPrint.HughesPJClass’:
      it was found in multiple packages:
      pretty-1.1.3.3 pretty-class-1.0.1.1
```

This doesn't mean that your package is broken. The reason it happens
is that doctest looks for packages in the local package database
instead of looking at declared build-depends. If you import a module
that also happens to be defined in another installed package doctest
doesn't know which package to get the module from and this error
occurs.

When developing your package in isolation you are unlikely to be
affected by this. In the example above the package depends on
`pretty`, but when `pretty-class` is installed as well this problem
occurs.

### Reproducing

Install all the mentioned packages together in one sandbox/package
database and run the tests after that.

```
$ cabal install pretty pretty-class my-package --enable-tests
$ cabal test my-package
```

### Solution

**Update:** For stackage there is now an option called `hide` in build-constraints.yaml that is used to hide packages that have overlapping module names (on a first-come first-serve basis). This only works if the hidden packages has no dependents.

Currently there is no good solution to this situation. You can accept that the
problem can occur since it's unlikely that this would hide any
bugs. The other solution is to use `-XPackageImports` to qualify which
package to get the module from.

```haskell
{-# LANGUAGE PackageImports #-}
module Language.Nix.PrettyPrinting where

import "pretty" Text.PrettyPrint.HughesPJClass
```

For packages that also have non-doctest tests you can use a cabal flag to disable just doctests.

```
flag run-doctests
  description:       Run doctests
  manual:            True
  default:           False

test-suite my-doctests
  if !flag(run-doctests)
    buildable: False
```

## Using hlint in test suites

If you intend to submit a package with a hlint test suite to stackage, please place that test suite under a disabled-by-default manual flag to prevent new versions of hlint from breaking the stackage build. If you want to get notified when new hlint releases add new warnings to your package you can set up a CI cron job instead.

https://www.snoyman.com/blog/2017/11/future-proofing-test-suites

