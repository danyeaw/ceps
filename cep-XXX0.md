# CEP XXXX - Wheel support in conda (overview)

<table>
<tr><td> Title </td><td> Wheel support in conda (overview) </td></tr>
<tr><td> Status </td><td> Draft </td></tr>
<tr><td> Author(s) </td><td>
  Dan Yeaw &lt;dyeaw@anaconda.com&gt;
</td></tr>
<tr><td> Created </td><td> Apr 24, 2026</td></tr>
<tr><td> Updated </td><td> Nov 18, 2026</td></tr>
<tr><td> Discussion </td><td> https://github.com/conda/ceps/pull/145 </td></tr>
<tr><td> Implementation </td><td> TBD </td></tr>
<tr><td> See also </td><td> [CEP XXX1 – Repodata wheel support](cep-XXX1.md), [CEP XXX2 – Wheel conda client support](cep-XXX2.md) </td></tr>
</table>

> This document provides informative context for the normative split between [CEP XXX1 – Repodata wheel support][cep-xxx1] (channel index / `whl` records) and [CEP XXX2 – Wheel conda client support][cep-xxx2] (solver, download, install). It collects motivation, history, and rejected alternatives. Unless stated otherwise, it does not add new SHALL requirements beyond pointing readers to those CEPs.

## Abstract

Pure Python wheels can be indexed in conda channel repodata so users solve environments in one pass with conda packages. This overview explains why that direction was chosen for conda, traces the historical discussion and tooling, and documents rejected ideas and ecosystem alternatives. The repodata format and publisher rules live in [CEP XXX1][cep-xxx1]; conda client behavior lives in [CEP XXX2][cep-xxx2].

## Companion CEPs

| CEP                                               | Role                                                                                                                                                                                                  |
|---------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| [CEP XXX1 – Repodata wheel support][cep-xxx1]     | Normative specification of the `whl` mapping under `v{revision}`, record fields, `METADATA` conversion, validation, and channel-operator guidance.                                                    |
| [CEP XXX2 – Wheel conda client support][cep-xxx2] | Normative specification of loading `whl` into the solver index with other repodata records, conditional dependencies and extras, download, and prefix integration as `noarch: python` conda packages. |

Together, the three documents describe native wheel support: wheels are first-class index entries in the same index as conda packages. Scope is limited to pure Python wheels (`py*-none-any` class) in [CEP XXX1][cep-xxx1] to avoid platform-specific binary compatibility questions in this round of work.

## Motivation

Conda remains a language-agnostic package manager, but most of its current use installs packages for Python. Users frequently encounter packages only available as wheels on PyPI, forcing them to use hybrid workflows that mix conda and PyPI installations. This approach creates several problems:

- Mixing of conda and pip or uv can result in overwritten files and broken environments
- Users must understand two package managers, their interaction points, and which of their dependencies are available from which ecosystem
- Tools that do support mixing environments need multiple solves, which slows installation

Adding native support for pure Python wheels, indexed in repodata per [CEP XXX1][cep-xxx1] and consumed by clients per [CEP XXX2][cep-xxx2], has the following advantages:

- Reduce the cognitive burden of managing two package managers
- Provide users with transparent access to a broader slice of the Python ecosystem through channels that choose to publish wheels
- Maintain environment consistency and reproducibility under conda's package model
- Resolve dependencies across conda and wheel records in a single solve where channels publish both
- Fill gaps in conda package availability for simpler pure-Python packages
- Reduce the maintenance burden for straightforward pure-Python packages that do not require recipe-level metadata fixes

[CEP XXX1][cep-xxx1] does not eliminate the need for conda recipes. Many packages still require feedstocks due to metadata differences, dependency corrections, compiled dependencies, or ecosystem-specific needs. Indexed wheels are an additional tool for simpler pure-Python cases.

## History

PyPI integration has been debated for years in tension with conda's emphasis on reproducible, binary-first environments.

### Early vision (2012-2014)

Early `conda` assumed environments could mix conda-installed packages with software brought in by other tools, and tried to fold those installs into environments managed by conda. The first concrete step was PyPI: within weeks of conda's first release, it shipped a `pip` subcommand.
This feature took a snapshot of untracked files, ran that environment's `pip install`, created a diff, packed the new files into one `.tar.bz2`, and then reinstalled them with conda. However, folding arbitrary pip installs into a single conda record was a poor fit for dependency identity, upgrades, and reuse. The subcommand was removed in 1.8 for that model and to reduce confusion between `conda pip` and ordinary `pip` in an activated environment.

Issue [#327](https://github.com/conda/conda/issues/327) added `--use-pypi`. conda 4.6.0 introduced experimental `prefix_data_interoperability` to reconcile pip-installed metadata with conda's when enabled. It stayed off by default because of the performance cost.

Representative issues from the same period: [#307](https://github.com/conda/conda/issues/307) (2013, pip, npm, gems, rpm, brew), [#292](https://github.com/conda/conda/issues/292) (2013, "ultimate package manager"), [#224](https://github.com/conda/conda/issues/224) (2012, native conda vs pip commands).

### The PyPI integration debate (2013-2016)

Two camps formed on whether and how to integrate PyPI better.

#### Pro-integration

Issue [#262](https://github.com/conda/conda/issues/262) (2013): conda should install directly from PyPI to reduce duplication and improve package availability.

conda-forge issue [#28](https://github.com/conda-forge/conda-forge.github.io/issues/28) (Feb 2016): Chris Barker raises the pure-Python package gap from the conda-forge community's perspective, proposing either a real-time PyPI bridge (on-the-fly conda package generation from PyPI) or an automated conda skeleton farm to mirror popular PyPI packages with version tracking.

Arguments for this approach:

- Users frequently need packages not available on conda channels
- Building conda packages for every PyPI package is unsustainable

#### Status quo

Arguments for maintaining an independent packaging ecosystem prevailed at the time for the following reasons:

- Conda's value proposition is reproducible, binary-focused environments with precise dependency resolution
- PyPI's source distributions and pip's resolver could compromise conda's guarantees
- The conda-forge community successfully scaled recipe creation

### Wheel support proposal (2017)

Issue [#5202](https://github.com/conda/conda/issues/5202) proposed direct wheel installation support:

- Wheels provide binary distributions similar to conda packages
- Growing wheel availability on PyPI reduced build complexity
- Wheel support could bridge the gap between conda's reliability and PyPI's breadth

### Broader context (2018)

The Python Discourse thread on [packaging scope boundaries](https://discuss.python.org/t/drawing-a-line-to-the-scope-of-python-packaging/883) shows the wider ecosystem struggling with the same questions:

- Multiple competing package management tools (pip, conda, poetry, pipenv)
- Unclear responsibilities and interoperability expectations
- Need for clearer standards and communication between tools

### conda-pypi development starts (2022)

Jaime Rodríguez-Guerra starts development on the conda-pypi plugin, aimed at improving conda and PyPI interoperability.

### rip development starts (2023)

Prefix.dev starts a barebones pip implemented in Rust to resolve and install PyPI dependencies with Pixi.

### whl2conda development starts (Aug 2023)

Christopher Barber develops [whl2conda](https://github.com/zuzukin/whl2conda), a command-line tool that generates conda packages from pure Python wheels (and related packaging workflows) without going through a full `conda-build` recipe for every package.

### Pixi integrates with uv (Jan 2024)

Pixi changes course to use uv directly instead of rip, which brings editable installs plus git and path dependencies.

### conda-pupa creates on-the-fly conversion plugin (July 2024)

Daniel Holth creates a conda plugin that supports on-the-fly conversion of PyPI wheels to conda packages in a local channel.

### conda-whl-channel creates proof-of-concept wheel channel from repodata (Nov 2024)

Jonathan Helmus and Anil Kulkarni begin development on [conda-whl-channel](https://github.com/Anaconda/conda-whl-channel), a proof-of-concept that adds wheels to the `packages` section of repodata and patches conda to recognize them. The implementation supports conditional dependencies through additional meta packages. In February 2025, the conda-specific functionality is extracted into a separate plugin called [conda-whl-support](https://github.com/Anaconda/conda-whl-support).

### conda-pypi merges in conda-pupa functionality (Oct 2025)

conda-pupa is merged into conda-pypi, which adds a `conda pypi install <package>` command and support for editable installations.

### conda-pypi integrates parts of conda-whl-support (Nov 2025)

conda-pypi incorporates the wheel detection logic from conda-whl-support. That covers core functionality beyond the solver and index changes needed for the `whl` section in [CEP XXX1][cep-xxx1], and adds channel relation metadata when channels adopt [Channel relations in repodata][cep-channel-relations].

### Native wheel unpack in conda-pypi; Rattler reads wheels in repodata (Dec 2025)

conda-pypi adds native support for unpacking `.whl` artifacts into the target environment (rather than delegating that step to pip). Rattler adds support for reading wheel records from repodata, for example, the `whl` mapping under a registered `v{revision}`, so clients built on Rattler can load the same index surface described in [CEP XXX1][cep-xxx1].

## Rejected ideas

### Only install Python and pip inside conda environments

In this scenario, users only install Python and pip inside a clean conda environment. Here, we simply use conda as an environment manager and let pip manage the project dependencies.

A typical session:

```bash
conda create -n pip-environment python pip
conda activate pip-environment
pip install <package>
```

This is safe for environment management, but using conda only this way gives up conda's package distribution features.

### Editable installs with `conda` for dependencies only

Conda provides all the dependencies of a given package. Then that package is installed on top in editable mode, without addressing dependencies to make sure conda files aren't accidentally overwritten:

```bash
git clone https://github.com/owner/package.git
conda create -n editable-install package --deps-only
conda activate editable-install
pip install -e . --no-deps
```

This pattern was rejected because it still splits responsibility between `conda` and `pip`: the editable project is not a first-class conda package, so `conda list`, upgrades, removals, and reproducible environment exports do not fully describe what is on disk.
Relying on `--no-deps` avoids one class of conflicts but leaves no single solver pass for the project under development and its dependencies, and it does not remove the risk of accidental `pip install` (with dependencies) overwriting files conda manages.

### Add more conda packages

Create and maintain new conda packages for each PyPI dependency needed. Tools like [Grayskull][grayskull] make the conversion easier, but this is a significant workload for the community, with over half of all conda-forge packages being pure Python. Even with more dedicated resources, creating recipes for over 500 thousand pure Python packages is not achievable.

### Add interoperability to tools through pip dependency scanning

The original version of the conda-pypi plugin called `pip` with the `--dry-run` option to work out what installing a package would pull in. With a list of all the dependencies needed, the plugin installed everything available in conda channels first and what was left over would be installed with `pip install --no-deps`. Disadvantages of this approach include:

- Users still have to know which packages they want from PyPI and run `conda pip install` on each.
- There is no guarantee that the conda and pip packages installed have ABI compatibility.
- Calling pip and conda multiple times is slow.

### Add interoperability to tools through on-the-fly conversion

The [conda-pupa][conda-pupa] plugin took this approach, and [conda-pypi][conda-pypi] then implemented it. When a user runs `conda pypi install <package>`, it fetches the required dependencies iteratively from PyPI, just like `pip`. As with the dependency scanning option above, it then installs as many dependencies as it can from `conda`.
The two diverge on what happens next: conda-pypi used `pip` to install the remaining Python dependencies, and conda-pupa converted wheel packages to conda, storing them in a local channel that caches the converted wheels on disk. That local channel also gets a repodata.json, so the solve happens entirely in conda.

This approach carries the same user-facing problem: users still have to know which packages they want from PyPI and run `conda pypi install` on them. It adds three more:

- Package conversion can take some time, especially for larger packages.
- Users must rely on a local cache for installing wheels, and this cache cannot easily be shared across computers.
- The current version must solve multiple times which is slow, although this could be optimized.

### Add interoperability to tools through uv integration

Pixi has integrated uv for installing packages from PyPI. The user adds the dependency through `pixi add --pypi <package>`. Then, when Pixi is solving the environment, it solves the conda packages using Rattler, and then calls uv to solve the PyPI dependencies. Disadvantages of this approach include:

- Like the approaches above, users still have to name the PyPI packages themselves with `pixi add --pypi`.
- Although Pixi and uv are both very fast, it is still slower than a single solve of the environment that resolves both ecosystems at once (see [Conda & PyPI solvers][pixi-conda-pypi] in the pixi documentation).

### Direct PyPI communication without repodata

Because wheels are indexed as ordinary repodata records, conda solvers resolve conda packages and wheels in the same pass, and a client does not need a separate solver that understands PyPI's index. The alternative of querying PyPI's API during a solve, fetching wheel metadata on demand instead of publishing it in repodata, was rejected:

- While resolvo (used by Rattler) supports dynamic metadata fetching during solving, as [rip](https://github.com/prefix-dev/rip) shows, libsolv requires complete package metadata upfront. This inconsistency across solvers would complicate implementation and limit compatibility.

- On-demand fetching suits pip and uv, but repodata gives conda clients a consistent model, cached index data, and a place for channels to curate and validate packages before users see them.

### Magic local channel approach

The "magic local channel" approach (<https://github.com/jaimergp/conda-pypi-channel>) was considered and rejected. It involves:

- A local FastAPI app intercepts the CLI and detects PyPI specs
- Fetches metadata on the fly and converts it to repodata (drawing on some of the ideas above)
- Downloads the wheels and converts them to .conda via whl2conda
- Caches the .conda artifacts and installs them

While this approach provides on-demand conversion and caching, it requires a separate service to be running and adds complexity to the user workflow. Native repodata wheel support keeps wheels pre-indexed in channels, so they work with standard conda workflows and no extra service has to run.

### Automatic wheel to conda package conversion and hosting

Another alternative would be establishing a build farm to automatically convert wheels to conda packages and host them on a channel, with conversion triggered by popularity, community requests, or on-demand.

#### Advantages of automatic conversion

- All packages are conda packages, which simplifies client implementation
- Mature conda infrastructure and tooling work without client changes
- Channels can enrich, check, and validate metadata before publishing

#### Disadvantages and why native wheel support was chosen

Despite these advantages, this approach was rejected because:

- Hosting converted packages requires significant storage and bandwidth and duplicates PyPI's CDN infrastructure.
- Wheels are already an excellent format for pure Python packages, so conversion adds no technical value and wastes resources.

Native wheel support gives the same user experience (transparent access to pure-Python packages through channels that index and host wheels like other packages) without the storage cost and duplicated hosting of conversion. Channel operators who prefer converted packages can continue building conda packages from PyPI sources.

## References

- [CEP XXX1 – Repodata wheel support][cep-xxx1]
- [CEP XXX2 – Wheel conda client support][cep-xxx2]
- [Adopting uv in pixi][uv-in-pixi]
- [rip][rip]
- [conda-pypi project][conda-pypi]
- [Channel relations in repodata (CEP 42)][cep-channel-relations]
- [conda-pupa][conda-pupa]

## Copyright

All CEPs are explicitly [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/).

<!-- links -->
[cep-xxx1]: cep-XXX1.md
[cep-xxx2]: cep-XXX2.md
[conda-pypi]: https://github.com/conda-incubator/conda-pypi
[cep-channel-relations]: https://conda.org/learn/ceps/cep-0042
[conda-pupa]: https://github.com/dholth/conda-pupa
[uv-in-pixi]: https://prefix.dev/blog/uv_in_pixi
[pixi-conda-pypi]: https://pixi.prefix.dev/latest/concepts/conda_pypi/#solvers
[rip]: https://github.com/prefix-dev/rip
[grayskull]: https://conda.github.io/grayskull/
