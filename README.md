# RISC OS CI templates

## Introduction

This repository contains a number of templates for use with building RISC OS
projects within CI systems. Primarily it focusses on GitHub, but there exists
a GitLab configuration as well.

More details about the build system can be found in the [build service documentation](https://build.riscos.online/ci-build.html).

More examples of using the CI system can be found on GitHub, tagged with the topic `riscos-ci`.

## How to use the .robuild.yaml

First you need to set up the RISC OS source that you want to build, or run, in your repository.
The files should be stored in the standard NFS format - using `filename,XXX` where `XXX` is the filetype.

Next, copy the `.robuild.yaml` from this repository to your respoitory. This file will be used to control what's built. The template file here runs a program called `TestProgram` and expects that the output of the build will be placed in the `Artifacts` directory. You can modify these in the the file by changing the `script` and `artifacts` blocks of the definition. Multiple script lines may be given, prefixed by `-`.

In your script any command which returns an error, or a non-zero return code when using a C-like tool, will cause the execution to fail, and no more code will be run. Any return code will be returned from the `riscos-build-online`, or a return code of 1 if an error was returned. If you wanted you could make the `script` block run an obey file (filetype `feb`) which does all the work. This may make it easier to test on RISC OS itself, but you will need to handle any returns from C-like tools yourself.

To test your code works under the RISC OS build system, you can download the build tool (see https://github.com/gerph/robuild-client/) and use it to test `.robuild.yaml` file. You can do this on RISC OS or a POSIX-like system, but examples here are only for POSIX-like systems.

Download the tool with something like this:

```bash
curl -s -L -o riscos-build-online https://github.com/gerph/robuild-client/releases/download/v0.05/riscos-build-online && chmod +x riscos-build-online
```

This leaves you with the build tool (`riscos-build-online`) in your working directory, so that you can use it.

To send code to the build service, you can either send a single file which should be built, or a zip archive containing the files to run (which may include a `.robuild.yaml` file). The simplest way to do this is with a command line that archives everything you want to send and then invokes the tool. To do that for this example repository, which runs the tool you could do something like this:

```bash
rm -f /tmp/source-archive.zip ; zip -9r /tmp/source-archive.zip TestProgram,fd1 .robuild.yaml ; ./riscos-build-online -i /tmp/source-archive.zip -o /tmp/built
```

This archives the files `TestProgram,fd1` and `.robuild.yaml` and sends them to the build system. The result is written to the file `/tmp/built,<filetype>` where the filetype depends on the type of file that was returned by the build system as its artifact. If the `artifacts` section is present in the `.robuild.yaml` file, this will be a RISC OS zip archive containing the files that were in that directory. Alternatively you can use the `*`-command `Clipboard_FromFile` to copy a file to the clipboard - this clipboard file will be used as the artifact to return.

You would see output something like the following for this template repository:

```
  adding: TestProgram,fd1 (deflated 27%)
  adding: .robuild.yaml (deflated 48%)
System: RISC OS Build System version 2.0.126-0.30.2761
Server: Source loaded
Server: Started build
Build: Build tool selected: ROBuild YAML
Output:
  Program renumbered
  OS version:    RISC OS 7.30 (09 Dec 2021) [Pyromaniac 0.30.2761 on Linux/x86_64]
  BASIC version: BASIC V version 1.36 © RISCOS Ltd
  
Result file: filetype=&a91, 132 bytes
RC: 0
Build: Return code: 0
Server: Build complete
```

If you look at the code in `TestProgram,fd1` you can see that this is the BASIC code that produced the above output.

Depending on the repository, you might need to archive different files. For example, if you have a C module you want to build you might use:

```bash
rm /tmp/source-archive.zip ; zip -9r /tmp/source-archive.zip c h cmhg Make* .robuild.yaml ; ./riscos-build-online -i /tmp/source-archive.zip -o /tmp/built
```

You may need to repeat the operation a few times to get to the point that you're happy that the code is building and running properly.


## How to use the .github/workflows/ci.yml

Once you have a working `.robuild.yaml` file, you can copy the relevant CI files to your repository. Here the example of the GitHub actions is given, but there is a separate configuration for GitLab's CI.

In the simplest of cases you should copy across the `.github/workflows/ci.yml` file to your repository, with the same directory names. You can name the file anything you like, so long as it ends in `.yml`.

The file will need some small tweaks for the project you're building:

- The name of the resulting archive is `ExampleBuild` in the template, so search and replace that with the name you want for the output.
- The script which builds the tool is set up to allow only 60 seconds of runtime. This is so that by default, we don't ever run away and soak up CPU time pointlessly. You're not paying for it, but it will slow the system down and might even cause it to crash, so be a nice citizen.
    - If you need more time, increase it by changing the `-t` parameter to the tool.
    - There is a maximum of 10 minutes for the default way of running the tool. There are ways to increase this, by talking directly to the back end, but if you need this, please contact the author at gerph@gerph.org as there are usually lots of things we can do better at that point.
- The version number applied to the output artifact is read from a `VersionNum` file which is commonly used in RISC OS sources. If no `VersionNum` is present, the tool will default to the git hash.
    - If you have a different way of determining the version number, change the code - this is shell code, so you'll need to be familiar with that. This is not specific to the build service, you can contact the author at gerph@gerph.org if you need specific help here.

Once you've made these changes commit the code and push. I recommend you commit this code on a branch with an appropriate name so that you're not polluting the main branch (for example, on a branch named `add-ci`). Once the code is pushed, GitHub will start to build the changes - go to the Actions tab to find the build details.  Here you will see a list of commits that triggered actions, and these link to their output and results.

The logs for the run can be seen whilst it's running (and afterward). This can help you identify problems. If something goes wrong, it will fail and report this with a red 'x'. Examine the logs and see what's happened.

If you need to make changes it is commonly useful to use `git commit -a --amend; git push -f` to just modify the previous change and push it. This avoids polluting your history with lots of silly little changes showing where you went wrong. You wouldn't usually do this, but for testing the CI, the only way to test things is to make the change and push, so avoiding cluttering history is good here.

Once it's working you should be able to see in the 'Summary' of the action that there were artifacts produced. You can download these and see that it's produced the correct RISC OS content. If it hasn't, repeat the modifications process until it does.

Finally, merge your fixed branch into the main branch and it will now be available to anyone who wants it. It will build everything you push, and report an 'x' if it fails - it can even email you to tell you that the builds or tests went wrong.

One additional thing is that if you create a tag which starts with a `v` this will cause a build to happen and at the end of the process a new draft 'Release' will be created. These can be found in the 'Releases' section on the right of the repository. A draft release contains the source referenced by that tag and the result artifacts from your build. You can either publish that release or delete it, if it was not suitable.

If you want to remove this feature, remove the `release` block from the `.github/workflows/ci.yml`file. To trigger based on different tags, change the `tags` block at the start of the file, and the `if: startsWith(github.ref, 'refs/tags/v')` in the `releases` block.
