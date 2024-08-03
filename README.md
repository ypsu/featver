# featver: feature stability oriented versioning scheme

featver is [semver](https://semver.org) tooling compatible, [calver](https://calver.org) inspired versioning scheme for software with rapidly changing features.
featver versioned software promises to follow these principles:

1. Uses MAJOR.YYMMDD.PATCH as its versioning scheme.
2. Separates the stable and unsupported features in documentation and provides tooling to disable unsupported features.
3. Stable features remain available for at least 6 months before removal within a MAJOR version.
4. Publishes a changelog that documents changes in the stable features.

featver is mostly compatible with semver and semver rules apply to featver too.
The biggest difference is in how the middle part is handled: featver replaces MINOR with YYMMDD.
This means that semver's rules about MINOR version bumps don't apply to featver.

## Motivation

As of 2024 the established versioning scheme is semver.
But its very strict: it does not allow evolving software in place.
People who try to evolve software through MINOR changes break others and cause drama.
Or if they don't want that then they would need to keep broken APIs forever around until the next MAJOR update which might come never for stable projects.
Example drama: https://github.com/jashkenas/underscore/issues/1805 and its hackernews discussion at https://news.ycombinator.com/item?id=8244700.

calver tries to answer the problem by giving up semantic versioning completely.
But this is also frustrating for both the software authors and its users: it's unclear what a new version entails.
The versioning scheme is simply a date which cannot differentiate between "this is just a minor update, feel free to update" vs "this is a major update, update with care".
Furthermore https://calver.org doesn't give exact guidelines on the versioning scheme other than listing some example projects that have a date component in their versioning scheme.

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
To remove a stable feature: mark it as unsupported and remove it 6 months or later.

Make it possible to easily enable and disable the unsupported features.
This toggle ability is featver's key feature, hence its name.
See the examples below how to achieve this technically.

Users preferring stability can opt into the stable mode.
If a stable feature gets downgraded into unsupported then they can switch to the unsupported mode and have 6 months to update their code and workflows to be stable compatible again.
The changelog should provide instructions how to migrate.
This significantly reduces the stress coming from updates.

All features not marked explicitly as unsupported should be considered as stable features to reduce confusion.

## Versioning scheme

The MAJOR.YYMMDD.PATCH versioning scheme has 3 components:

- MAJOR: This component is the "semantic" part of the version.
  When this version is bumped then users of the software are expected to manually update because this change can contain many breaking changes.
- YYMMDD: This is the date when the release was made.
  The aim of using a date here is to aid the software users estimate the amount of changes between versions.
  If the date between two versions is less than 6 months, the update can be considered safe and any small incompatibility should be easily resolvable per instructions in the changelog.
- PATCH: This is meant for small bugfixes.
  This should contain no breaking changes in the stable feature set compared to a previous PATCH release in the same MAJOR.YYMMDD version.

## Major versions

The MAJOR part of the version number is manually incremented whenever an incompatible change is being made or otherwise a manual update is desired.

MAJOR version is 0 is special though: the 6 month compatibility guarantee doesn't apply there.
It's meant to be used for development versions.
It might make sense for the v0 branch to not follow YYMMDD for the middle part of the version but something like YYMMDDHHMMSS.
E.g. a continuous build tool might be making v0 versions after each commit and then a weekly cronjob just stamps a v1 version tag on the latest green v0 version.
This scheme allows catering to users desiring rapid releases (they can follow v0) and to users desiring infrequent but stable releases (they can follow v1).

## Example: Go modules

Go tooling requires the use of semver but featver is compatible with that.
In Go MAJOR version bumps require significant changes: v2+ versions must be in their own subdirectory, see https://go.dev/blog/v2-go-modules.
This is why a simple date based versioning scheme such as YY.MMDD.PATCH couldn't work for Go.
But featver allows in-place evolution so a Go module could use v1 indefinitely.

Put all unsupported functions into files guarded with the `//go:build !stable_features_only` [build constraint](https://pkg.go.dev/cmd/go#hdr-Build_constraints).
This means that `-tags=stable_features_only` builds won't include these files.
The users can then run their nightly stability test with `go test -tags=stable_features_only`.
If a stable feature gets downgraded into unsupported then the users get a warning from their nightly test that they have 6 months to fix.
But otherwise they can continue having up to date dependencies, no need to hold back updates and risk being open to bugs and exploits.

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
When `strcpy` gets unsupported (i.e. moved into the `#ifndef` section above) they get a nighthly test failure and have 6 months to fix it before `strcpy` gets removed for good.

## Example: CLI tools

CLI tools should disable all their unsupported features if the STABLE_FEATURES_ONLY environment variable is set.
If a user tries to use an unsupported features when that envvar is set then the tool will tell them to unset it first.
Users can then set this in their .bashrc.
If something breaks after an update then user can remove that flag from their .bashrc and investigate migration when they have some free time.
They won't need to interrupt whatever they were doing and go down a migration rabbit hole at the most inappropriate times.

## Changelog

Users can learn about feature changes from the changelog.
Keep it simple: just a list of changes for each version.
Put the most important bits at the top.
Perhaps prefix each change with a tag for clarity.
Recommended tags in priority order:

- remove: removes an unsupported feature.
- unsupport: this marks a feature as unsupported which can get removed 6 months later (no such guarantee on the v0 branch).
- change: this alters behaviour but should not cause any problems during upgrades (except on the v0 branch).
- fix: this fixes a bug, should not cause any problems during upgrades.
- new: this is a new feature, should not cause any problems during upgrades.
- info: this is not a change, just an informational message.

## Changelog of featver

This is the changelog of this document mostly for the sake of an example.
This document is still a draft, will be marked as v1 once a few people reviewed it.

**0.240803.1 [pending]:**

- fix: add the github discussions link.

**0.240803.0:**

- new: finish the draft version.

## Feedback and discussion about featver

See https://github.com/ypsu/featver/discussions.
Feel free to open new topics.
