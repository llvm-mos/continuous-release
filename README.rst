build-compiler-dist
===================

This project downloads, builds, packages, and releases distributions of the
llvm-mos project.  It depends on:

- CMake 3.17 or later
- `Github CLI <https://cli.github.com/>`_
- A locally installed C/C++ compiler combination
- Accurate settings for the `GITHUB_TOKEN and GH_REPO environment variables
  <https://cli.github.com/manual/gh_help_environment>`_