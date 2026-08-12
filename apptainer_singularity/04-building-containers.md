# Building Containers

:::{admonition} Overview
:class: note
**Teaching:** 20 min | **Exercises:** 10 min

**Questions**
- How to build containers with my requirements?

**Objectives**
- Download and assemble containers from available images in the repositories.
:::

Running containers from the available public images is not the only option. In many cases, it is required to modify
an image or even to create a new one from scratch. For such purposes, Apptainer provides the command `build`,
defined in the documentation as the _Swiss army knife_ of container creation.

The usual workflow is to prepare and test a container in a build environment (like your laptop),
either with an interactive session or from a definition file,
and then to deploy the container into a production environment for execution (such as your institutional cluster).
Interactive sessions are great to experiment and test your new container.
If you want to distribute the container or use it in production, then we recommend building it from a definition file, as described in the next episode.
This ensures the greatest possibility of reproducibility and transparency.

<figure>
  <img src="https://journals.plos.org/plosone/article/figure/image?size=large&id=10.1371/journal.pone.0177459.g001" alt="Apptainer/Singularity usage workflow"/>
  <figcaption>'Apptainer/Singularity usage workflow' via <i>Kurtzer GM, Sochat V, Bauer MW (2017) Singularity: Scientific containers for mobility of compute. PLoS ONE 12(5): e0177459. <a href="https://doi.org/10.1371/journal.pone.0177459">https://doi.org/10.1371/journal.pone.0177459</a></i></figcaption>
</figure>

## Build a container in an interactive session

While images contained in the `.sif` files are more compact and immutable objects, ideal for reproducibility, for building and testing images
it is more convenient to use a _sandbox_, which can be easily modified.

The command `build` provides a flag `--sandbox` that will create a writable directory, `myAlma10`, in your work directory:
```bash
apptainer build --sandbox myAlma10 docker://almalinux:9
```

:::{admonition} Notes on shared file systems like AFS
:class: tip
Avoid using the [`AFS` (Andrew File System)](https://en.wikipedia.org/wiki/Andrew_File_System) and possibly other shared file systems
as sandbox directory, as these systems can lead to permission issues.
Symptoms could be warnings like `harmless EPERM on setxattr "security.capability"` when building the sandbox and
IO errors during commands execution, e.g. failure to install RPM packages.
In particular, this applies to your home directory on [`lxplus`](https://cern.service-now.com/service-portal?id=service_element&name=lxplus-service)
and to the home and nocache directories on [`cmslpc`](https://uscms.org/uscms_at_work/computing/getstarted/uaf.shtml).
Instead, make sure to use the local file system by creating a folder in `/tmp/`: `mkdir /tmp/$USER`.
Then, replace `myAlma10` in the previous (and next) command with `/tmp/$USER/myAlma10`.
:::

The container name is `myAlma10`, and it has been initialized from the [official Docker image](https://hub.docker.com/_/almalinux)
of AlmaLinux 9.
To initialize an interactive session use the `shell` command. And to write files within the sandbox directory use the `--writable` option.
Finally, the installation of new components will require superuser access inside the container, so use also the `--fakeroot` option, unless you are already root also outside.
```bash
apptainer shell --writable --cleanenv --fakeroot myAlma10
Apptainer> whoami
```
```text
root
```
`--cleanenv` clears the environment. It has been added to make sure that the eventual setting of
variables on the host is not affecting the container.
Variables like PYTHONPATH or PYTHONHOME are affecting the Python execution inside the container.
A corrupted Python environment could cause errors like "ModuleNotFoundError: No module named 'encodings'".

:::{admonition} Apptainer environment
:class: tip
[Environment variables in Linux](https://www.geeksforgeeks.org/environment-variables-in-linux-unix/)
are dynamic values that can affect programs and containers.
You can use the environment to pass variables to a container.
Apptainer by default preserves most of the outside environment inside the container
but has many options to control that.
You can clear the environment with the `--cleanenv` option and you can set variables with `--env`.
See the [Apptainer manual](https://apptainer.org/docs/user/main/environment_and_metadata.html)
for more options and details.

PYTHONPATH and PYTHONHOME affect the Python execution, but other variables could affect other programs so,
if you don't care about the outside environment, you can add `--cleanenv`
every time you start a container (`apptainer shell`, `exec` and `instance start` commands).
:::

Depending on the Apptainer/Singularity installation (privileged or unprivileged) and the version,
you may have some requirements, like the `fakeroot` utility or `newuidmap` and `newgidmap`.
If you get an error when using `--fakeroot` have a look at the [fakeroot documentation](https://apptainer.org/docs/user/main/fakeroot.html).

:::{admonition} `--fakeroot` is not root
:class: tip
ATTENTION! [`--fakeroot`](https://apptainer.org/docs/user/main/fakeroot.html) allows you to be root inside a container that you own but is not changing who you are outside.
All the outside actions and the writing on bound files and directories will happen as your outside user, even if inside the container it is done by root.
:::

As an example, let's create a container with Pythia8 available using the `myAlma10` sandbox.
First, we need to enable the [EPEL](https://docs.fedoraproject.org/en-US/epel/) (Extra Packages for Enterprise Linux) repositories
and to install the development tools (remember that in this interactive session we are superuser):
```bash
Apptainer> dnf install epel-release
Apptainer> dnf groupinstall 'Development Tools'
Apptainer> dnf install python3-devel
```
Where `dnf` is the [package manager used in RHEL distributions](https://en.wikipedia.org/wiki/DNF_(software))
(like AlmaLinux).

Pythia is now distributed as RPM in EPEL, so you can use this to install it:
```bash
Apptainer> dnf install pythia8
Apptainer> dnf install python3-pythia8
Apptainer> dnf install pythia8-devel  # optional, if you need also the development libraries to compile
```
:::

Now, open an interactive session with your user (no `--fakeroot`). You can use now the container with Pythia8 ready in a
few steps. Let's use the Python interface:

```bash
apptainer shell myAlma10

Apptainer> python3

>>> import pythia8
>>> pythia = pythia8.Pythia()
```



::::{admonition} Build a container with Uproot available
:class: important
Build a container to use [Uproot](https://github.com/scikit-hep/uproot5),
a library for reading and writing ROOT files in pure Python and NumPy, in Python 3.12.

:::{admonition} Solution
:class: dropdown
Start from the [Python 3.12 Docker image](https://hub.docker.com/_/python) and create the `myPython` sandbox:
```bash
apptainer build --sandbox myPython docker://python:3.12
apptainer shell myPython
```
Once inside the container, you can install [Uproot](https://uproot.readthedocs.io/en/latest/index.html).
```bash
Apptainer> python3 -m pip install --upgrade pip
Apptainer> python3 -m pip install uproot awkward
```
Exit the container and use it as you like:
```bash
apptainer exec myPython python -c "import uproot; print(uproot.__doc__)"
```
```text
Uproot: ROOT I/O in pure Python and NumPy.
...
```
Notice how we needed neither `--writable` nor `--fakeroot` for the installation, but everything worked fine since pip installs user packages in the user `$HOME` directory.
You will see new files under `$HOME/.local/`.
In addition, Apptainer/Singularity by default mounts the user home directory as read+write, even if the container is read-only.
This is why a _sandbox_ is great to test and experiment locally, but should not be used for containers that will be shared or deployed. Manual changes and local directories are difficult to reproduce and control. Once you are happy with the content, you should use definition files, described in the next episode.
:::
::::

:::{admonition} Key Points
:class: note
- The command `build` is the basic tool for the creation of containers.
- A _sandbox_ is a writable directory where containers can be built interactively.
- Superuser permissions are required to build containers if you need to install packages or manipulate the operating system.
- Use interactive builds only for development and tests, use definition files for production or publicly distributed containers.
:::
