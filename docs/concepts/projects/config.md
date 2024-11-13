# Configuring projects

## Entry points

uv uses the standard `[project.scripts]` table to define entry points for the project.

For example, to declare a command called `hello` that invokes the `hello` function in the
`example_package_app` module:

```toml title="pyproject.toml"
[project.scripts]
hello = "example_package_app:hello"
```

!!! important

    Using `[project.scripts]` requires a [build system](#build-systems) to be defined.

## Build systems

Projects _may_ define a `[build-system]` in the `pyproject.toml`. The build system defines how the
project should be packaged and installed.

uv uses the presence of a build system to determine if a project contains a package that should be
installed in the project virtual environment. If a build system is not defined, uv will not attempt
to build or install the project itself, just its dependencies. If a build system is defined, uv will
build and install the project into the project environment. By default, projects are installed in
[editable mode](https://setuptools.pypa.io/en/latest/userguide/development_mode.html) so changes to
the source code are reflected immediately, without re-installation.

## Project packaging

While uv usually uses the declaration of a [build system](#build-systems) to determine if a project
should be packaged, uv also allows overriding this behavior with the
[`tool.uv.package`](../../reference/settings.md#package) setting.

Setting `tool.uv.package = true` will force a project to be built and installed into the project
environment. If no build system is defined, uv will use the setuptools legacy backend.

Setting `tool.uv.package = false` will force a project package _not_ to be built and installed into
the project environment. uv will ignore a declared build system when interacting with the project.

## Project environment path

The `UV_PROJECT_ENVIRONMENT` environment variable can be used to configure the project virtual
environment path (`.venv` by default).

If a relative path is provided, it will be resolved relative to the workspace root. If an absolute
path is provided, it will be used as-is, i.e. a child directory will not be created for the
environment. If an environment is not present at the provided path, uv will create it.

This option can be used to write to the system Python environment, though it is not recommended.
`uv sync` will remove extraneous packages from the environment by default and, as such, may leave
the system in a broken state.

!!! important

    If an absolute path is provided and the setting is used across multiple projects, the
    environment will be overwritten by invocations in each project. This setting is only recommended
    for use for a single project in CI or Docker images.

!!! note

    uv does not read the `VIRTUAL_ENV` environment variable during project operations. A warning
    will be displayed if `VIRTUAL_ENV` is set to a different path than the project's environment.

## Limited resolution environments

If your project supports a more limited set of platforms or Python versions, you can constrain the
set of solved platforms via the `environments` setting, which accepts a list of PEP 508 environment
markers. For example, to constrain the lockfile to macOS and Linux, and exclude Windows:

```toml title="pyproject.toml"
[tool.uv]
environments = [
    "sys_platform == 'darwin'",
    "sys_platform == 'linux'",
]
```

Entries in the `environments` setting must be disjoint (i.e., they must not overlap). For example,
`sys_platform == 'darwin'` and `sys_platform == 'linux'` are disjoint, but
`sys_platform == 'darwin'` and `python_version >= '3.9'` are not, since both could be true at the
same time.

## Build isolation

By default, uv builds all packages in isolated virtual environments, as per
[PEP 517](https://peps.python.org/pep-0517/). Some packages are incompatible with build isolation,
be it intentionally (e.g., due to the use of heavy build dependencies, mostly commonly PyTorch) or
unintentionally (e.g., due to the use of legacy packaging setups).

To disable build isolation for a specific dependency, add it to the `no-build-isolation-package`
list in your `pyproject.toml`:

```toml title="pyproject.toml"
[project]
name = "project"
version = "0.1.0"
description = "..."
readme = "README.md"
requires-python = ">=3.12"
dependencies = ["cchardet"]

[tool.uv]
no-build-isolation-package = ["cchardet"]
```

Installing packages without build isolation requires that the package's build dependencies are
installed in the project environment _prior_ to installing the package itself. This can be achieved
by separating out the build dependencies and the packages that require them into distinct extras.
For example:

```toml title="pyproject.toml"
[project]
name = "project"
version = "0.1.0"
description = "..."
readme = "README.md"
requires-python = ">=3.12"
dependencies = []

[project.optional-dependencies]
build = ["setuptools", "cython"]
compile = ["cchardet"]

[tool.uv]
no-build-isolation-package = ["cchardet"]
```

Given the above, a user would first sync the `build` dependencies:

```console
$ uv sync --extra build
 + cython==3.0.11
 + foo==0.1.0 (from file:///Users/crmarsh/workspace/uv/foo)
 + setuptools==73.0.1
```

Followed by the `compile` dependencies:

```console
$ uv sync --extra compile
 + cchardet==2.1.7
 - cython==3.0.11
 - setuptools==73.0.1
```

Note that `uv sync --extra compile` would, by default, uninstall the `cython` and `setuptools`
packages. To instead retain the build dependencies, include both extras in the second `uv sync`
invocation:

```console
$ uv sync --extra build
$ uv sync --extra build --extra compile
```

Some packages, like `cchardet` above, only require build dependencies for the _installation_ phase
of `uv sync`. Others, like `flash-attn`, require their build dependencies to be present even just to
resolve the project's lockfile during the _resolution_ phase.

In such cases, the build dependencies must be installed prior to running any `uv lock` or `uv sync`
commands, using the lower lower-level `uv pip` API. For example, given:

```toml title="pyproject.toml"
[project]
name = "project"
version = "0.1.0"
description = "..."
readme = "README.md"
requires-python = ">=3.12"
dependencies = ["flash-attn"]

[tool.uv]
no-build-isolation-package = ["flash-attn"]
```

You could run the following sequence of commands to sync `flash-attn`:

```console
$ uv venv
$ uv pip install torch
$ uv sync
```

Alternatively, you can provide the `flash-attn` metadata upfront via the
[`dependency-metadata`](../../reference/settings.md#dependency-metadata) setting, thereby forgoing
the need to build the package during the dependency resolution phase. For example, to provide the
`flash-attn` metadata upfront, include the following in your `pyproject.toml`:

```toml title="pyproject.toml"
[[tool.uv.dependency-metadata]]
name = "flash-attn"
version = "2.6.3"
requires-dist = ["torch", "einops"]
```

!!! tip

    To determine the package metadata for a package like `flash-attn`, navigate to the appropriate Git repository,
    or look it up on [PyPI](https://pypi.org/project/flash-attn) and download the package's source distribution.
    The package requirements can typically be found in the `setup.py` or `setup.cfg` file.

    (If the package includes a built distribution, you can unzip it to find the `METADATA` file; however, the presence
    of a built distribution would negate the need to provide the metadata upfront, since it would already be available
    to uv.)

Once included, you can again use the two-step `uv sync` process to install the build dependencies.
Given the following `pyproject.toml`:

```toml title="pyproject.toml"
[project]
name = "project"
version = "0.1.0"
description = "..."
readme = "README.md"
requires-python = ">=3.12"
dependencies = []

[project.optional-dependencies]
build = ["torch", "setuptools", "packaging"]
compile = ["flash-attn"]

[tool.uv]
no-build-isolation-package = ["flash-attn"]

[[tool.uv.dependency-metadata]]
name = "flash-attn"
version = "2.6.3"
requires-dist = ["torch", "einops"]
```

You could run the following sequence of commands to sync `flash-attn`:

```console
$ uv sync --extra build
$ uv sync --extra build --extra compile
```

!!! note

    The `version` field in `tool.uv.dependency-metadata` is optional for registry-based
    dependencies (when omitted, uv will assume the metadata applies to all versions of the package),
    but _required_ for direct URL dependencies (like Git dependencies).

## Editable mode

By default, the project will be installed in editable mode, such that changes to the source code are
immediately reflected in the environment. `uv sync` and `uv run` both accept a `--no-editable` flag,
which instructs uv to install the project in non-editable mode. `--no-editable` is intended for
deployment use-cases, such as building a Docker container, in which the project should be included
in the deployed environment without a dependency on the originating source code.
