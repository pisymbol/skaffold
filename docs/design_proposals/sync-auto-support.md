# Auto sync support design doc

* Author(s): Appu Goundan (@loosebazooka)
* Design Shepherds: Tejal Desai (@tejal29), Balint Pato (@balopat)
* Date: 09/17/2019
* Status: Implementation in progress

## Background

Currently skaffold does not support `sync` for files that are generated
during a build. For example when syncing java files to a container, one
would normally expect `.class` files to be sync'd, but skaffold is
really only aware of `.java` files in a build.

1. Why is this required?
  - We would like to support the following features
      - Skaffold can sync files that are not watched (these files are not
        considered inputs for the container builder). Motivating example: user
        compiles java class files outside of the container _manually_, while
        Spring Boot DevTools is running inside the container. The class files
        would be picked up by Skaffold and copied over, and the app would pick
        up the changes.
      - Skaffold can sync files that are generated by a buildscript. Motivating
        example: user changes java files, Skaffold watchers notices this, runs the
        buildscript that generates class files. Since class files are marked 'generated'
        synacbles, they are synced to the container, where Spring Boot DevTools
        picks up the change.
      - And one or both of the following:
          - Skaffold can be configured to run a custom script to when certain files change.
            Motivating example. Jib by default will run a full container build
            on "build", but we only want to generate intermediate assets (class
            files, etc) when doing a sync build.
          - Skaffold can sync an externally generated tar file to a remote container (see
            [alternative design for builder using tar](#delegate-generation-of-sync-tar-to-builder))

2. If this is a redesign, what are the drawbacks of the current implementation?
  - This is not a redesign, but a new sync mode that may or may not be available
    in the skaffold.yaml directly to users. It could simply be an internal API
    that is usable by builders like Jib.
  - This is similar to `_smart_` described in [sync-improvements](sync-improvements.md)

3. Is there any another workaround, and if so, what are its drawbacks?
  - Currently one can create a docker file that only copies specific build
    results into the container and relies on a local build to generate those
    intermediate artifacts. This requires a user to trigger a first build
    manually before docker kicks in to containerize the application. While
    it may be possible to automate this (for example: gradle --continuous), it
    is not an acceptable solution to require manual external processes for
    a build to succeed. Covered in [Hack it](#hack-it)

4. Mention related issues, if there are any.
  - This is not trying to solve the problem of dealing with a multistage
    dockerbuild. Intermediate build artifacts might still be possible to
    determine, however that would require an extra mechanism to do a partial
    docker build and sync files from a built container to a running container --
    something we do not intend to cover here.

#### Problems with current API/config
The current `sync` system has the following problems:
1. No way to trigger local out-of-container processes - for example, in jib, to build a
   container, one would run `./gradlew jib`, but to update class files so the
   system may sync them, one would only be required to run `./gradlew classes`.
2. No way to tell the system to sync non build inputs. Skaffold is
   normally only watching `.java` files for a build, in the sync case, we want
   it to watch `.java` files, trigger a partial build, and sync `.class` files
   so a remote server can pick up and reload the changes.

## Design

### Hack it
To get close to the functionality we want, without modifying skaffold at all, a
Dockerfile which depends on java build outputs could be used, like:
```
FROM openjdk:8
COPY build/dependencies/ /app/dependencies
COPY build/classes/java/main/ /app/classes

CMD ["java", "-cp", "/app/classes:/app/dependencies/*", "hello.Application"]
```

with a skaffold sync block that looks like:
```
sync:
  manual:
    - src: "build/classes/java/main/**/*.class"
      dest: "/app/classes"
      strip: "build/classes/java/main/"
```

A user's devloop then looks like this:

1. run `./gradlew classes copyDependencies`
1. run `skaffold dev`
1. *make changes to some java file*
1. run `./gradlew classes`
1. *skaffold syncs files*

which is far from ideal.

### A new `auto` option

Provide users with an `auto` option, users that use `auto` should expect
the builder-sync to do the right thing and will not be required to do much
configuration.

`auto` will only work with builders that have implemented the `auto` spec. We
expect at least `jib` to do implement the spec.

#### User Configuration

```yaml
build:
  artifacts:
  - image: ...
    context: jib-project
    jib: {}
    sync:
      auto: {}
```


#### Get necessary information from the builder

Skaffold can expose an API that can accept a complex configuration on how
skaffold should be doing synchronization.

The builder will talk to sync component by providing it with the following data
1. A list of `generated` configs, each containing
    1. A `command` to generate files to sync
    1. A list of inputs to watch as triggers
    1. A list of syncs (src, dest) to execute after generation
1. A list of `direct` sync directives (src, dest) to execute without any script
   execution

So maybe some datastructures like (I don't really know a lot of go, so assume
this will be written in some consistent way eventually):

```golang
type AutoSync struct {
  generated []Generated
  direct []Direct
}

type Generated struct {
  command []String
  inputs []File
  syncables []Syncables
}

type Direct struct {
  syncables []Syncables
}

type Syncables struct {
  src String
  dest String
}
```

#### Jib - Skaffold Sync interface

How a tool like Jib might surface the necessary information to Skaffold

I would expect to add a task like `_jibSkaffoldSyncMap` that will produce
json output for the skaffold-jib intergration to consume and forward to the sync
system. And example output might look like:

```
BEGIN JIB JSON: SYNCMAP/1
{
  "generated": [
    {
      src: “target/classes”
      dest: "app/classes",
    },
    {
      src: “target/resources"
      dest: "app/resources",
    }
  ]
  "direct": [
    {
      src: "src/main/extra1",
      dest: "/"
    },
    {
      src: "src/main/extra2",
      dest: "/"
    },
    {
      src: ".m2/some/dep/my-dep-SNAPSHOT.jar",
      dest: "app/dependencies/my-dep-SNAPSHOT.jar"
    }
  }
}
```

Files in the `generated` section will trigger a partial rebuild of the container
(everything before containerization) while files in the `direct` section can
just be synchronized to the running container without a rebuild of anything.

##### Sync or Rebuild?

Each builder implementing an `auto` builder should be able to decide when a sync
should be skipped and a rebuild done instead. In the jib case for instance, a
rebuild will be triggered if:
- a build file has changed (`build.gradle`, `pom.xml`, etc)
- a file is deleted

#### Open Issues/Questions

**What about files that have specific permissions?**

Jib allows users to customize file permissions, for instance a file on the
container can be configured to be 0x456 or something. One option instead of
dealing with this, is to just make all sync'd files 777? Or we allow passthrough
of permissions from the build system to the sync system.

**Should we allow the user to configure the auto block?**

Perhaps the user knows something that jib doesnt, and wants to ignore some files
from synchronization. They might want to do:

```
sync:
  auto:
    ignored:
    - "src/main/jib/myExtraFilesThatBreakADeployment"
```

## Implementation plan

- [`schemas/<version>.go`] Add `Auto` to the schema under `Sync`
- Before first build, initialize the `sync` state for builders in `auto` mode, this
sync state is saved in the builder's specific implementation of `auto`

- On file change in dev mode:
```
if (files were deleted)
  return REBUILD

if (changes were made to build def)
  return REBUILD

lastSyncState = syncStates["this project"]

if (if all files changes are in lastSyncState.direct)
  return SYNC{list of direct files}

newSyncState = buildAndCalculateNewSyncState("this project")
syncStates["this project"] = newSyncState

diff = diff(lastSyncState, newSyncState)
return SYNC{files in diff}
```

## Integration test plan

*TBD*

## Alternatives Explored

The following implementation had a few problems and that were potential deal
breakers, this implementation is left in here for information purposes, no
attempt was made to implement this. It has the following issues:
1. Depends on `sync` using `tar` which should be an implementation detail
2. Doesn't really provide a good base for exposing the `auto` sync configuration
   block. As skaffold takes care of fewer things, the user must be responsible
   for implementing it.

#### Delegate generation of sync tar to builder

One option is to complete hand over the detection of container updates to the
builder which would provide skaffold with a tar to synchronize.

A potential runthrough might look like:

1. Skaffold detects changes
1. Skaffold asks build system if sync should happen
1. Build system can reply
  1. Yes: and here is the `tar` to sync
  1. No: this will tell skaffold it should do a rebuild

This allows the builder implementation to make the sync decisions within it's
own system.

#### Jib - Skaffold Sync interface

For a potential consumer of this mechanism like jib, we would expose a task like
`_jibCreateSyncTar` which would so something like

1. If we should rebuild -> tell skaffold to rebuild, so output something like
  ```
  REBUILD
  ```
1. If we can sync
  1. Compare against last build
  2. Tar up files for synchronization
  3. Tell skaffold where to find that tar
  ```
  SYNC: /path/to/sync.tar
  ```