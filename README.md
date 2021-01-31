build-compiler-dist
===

This project downloads, builds, packages, and releases distributions of the
llvm-mos project.

Dependencies
---

- [CMake](https://www.cmake.org) 3.10 or later
- [Github CLI](https://cli.github.com/)
- [zip](https://zlib.net/)
- [git](https://www.git-scm.com)
- A locally installed C/C++ compiler combination
- Accurate settings for the [GITHUB_TOKEN and GH_REPO](https://cli.github.com/manual/gh_help_environment) 
  environment variables, for pushing the distribution

Targets
---

Name           | Description
---            | ---
build | Download and build llvm-mos
download-cache | Downloads a CMake cache file for priming the LLVM build
package        | Zip up the built compiler suite
ship           | Deploy the package on Github as per GITHUB_TOKEN and GH_REPO

