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

Using cabal-install:
```
$ cabal sdist
$ tar xf ./dist/aeson-1.0.0.0.tar.gz
$ cd aeson-1.0.0.0/
$ cabal sandbox init
$ cabal install --enable-tests
$ cabal test
```

Using stack (assuming your `stack.yaml` is also in declared an extra-source-file!):
```
$ stack sdist
$ tar xf .stack-work/**/aeson-1.0.0.0.tar.gz
$ cd aeson-1.0.0.0/
$ stack test
```


