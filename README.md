# featver: feature stability oriented versioning scheme

featver is a [semver](https://semver.org) tooling compatible, [calver](https://calver.org) inspired versioning scheme for software with rapidly changing features.
A featver versioned software commits to these principles:

1. Uses MAJOR.YYMMDD.PATCH as its versioning scheme.
2. Separates the stable and unsupported features in documentation and provides tooling to disable unsupported features.
3. Stable features can be downgraded to unsupported features within a major version if and only if the release removing it is the first release in a year.
4. Unsupported features can be removed within a major version if and only if the release removing it is the first release in a year.
5. Publishes a changelog that documents changes in the stable features.

featver is mostly compatible with semver and semver rules apply to featver too.
The biggest difference is in how the middle part is handled: featver replaces MINOR with YYMMDD.
This means that semver's rules about MINOR version bumps don't apply to featver.

These commitments are voluntary.
But giving such commitment signals that a piece of software deeply cares about its users and wants to ensure its users have a smooth ride through its rapidly changing features.
Gold standard is Go: new Go versions never break older Go versions.
But such standard is impractical for most small software.

Sidenote: if the above too much then even adopting the scheme and a more relaxed "breaking change only at YY boundaries" rule already makes things more predictable.
Do that as the first step.

## Motivation

As of 2024 the established versioning scheme is semver.
But it is very strict: it does not allow evolving software in place.
People who try to evolve software through MINOR changes break others and cause drama.
Or if they don't want that then they would need to keep broken APIs forever around until the next MAJOR update which might come never for stable projects.
Or they just give up on semver altogether and use [zerover](https://0ver.org).
Example drama about semver: [https://github.com/jashkenas/underscore/issues/1805](https://github.com/jashkenas/underscore/issues/1805) and its hackernews discussion at [https://news.ycombinator.com/item?id=8244700](https://news.ycombinator.com/item?id=8244700).

calver tries to answer the problem by giving up semantic versioning completely.
But this is also frustrating for both the software authors and its users: it's unclear what a new version entails.
The versioning scheme is simply a date which cannot differentiate between "this is just a minor update, feel free to update" vs "this is a major update, update with care".
Furthermore [calver.org](https://calver.org) doesn't give exact guidelines on the versioning scheme other than listing some example projects that have a date component in their versioning scheme.
For that there's [chronver.org](https://chronver.org/) but that is incompatible with semver tooling so it doesn't work for Go modules.

featver tries to give a specific, semver-tooling compatible scheme with specific rules that allow in place evolution and also allow communicating large changes requiring manual update.

## Features

A piece of software can be considered as a bag of "features":

- A software library's features could be its API, the classes and methods it exports.
- A command line tool's features could be their command line flags.
- An UI's or website's features could be the buttons and links and their position on the screen.

Whenever such a feature is removed or changed, it creates frustration for the users.
Often these changes are done poorly: the users have no recourse other than migrating their code or workflows right away when an update happens.
On the other hand changes are often necessary.

featver suggests a compromise to alleviate this tension.
Categorize features into two, stable and unsupported, and allow the users to restrict themselves to the stable feature set only.
To remove a stable feature: mark it as unsupported in a year's first release and then remove the next year's first release.
This ensures that breaking changes can happen only once a year and they can only be expected if the YY part of the version string changes.
This way such changes happen in a predictable manner.

Software using featver should make it possible to easily enable and disable the unsupported features.
This toggle ability is featver's key feature, hence its name.
See the examples below how to achieve this technically.

Users preferring stability can opt into the stable mode.
If a stable feature gets downgraded into unsupported then they can switch to the unsupported mode and have a year to update their code and workflows to be stable compatible again.
The changelog should provide instructions how to migrate.
This significantly reduces the stress coming from updates.

All features not marked explicitly as unsupported should be considered as stable features to reduce confusion.

## Versioning scheme

The MAJOR.YYMMDD.PATCH versioning scheme has 3 components:

- MAJOR: This component is the "semantic" part of the version.
  When this version is bumped then users of the software are expected to manually update because this change can contain many breaking changes.
- YYMMDD: This is the date when the release was made.
  The aim of using a date here is to aid the software users clear expectations about the nature of changes between versions.
  If the YY part doesn't change during an update then the update can be considered safe.
  If the YY part does change during an update then then the user can expect to make some changes.
  The changes stemming from updates are more predictable this way solely by looking at the version number change.
- PATCH: This is meant for small bugfixes.
  This should contain no breaking changes in the stable feature set compared to a previous PATCH release in the same MAJOR.YYMMDD version.

## Major versions

The MAJOR part of the version number is manually incremented whenever an incompatible change is being made or otherwise a manual update is desired.

MAJOR version 0 is special though: the feature stability guarantee doesn't apply there.
It's meant to be used for development versions.
This is similar how v0 works in semver.

## Variants

Some systems might impose limits on the version number components such as they have to be less than 65536.
[Browser extension](https://developer.chrome.com/docs/extensions/reference/manifest/version) versions are one example.

In that case a MAJOR.YYMM.PATCH (month resolution) or MAJOR.YYWW.PATCH (week resolution) could work just as well in exchange for a reduced max release cadence.
If going with the week version then use ISO weeks as generated via `date +%g%V` on linux: 2021-01-02 in YYWW format is 2053.

If releases are expected to be rare then a MAJOR.YYR.PATCH could work too.
R is "release", a number that increases by one on each release.
This allows for short version numbers.

## Example: Go modules

Per [https://go.dev/doc/modules/version-numbers](https://go.dev/doc/modules/version-numbers) Go tooling requires the use of semver.
Furthermore MAJOR version bumps are disruptive because they require significant changes: v2+ versions must be in their own subdirectory, see [https://go.dev/blog/v2-go-modules](https://go.dev/blog/v2-go-modules) and [https://go.dev/doc/modules/release-workflow#breaking](https://go.dev/doc/modules/release-workflow#breaking).
This is why a simple date based versioning scheme such as YY.MMDD.PATCH couldn't work for Go.
But featver allows in-place evolution so a Go module could use v1 indefinitely.
A featver Go module breaks Go module versioning commitments so tread carefully.

Put all unsupported functions into files guarded with the `//go:build !stable_features_only` [build constraint](https://pkg.go.dev/cmd/go#hdr-Build_constraints).
This means that `-tags=stable_features_only` builds won't include these files.
The users can then run their nightly stability test with `go test -tags=stable_features_only`.
If a stable feature gets downgraded into unsupported then the users get a warning from their nightly test that they have about a year to fix.
But otherwise they can continue having up to date dependencies, no need to hold back updates and risk being open to bugs and exploits.

## Example: Go modules alternative

A simpler but more disruptive alternative to the above would be to put the unstable functions into a separate package.
E.g. if the main functions are in mypkg then the unstable ones could be in mypkgunstable.
mypkgunstable would also alias all symbols from mypkg.

When a stable function gets deprecated then the user can change their imports to import mypkgunstable under the mypkg alias.
So deprecating functions won't need immediate code changes, only immediate import changes.
When the user refactored their code away from the deprecated function then they can change the import back to the mypkg one (the stable one).

## Example: C library

A C library could implement unsupported features like this:

```
// strlcpy copies src to dst.
// It copies at most size-1 bytes.
// dst will be null terminated.
size_t strlcpy(char *dst, const char *src, size_t size);

#ifndef STABLE_FEATURES_ONLY
// strcpy copies src to dst.
// dst must be large enough to hold src and its null terminator.
// Deprecated: use the safer strlcpy instead.
char *strcpy(char *dst, const char *src);
#endif
```

The user can run their nightly stability test with `-DSTABLE_FEATURES_ONLY=1`.
When `strcpy` gets unsupported (i.e. moved into the `#ifndef` section above) they get a nighthly test failure and have about a year to fix it before `strcpy` gets removed for good.

## Example: C library alternative

Similarly to the Go alternative above the unstable functions could live in a "mypkg-unstable.h" header.
Deprecating a function (moving it from the stable to the unstable section) would create disruptive compilation failures.
But users can quickly get things compiling again after a new release with a single header change and leave the refactoring work to a later point.

## Example: CLI tools

CLI tools should disable all their unsupported features if the STABLE_FEATURES_ONLY environment variable is set.
If a user tries to use an unsupported features when that envvar is set then the tool will tell them to unset it first.
Users can then set this in their .bashrc.
If something breaks after an update then user can remove that flag from their .bashrc and investigate migration when they have some free time.
They won't need to interrupt whatever they were doing and go down a migration rabbit hole at the most inappropriate times.

Note that this is just one example way to implement the featver principles, not necessarily the best.
It is just a demonstration that the featver principles can be implemented in creative ways.

In a more disruptive alternative stable features could be removed right away but still available in a toolname-unstable version of the tool.
Both toolname and toolname-unstable versions would be installed by default so that users can easily switch to the more featureful unstable version that will keep the newly deprecated feature for about a year.

## Example: User interfaces

The user interfaces of desktop and web applications should allow reverting to previous appearances for about a year whenever an UI refresh happens that shuffles the buttons around.
Just have a settings page somewhere where the user can revert and render the old UI if a user's cookie wants the legacy UI.

## Example: underscore

The motivation section mentioned [https://github.com/jashkenas/underscore/issues/1805](https://github.com/jashkenas/underscore/issues/1805).
How would featver resolve that?

The issue was that the `template` function changed:

- [1.6.0](https://cdn.statically.io/gh/jashkenas/underscore/1.6.0/index.html#template): `_.template(templateString, [data], [settings])`
- [1.7.0](https://cdn.statically.io/gh/jashkenas/underscore/1.7.0/index.html#template): `_.template(templateString, [settings])`

Legacy 1.6.0 code:

```
_.template("Using 'with': <%= data.answer %>", {answer: 'no'}, {variable: 'data'});
```

When migrating to 1.7.0 the above code had to be changed to this:

```
_.template("Using 'with': <%= data.answer %>", {variable: 'data'})({answer: 'no'});
```

And this change had to be made at the same time the update happened.
This is annoying and makes updates risky because the local code changes cannot be tested in isolation.

With featver underscore v1.140210.0 (equivalent to [1.6.0](https://underscorejs.org/#1.6.0) would have made the multiargument `template` unsupported and would have added a newly supported `template2(templateString, [settings])` function.
So teams having a nightly test with `underscore-esm-min-stable_features_only.js` would have noticed the incompatibility after the update and could apply the fix of migrating to `template2` when they had time.
Otherwise `template`'s implementation would have remained `_.template(templateString, [data], [settings])` in the normal version.

The breaking change would have been appeared in v1.150219.0 (equivalent to [1.8.0](https://underscorejs.org/#1.8.0)).
In there `_.template(templateString, [data], [settings])` can be changed to `_.template(templateString, [settings])` because the multiargument version was unsupported.
The users had about a year to migrate.
The new 2 argument `template` can be marked as stable.
Furthermore the previously introduced `template2` would be no longer needed so it can be marked as unsupported and then removed in a future release.
Users can migrate back to `template` which should be equivalent to `template2` at this point.

The users have to change their code 2 times.
But both times they have about a year to do it whenever they have time rather part of a large update that might break many other things too.
And the second change is a very trivial function rename.
Due to the relaxed timeframe this would have been a less stressful way to make this change.

## Changelog

Users can learn about feature changes from the changelog.
Keep it simple: just a list of changes for each version.
Put the most important bits at the top.
Perhaps prefix each change with a tag for clarity.
Recommended tags in priority order:

- remove: removes an unsupported feature.
- unsupport: this marks a feature as unsupported which can get removed the next year's first release (no such guarantee on the v0 branch).
- change: this alters behaviour but should not cause any problems during upgrades (except on the v0 branch).
- fix: this fixes a bug, should not cause any problems during upgrades.
- new: this is a new feature, should not cause any problems during upgrades.
- info: this is not a change, just an informational message.

It's important to put the breaking changes to the top and highlight them.
For example in the above referenced underscore example, the `template` related breaking change was mentioned only at the end so it was very easy to miss: [https://underscorejs.org/#1.7.0](https://underscorejs.org/#1.7.0).

## Notes

### How strict are the rules? What if urgent incompatible changes are required by sensitive security issue?

There will be cases sometimes when the compatibility has to broken early due to various issues.
featver should be considered as a set of principles or guidelines to strive for rather than strict rules.
It's OK for pragmatism to prevail occasionally.
This document avoids [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119) level of precision in order to encourage projects to implement or adjust featver in the way it makes most sense for them.

### Is "unsupported" same as "deprecated"?

It's different but sometimes there's overlap.

New features might start out as an unsupported feature and only get the supported label later.
But during that time it would make no sense to mark such new unsupported functions as "deprecated".

It's also possible for a legacy API to be deprecated but still supported.
E.g. in the C example above you could mark `strcpy` as deprecated but still keep it around in the supported feature set.

But if the plan is to delete a feature then in that case it makes sense to mark it as deprecated, then as unsupported, and then actually delete it.

### Are there other similar versioning schemes?

Yes.
In terms of intent [BreakVer](https://www.taoensso.com/break-versioning) comes to closest to featver.
It also acknowledges the impracticality of semver and allows breaking changes in the minor field.

[StableVer](https://gist.github.com/brandonbloom/465625acaf0120354614e7fc0c117c62) is similar too that it separates stable and unstable features and demands smooth transition for breaking changes.
It still restricts most breaking changes to major bumps only though.

For reference [https://lobste.rs/s/mkaigy/solover_is_simple_expressive_versioning#c_2ruerf](https://lobste.rs/s/mkaigy/solover_is_simple_expressive_versioning#c_2ruerf) lists many other versioning schemes.
Not all of them are semver patterned.
featver is semver patterned so that it can be used for Go modules where semver is required.

## Changelog of featver

This is the changelog of this document mostly for the sake of an example and thus a bit exaggerated.
This document is still a draft, will be marked as v1 once a few people reviewed it.
Previous releases: [https://github.com/ypsu/featver/tags](https://github.com/ypsu/featver/tags).

**0.250219.0**

- change: changed the requirements to allow breaking changes only in the year's first release to make everything more predictable.

**0.241105.0:**

- new: expand the Go, C, and CLI examples with an alternative.

**0.240906.0:**

- new: mention more versioning schemes.

**0.240805.0:**

- new: add the underscore example.
- new: add a notes section.
- change: clarify a few things and minor formatting changes.

**0.240804.0:**

- change: mention [zerover](https://0ver.org) and [chronver](https://chronver.org).
- change: add more Go documentation links as reference.
- new: mention the monthly and weekly variants.
- new: highlight that this is a voluntary commitment.
- new: add the UI example.
- fix: linkify links so that it works on github.io too.
- fix: add the github discussions link.
- fix: fix some other small typos.

**0.240803.0:**

- new: finish the draft version.

## Feedback and discussion about featver

See [https://github.com/ypsu/featver/discussions](https://github.com/ypsu/featver/discussions).
Feel free to open new topics.
