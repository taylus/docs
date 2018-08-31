# Target audience
.NET Core developers using [Travis CI](https://travis-ci.org/) builds who want code coverage automatically generated and uploaded to [Coveralls](https://coveralls.io/).

# Generating code coverage
Add the `coverlet.msbuild` NuGet package to your test project:

``` powershell
dotnet add package coverlet.msbuild
```

This adds code coverage support to `dotnet test` by passing the `/p:CollectCoverage=true` parameter. Providing this parameter will cause the test run to output code coverage results on the console as well as write them to a file `coverage.json`.

# Coveralls
[Coveralls](https://coveralls.io/) is a free service for uploading code coverage for history tracking and statistics. Sign up, link your GitHub repository, and you'll get a token (which you should keep private!) for uploading code coverage files.

Coveralls only supports particular code coverage formats, but one of those is OpenCover, which Coverlet supports with `/p:CoverletOutputFormat=opencover`.

# Travis CI
[Travis CI](https://travis-ci.org/) is a free continuous integration platform for building and testing open source projects. It builds your project by spinning up fresh Ubuntu VM, configuring it according to the `.travis.yml` file at the root of your repository, and running commands within it. Link your repository and set up that file and you should get a working build. A basic .NET Core `.travis.yml` file can look like this:

``` yaml
language: csharp
mono: none
dotnet: 2.1.300

install:
- dotnet restore

script:
 - dotnet build
 - dotnet test
```

Travis CI also supports managing environment variables so they don't have to be checked into scripts/publicly available in your project. This is great for the Coveralls token you got earlier. Set that up in Travis by going to More Options -> Settings and scrolling down to Environment Variables. Name your variable `COVERALLS_REPO_TOKEN` and paste the value in, leaving "Display value in build log" unchecked.

# Putting it all together
We're going to add code coverage generation to our Travis CI build and then use a [dotnet tool](https://docs.microsoft.com/en-us/dotnet/core/tools/global-tools) called [`coveralls.net`](https://www.nuget.org/packages/coveralls.net/) to upload it to Coveralls all automatically at the end of the build.

### The updated `.travis.yml`
``` yaml
language: csharp
mono: none
dotnet: 2.1.300

install:
- dotnet restore

script:
 - dotnet build
 - dotnet test
 - dotnet tool install --global coveralls.net --version 1.0.0
 - export PATH="$PATH:/home/travis/.dotnet/tools"
 - csmacnz.Coveralls --opencover -i /path/to/coverage.opencover.xml --useRelativePaths --commitId $TRAVIS_COMMIT --commitBranch $TRAVIS_BRANCH --commitAuthor "$REPO_COMMIT_AUTHOR" --commitEmail "$REPO_COMMIT_AUTHOR_EMAIL" --commitMessage "$REPO_COMMIT_MESSAGE" --jobId $TRAVIS_JOB_ID  --serviceName travis-ci
```

If everything works right your Travis CI build should succeed and you should see code coverage results on Coveralls and your commit history on GitHub!

# References
* http://tattoocoder.com/cross-platform-code-coverage-arrives-for-net-core/
* https://medium.com/@tonerdo/setting-up-coveralls-with-coverlet-for-a-net-core-project-2c8ec6c5dc58
* https://docs.coveralls.io/
* https://github.com/csmacnz/coveralls.net\
* https://docs.travis-ci.com/user/coveralls/
* https://docs.travis-ci.com/user/environment-variables/