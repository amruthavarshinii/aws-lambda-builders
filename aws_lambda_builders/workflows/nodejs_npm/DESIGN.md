## NodeJS - NPM Lambda Builder

### Scope

This package is an effort to port the Claudia.JS packager to a library that can
be used to handle the dependency resolution portion of packaging NodeJS code
for use in AWS Lambda. The scope for this builder is to take an existing
directory containing customer code, including a valid `package.json` manifest
specifying third-party dependencies. The builder will use NPM to include
production dependencies and exclude test resources in a way that makes them
deployable to AWS Lambda.

### Challenges

NPM normally stores all dependencies in a `node_modules` subdirectory. It
supports several dependency categories, such as development dependencies
(usually third-party build utilities and test resources), optional dependencies
(usually required for local execution but already available on the production
environment, or peer-dependencies for optional third-party packages) and
production dependencies (normally the minimum required for correct execution).
All these dependency types are mixed in the same directory.

To speed up Lambda startup time and optimise usage costs, the correct thing to
do in most cases is just to package up production dependencies. During development 
work we can expect that the local `node_modules` directory contains all the 
various dependency types, and NPM does not provide a way to directly identify
just the ones relevant for production. To identify production dependencies, 
this packager needs to copy the source to a clean temporary directory and re-run
dependency installation there.
  
A frequently used trick to speed up NodeJS Lambda deployment is to avoid 
bundling the `aws-sdk`, since it is already available on the Lambda VM.
This makes deployment significantly faster for single-file lambdas, for
example. Although this is not good from a consistency and compatibility
perspective (as the version of the API used in production might be different
from what was used during testing), people do this frequently enough that the
packager should handle it in some way. A common way of marking this with ClaudiaJS
is to include `aws-sdk` as an optional dependency, then deploy without optional
dependencies. 

Other runtimes do not have this flexibility, so instead of adding a specific
parameter to the SAM CLI, the packager should support a flag to include or
exclude optional dependencies through environment variables. 

NPM also provides support for running user-defined scripts as part of the build
process, so this packager needs to support standard NPM script execution.

NPM, since version 5, uses symbolic links to optimise disk space usage, so
cross-project dependencies will just be linked to elsewhere on the local disk 
instead of included in the `node_modules` directory. This means that just copying
the `node_modules` directory (even if symlinks would be resolved to actual paths)
far from optimal to create a stand-alone module. Copying would lead to significantly
larger packages than necessary, as sub-modules might still have test resources, and
common references from multiple projects would be duplicated.

NPM also uses a locking mechanism (`package-lock.json`) that's in many ways more
broken than functional, as it in some cases hard-codes locks to local disk
paths, and gets confused by including the same package as a dependency
throughout the project tree in different dependency categories
(development/optional/production). Although the official tool recommends
including this file in the version control, as a way to pin down dependency
versions, when using on several machines with different project layout it can
lead to uninstallable dependencies. 

NPM dependencies are usually plain javascript libraries, but they may include
native binaries precompiled for a particular platform, or require some system 
libraries to be installed. A notable example is `sharp`, a popular image 
manipulation library, that uses symbolic links to system libraries. Another 
notable example is `puppeteer`, a library to control a headless Chrome browser,
that downloads a Chromium binary for the target platform during installation.

To fully deal with those cases, this packager may need to execute the
dependency installation step on a Docker image compatible with the target
Lambda environment.

### Implementation

The general algorithm for preparing a node package for use on AWS Lambda
is as follows.

#### Step 1: Prepare a clean copy of the project source files

Execute `npm pack` to perform project-specific packaging using the supplied
`package.json` manifest, which will automatically exclude temporary files, 
test resources and other source files unnecessary for running in a production 
environment.

This will produce a `tar` archive that needs to be unpacked into the artifacts
directory.  Note that the archive will actually contain a `package`
subdirectory containing the files, so it's not enough to just directly unpack
files. 

#### Step 2: Rewrite local dependencies

_(out of scope for the current version)_

To optimise disk space and avoid including development dependencies from other
locally linked packages, inspect the `package.json` manifest looking for dependencies
referring to local file paths (can be identified as they start with `.` or `file:`),
then for each dependency recursively execute the packaging process 

Local dependencies may include other local dependencies themselves, this is a very 
common way of sharing configuration or development utilities such as linting or testing 
tools. This means that for each packaged local dependency this packager needs to
recursively apply the packaging process. It also means that the packager needs to 
track local paths and avoid re-packaging directories it already visited.

NPM produces a `tar` archive while packaging that can be directly included as a
dependency.  This will make NPM unpack and install a copy correctly. Once the
packager produces all `tar` archives required by local dependencies, rewrite
the manifest to point to `tar` files instead of the original location.

If the project contains a package lock file, this will cause NPM to ignore changes
to the package.json manifest. In this case, the packager will need to remove 
`package-lock.json` so that dependency rewrites take effect. 
_(out of scope for the current version)_

#### Step 3: Install dependencies

The packager should then run `npm install` to download an expand all dependencies to
the local `node_modules` subdirectory. This has to be executed in the directory with
a clean copy of the source files.

Note that NPM can be configured to use proxies or local company repositories using 
a local file, `.npmrc`. The packaging process from step 1 normally excludes this file, so it may
need to be copied additionally before dependency installation, and then removed. 
_(out of scope for the current version)_

Some users may want to exclude optional dependencies, or even include development dependencies. 
To avoid incompatible flags in the `sam` CLI, the packager should allow users to specify 
options for the `npm install` command using an environment variable.
_(out of scope for the current version)_

To fully support dependencies that download or compile binaries for a target platform, this step
needs to be executed inside a Docker image compatible with AWS Lambda. 
_(out of scope for the current version)_

