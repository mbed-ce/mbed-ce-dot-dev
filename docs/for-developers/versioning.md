# Versioning

## Version Numbers
The first release of Mbed CE is version 7.0.0, to provide an easy way to distinguish Mbed CE from earlier versions of Mbed. 7.0.0 contains the largely new CMake build system, and also makes some breaking changes to the C/C++ code, especially to the SPI/I2C API.

Starting from this release, Mbed CE uses [semantic versioning](https://semver.org/): 

- Any version that contains a breaking change will increment the major version
- Any version that contains new functionality will increment the minor version
- Any version that contains only bugfixes will increment the patch version

A patch version of 99 signifies a development version of Mbed from the `main` branch (this is a slight departure from semantic versioning but it's traditional for Mbed development...).

### Accessing the Version Number

The version number may be read from the `VERSION` file in the mbed-os directory (this is a new addition in Mbed CE).

It is also available in C++ code via the `MBED_MAJOR_VERSION`, `MBED_MINOR_VERSION`, and `MBED_PATCH_VERSION` macros after including `mbed_version.h` or `mbed.h` (this way is compatible with ARM Mbed). If you wish to check for a certain minimum Mbed version, you can check for that with code like the following:

```cpp
#include <mbed.h>

#if MBED_VERSION < MBED_ENCODE_VERSION(7, 1, 0)
#error "Mbed version is too old!"
#endif
```

## Branches
The following branches are maintained by the Mbed CE maintainers:

- `main`: Development version of Mbed, where day to day development happens. Version number on this branch will always be `x.y.99`.
- `release`: Latest release of Mbed can be found here
- `release/<n>.x` (where `<n>` is a major release version of Mbed CE)

The `master` branch was previously the only active branch and was used on ARM Mbed, but is being retired with this release process.

## Release Process

### Criteria for Release
Generally, major and minor releases will be made when there are new features that have been implemented that we wish to make available to users, and patch releases will be made when there are significant bugfixes that need to be published. You can probably expect major/minor releases to be made approximately a few times a year, though this depends on development pace. 

Mbed CE releases do not involve any specific additional testing: our goal is to make all of our tests continuous, so that they run on every merge to `main`. However, releases do mean that all changes between the previous release and this release must be documented properly in the `CHANGELOG.md` file to assist those upgrading to the new release in understanding what has changed. Also, releases can only be made when there are no outstanding issues introduced since the last release that might cause problems for users of Mbed. For example, if a large change gets split into multiple PRs, only one of those PRs has been merged, and some functionality is broken until the next one is merged, a release could not happen until the remaining PRs are merged.

### Standard Release Process
This release process is used to create a new major or minor release of Mbed. It's also used for bugfix releases if there are no changes on `main` that would cause a major or minor release to be needed.

1. Push a new commit to `main` that changes the `VERSION` file to contain the desired version number and updates the `CHANGELOG.md` file to show the new release.
2. Force-push the `release` branch to be equivalent to the `main` branch
3. Push a new tag to the repository based on head of `main` called `mbed-os-x.y.z`, where `x.y.z` is the version number being released. GitHub Actions automation will then create a GitHub release with the content from the changelog.
4. Push a new PlatformIO release by manually checking out `release` and running the `tools/make_pio_release.sh` script
5. Push a new commit to `main` that changes the `VERSION` file back to `x.y.99`.

### Cherry-Pick Bugfix Release Process
This release process is used if we commit a new bugfix to `main` and wish to release it immediately without releasing other changes on `main` (e.g. because `main` is not in a good state for release).

1. Cherry-pick the commit or commits that should be released onto the `release` branch and push them.
2. Push another commit updating the `CHANGELOG.md` file on `release` to contain the new bugfix release with the appropriate changes, and also updating `VERSION` with the new version number.
3. Push a new tag to the repository based on head of `release` called `mbed-os-x.y.z`, where `x.y.z` is the version number being released. GitHub Actions automation will then create a GitHub release with the content from the changelog.
4. Push a new PlatformIO release by manually checking out `release` and running the `tools/make_pio_release.sh` script
5. Finally, push a new commit to `main` that updates `CHANGELOG.md` with the new contents added in `release`, and removes changes from the in-progress release section that have now been released.

### Backport Releases
We may be in a situation where we want to release small changes or bugfixes for a previous release after it has been superseded by a new major release. This would be the case, for example, if we release version 8.0, and then find a serious bug in the latest 7.y.z release that needs a patch. Under this situation we could not use the process above because the `release` branch would already contain an 8.x release.

To fix this situation, we would create a new branch called `release/7.x` that is equal to `release` at the time of the last `7.x` release. Then, we would perform the cherry-pick release process above to cherry-pick the bugfix(es) onto this branch and release them.
