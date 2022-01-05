
# What versions should the package manager choose.

The main requirement I'd like to fulfill is that the package manager always provides a set of dependencies that have been tested together on the current platform.  Note that this includes the compiler as well.

This means when a package is updated, the package manager should not pull down the new version when it's a dependency until it has been tested with the package that is depending on it.  I also think that at a minimum, no package updates will be received unless that package passes its own tests.

# Database Use Case

For now the primary purpose of the package database is to be able to providing the following inputs:

* A package name
    - Note: this probably needs to be namespaced so that multiple publishers
            can create packages with the same name
* A revision (could be "CalendarVersion", a "git SHA" or a custom version string)
* A platform (i.e. linux-x86_64)

and get a list of package dependencies.  The database guarantees that the given package has been tested with the package dpendencies it supplies back.

# Size and Growth of Database

For now I'm just going to use this repo itself as the database.  I don't like this solution long-term because it means that the database will always be growing, never getting smaller.  However, whenever a package is updated, if nobody depends on the old version any longer, then that old revision can be removed from the database.  I think a good solution would be to make a way to "cull" old revisions no longer being used and create a new database that only has the latest information.  The git repository could be used long term to contain the whole history of everything, but the actual database people download could be either be generated from this repo, or maybe this repo only keeps the latest info on the current commit and users can essentialy do a shallow clone to get the latest info only.  There also could be cases where someone wants to build an old version, in which case they could potentially download the whole git repo to access that info.

# Package Data

Each package consists of the following data:

### 1. PackageName (maybe a namespace)
### 2. For each Revision

* A hash of the contents
* Hosts for that version (i.e. github.com urls, archive URLs, etc)
* Git SHA if one of the hosts is a git repo
* Custom Version String
* For each platform
    - The latest revision of PackageName/InterfaceID's that pass the tests for this package

# Current Format

```
PACKAGE_NAME/
|---NAMESPACE/              # a namespace to uniquely identify this package
    |---hosts               # hosts for this package
    |---platforms           # a list of platforms this package should run on
    |---CALENDAR_VERSION/
        |---hash            # a hash of the package contents at this revision
        |---gitsha          # the git SHA of this revision OPTIONAL
        |---customversion   # a custom version string for this revision
        |---hosts           # override the hosts for this revision (i.e. if this revision is on a different branch in a git repo)
        |---platform/       # directory of platform-specific info, each file in this
            |               # directory is named by the platform it applies to and
            |               # contains a list of dependencies known to pass the tests
            |               # on that platform
            |---linux-x86_64
            |---linux-aarch64
            |---macos-x86_64
            |---macos-aarch64
            |---windows-x86_64
            |---freebsd-x86_64
```

# Hash

The `hash` file in each package revision should be a hash of the contents excluding the `.git` folder.  It should include the permissions of each file, filenames are case sensitive.  The `zigpkg` tool will hash a command to perform this hash.  When hashing git repositories we'll also need to resolve/handle problems with git modfiying line endings.

# Hosts

The `hosts` file for a package contains a list of hosts by line that the package can be retrieved from.

```
HOST_TYPE HOST_INFO
archive URL
git URL branch=BRANCH_NAME
```

Note that `HOST_INFO` can contain the following variables in the style of `${VARIABLE}`:

```
CUSTOM_VERSION                  | The customversion of the current package revision.
BUILD_HOST_ZIG_BINARY_PLATFORM  | The host zig platform, i.e. linux-x86_64, windows-x86_64.
BUILD_HOST_ZIG_ARCHIVE_EXT      | The archive extension Zig uses for archives on the build host platform.
```

Examples:

```
archive https://ziglang.org/builds/zig-${BUILD_HOST_ZIG_BINARY_PLATFORM}-${CUSTOM_VERSION}.${BUILD_HOST_ZIG_ARCHIVE_EXT}
```
```
git https://github.com/ziglang/zig
```

# Calendar Version

Example:

```
2057_3_27 (Year 2057, Month 3, Day 27)
```

Note that this version is only granular to the day which implies that packages can only release a new revision once per day. This might seem coarse but with the amount of testing that potentially needs to go into a package update this might be reasonable.

If this is a problem, some options to resolve this are possibly allowing multiple updates per day and taking the latest one, in which case a "Calendar Version" isn't fixed until the end of the day.  We could also add more granularity like the hour or minute into the day, but I'll wait to see if this is even a problem.
