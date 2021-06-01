continuous-release
===

This project downloads, builds, packages, and releases distributions of the
llvm-mos project.

It does so by priming the CMake cache with a cmake file
downloaded from the llvm-mos-sdk project, and then downloading and building 
llvm-mos using CMake's ExternalProject functionality.  After building 
a distribution and optionally testing, it pushes the resultant build to 
Github as a named and tagged release.

User-configurable flags are at the beginning of CMakeLists.txt, and they
should be both self-explanatory and overrideable.

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

Name | Description
--- | ---
build | Download and build llvm-mos
package | Zip up the built compiler suite
ship | Deploy the package on Github as per GITHUB_TOKEN and GH_REPO

