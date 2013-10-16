# Why I'm starting to like upper bounds enough to write a blog post about it


**Disclaimer**: I'm using Roman's packages as an example here, and I also argue based on his Against-PVP blog post. I don't mean to point fingers, I just thought it would be easier to explain this using a concrete example. This whole post turns into a minor annoyance in comparison to the great work he has done on the haskell-suite.


I've been thinking a lot about the PVP lately, Fay doesn't have upper bounds but I'll probably add them soon. I used to think it was too much work maintaining upper bounds and I can only think of two or three cases where this has been a problem for Fay which has a fair amount of dependencies. That was me as a package maintainer. Working at Silk for a year I've probably spent as much time maintaining packages as I have been using third party packages and I've slowly started to change my mind.



## Case in point

A few days ago I got a pull request from Michael Snoyman because optparse-applicative 0.6 was released and didn't build against Fay. I would say sorry for breaking his build, but I think it's more important that Fay builds against the latest version for him (correct me if I'm wrong) so he would probably have sent the PR anyway. He chose to put in some CPP to support both versions of optparse-applicative. I would probably just have bumped the dependency myself but I certainly don't mind CPP [when it's as unintrusive as this](https://github.com/faylang/fay/commit/4bfdb439bf238ce3b1def42aa998477699d919ab).

Worked fine for me locally, [but the travis build failed](https://travis-ci.org/faylang/fay/builds/12510902).


For the really adventorous, try figuring out how fay should be fixed before reading on (no cheating by looking at the other builds!)


Here are the key pieces of information:

* Fay depends on `haskell-packages -any` and `optparse-applicative >= 0.5`
* Travis CI runs Haskell Platform-2012.2.0.0, containing mtl-2.0.1.0
* haskell-packages-0.2.3.1 depends on `mtl >= 2.1` and `opt-parse-applicative >= 0.6` (bumped dependency, instead of cpp)
* haskell-packages-0.2.1 depends on `mtl -any` and `optparse-applicative >= 0.5.1`


For the mildly adventurous, can you now see what goes wrong?


### Answer


There are now two possible install plans that Cabal might pick when installing Fay:
 * `haskell-packages-0.2.3.1`, `optparse-applicative-0.6`, `mtl-2.1.2`
 * `haskell-packages-0.2.1`, `optparse-applicative-0.6`, `mtl-2.0.1.0`

N.B. If you looked closely there are more possible build plans, but they are irrelevant in this case.


The first build plan is good. It contains the latest versions of everything. haskell-packages 0.2.3.1 was released to add support for optparse-applicative-0.6.

The second one is bad. The build fails because the chosen versions of haskell-packages and optparse-applicative are incompatible.


My first attempt to fix this was to bump Fay to require `optparse-applicative >= 0.6` and `haskell-packages >= 0.2.3.1`. [It didn't work either](https://travis-ci.org/faylang/fay/builds/12588296) because it breaks other packages depending on the old mtl. `--force-reinstall`'ing it worked though.

I released this today as fay-0.18.0.1. But it hasn't solved the problem completely. Everyone using an old mtl and installing fay will get a compile error, not a dependency resolution error. This is when a lot of people will `rm -rf ~/.ghc/`, but that won't fix the problem here since mtl is in the platform. End-users of my package will have to spend time figuring out that there is a new version of haskell-packages out which they can use.

Which brings me to my first point:


## Not having upper bounds shifts a disproportionate amount of workload to the direct and transitive dependents of your package.

haskell-packages was bumped to 0.2.3.1 to support the new version of optparse-applicative. But the lack of upper bounds on earlier versions means the newest version won't necessarily be picked, and a build failure will occur during compilation instead of during dependency resolution.

If it had upper bounds, I would immediately have seen the problem on Travis. Instead of an older version of haskell-packages being chosen and generating the compile error it would have failed because mtl's dependents break, with the actual fix suggested: `--force-reinstalls`. If mtl had no dependents installation would simply succeed! Instead I had to figure out why it was picked, which wasn't immediately clear.

If both Fay and haskell-packages had upper bounds, Michael would still have sent the pull request and I would have noticed that I needed to bump haskell-packages in the process. That would have been quick. Instead every haskell-packages user who runs across this problem will have to redo all the investigation I did. I can't expect users of Fay to be as diligent and figure this out by themselves.


## Upper bounds don't allow accidents

I draw the opposite conclusion from "Reason 1" in [Why PVP doesn't work](http://ro-che.info/articles/2013-10-05-why-pvp-doesnt-work.html). To sum up the argument:

"package-a depends on containers ==0.4.*, package-b depends on containers ==0.5.* [...] These packages cannot be built together. [...] Having just the lower bounds is much better — if each one of them is satisfiable, then they all can be satisfied simultaneously (e.g. by the latest version)."

I think the upper bound is a good thing here. I see three possibilities, the author of package-a...

* may know it to be incompatible with containers-0.5. It would be a lie to remove the upper bound, and wouldn't help anyone. I assume this wasn't the case that was argued for.
* hasn't tested it with containers-0.5, but it turns out it works. He can simply bump the upper bound and release a new version that works with both versions. This is minimal amount of work for him, and equally easy for a dependent to do as well.
* hasn't tested it with containers-0.5, and it turns out it breaks. He needs to figure out how to fix it and either try to support both or bump the lower bound. This may be hard, but he will probably be notified of the problem quickly (by Stackage, users, or by keeping an eye on his own package's dependencies)

The person closest to and most knowledgeable about the problem can test and possibly fix new versions the quickest, upper bounds will make it obvious which maintainer that is. This probably won't cause the need for a chain of package bumps since bumping your dependencies only results in a minor bump for your package, but even for the cases when it does you retain the locality of the issue at hand.



## You **can** nail it down

The second argument in "Why PVP doesn't work" is:

"In theory, you pin down every dependency and in five years from now it’s going to build in the same way as it just did today."

It will, assuming you are running the same version of GHC. It won't if packages don't have upper bounds. Every dependency that doesn't have an upper bound will cause issues. Putting upper bounds on base is a good way to tell your users about what version of GHC you have tested the library against.

"In practice, people would want to build your package with modern compilers, on modern operating systems, and with external libraries provided by their distributions."

This will only occur when someone upgrades their GHC. If the package maintainer also does this he will fix the problem.

"Every maintained package which has this problem would sooner or later get the fix, but probably only the latest major version would be fixed."

I don't see this being an issue. A user is upgrading his packages that are dependents of yours, but he doesn't want to deal with upgrading his own dependencies? If he doesn't want to, it should be fine if there are upper bounds, he won't accidentally use other packages that are incompatible with yours. Again, if he doesn't have upper bounds his package might simply break.


## Use version bounds for stability

Summing up what I wrote earlier, a version bound can mean two things: Your package is incompatible with a range, or the compatibility is unknown.

The only case where this is a problem is when the bound accidentally turns out to be unnecessary. This is easy to test and fix for everyone. It results in a minor bump that won't affect dependent libraries, but may require a `--force-reinstalls` for end users.

Not having upper bounds doesn't tell the user anything. It may or may not be tested, it may or may not work. If it turns out it doesn't work it's much harder for users to spot the problem. Once it's fixed a new version will be uploaded but the old version is still broken for this combination of dependencies, meaning several users may spend time trying to decipher the problem because of how the dependencies were resolved. This will noteably occur if someone does a cabal update before the dependent package has been updated. His build will fail, and he has to check whether the issue has been reported already (This has happened to me numerous times.)


## Hackage2 edits of .cabal files

Hackage2 adds support for revisions of .cabal files where maintainers can edit version bounds of packages without having to upload a new version. This is currently disabled because there isn't enough tooling to support it properly. [See this github issue](https://github.com/haskell/hackage-server/issues/52). It would be equally advantageous to both camps. Pro-PVPers can relax their bounds when necessary, and Against-PVPers can constrain them. This would make the argument that packages without upper bounds end up broken moot, but that's only solves one of my issues regarding lack of upper bounds.



## THE DRAMATIC CONCLUSION

When I asked Erik Hesselink about why you should honor the PVP he (jokingly) said "Because Simon Marlow wrote it!" Hats off to Simon, but my point is not to argue whether he made the right design choice. My point would be that he made **a** choice. People are allowed to disagree and not specify bounds, but the result is that everyone is annoyed. Pro-PVPers complain that the against-PVPers' old package versions break. Against-PVPers complain that the pro-PVPers' packages end up working.

It's probably not going to happen, but if we can come to an agreement, no matter which one it is, we'll be better off than we currently are. But until then, can't we stick with what was already agreed upon?





## Side note

I feel the need to strongly advocate [Stackage](https://github.com/fpco/stackage). It works extremely well with upper bounds. You will automatically[1] get notified when you need to bump a package, they do regular builds on multiple versions of GHC/Haskell Platform, so even if you are deep into the dependency chain, assuming everyone is on Stackage it will be a short amount of time before the changes have trickled down to the leaves.


[1] I'm not sure how automated this process really is, I know Michael Snoyman is very active maintaining the project (thank you Michael!) I hope this will require much less supervision in the future.
