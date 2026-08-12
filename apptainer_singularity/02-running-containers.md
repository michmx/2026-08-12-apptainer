# Containers and Images

:::{admonition} Overview
:class: note
**Teaching:** 20 min | **Exercises:** 5 min

**Questions**
- How to pull Apptainer images from the libraries?
- How to run commands inside the containers?

**Objectives**
- Learn to search and pull images from the Sylabs Singularity library and Docker Hub.
- Interact with the containers using the command-line interface.
:::

## The Apptainer Command Line Interface

Apptainer provides a command-line interface (CLI) to interact with the containers. You can search, build or run
containers in a single line.

You can check the version of the Apptainer command you are using with the `--version` option:
```bash
apptainer --version
```
For this training we recommend Apptainer >= 1.0. Older versions may not have some of the features or behave differently.
If you need to install or upgrade Apptainer please refer to the [Setup section](setup.md).

When asking for support please remember to include the version of Apptainer being used, as in the output of the above command.

You can check the available options and subcommands using `--help`:

```bash
apptainer --help
```

## Downloading Images

Container Images are executables that bundle together all necessary components for an application or an environment,
like a template for containers.
Containers are the runtime instances of images — they are images with a state. CircleCI has a nice
[explanation of the differences](https://circleci.com/blog/docker-image-vs-container/).

Apptainer can store, search and retrieve images in registries (searchable catalogs and repositories for images and containers).
Images built by other users can be accessed using the CLI, pulled down, and become containers at runtime.

Sylabs, the developer of one Singularity flavor, hosts a public image registry, the
[Singularity Container Library](https://cloud.sylabs.io/library) where many user built images are available.

Apptainer, the Linux Foundation flavor, does not point by default to the Sylabs registry via the
[Library API](https://singularityhub.github.io/library-api/#/) as previous versions did.
You can change that running these commands (documented [here](https://apptainer.org/docs/user/main/endpoint.html#restoring-pre-apptainer-library-behavior)):
```bash
apptainer remote add --no-login SylabsCloud cloud.sycloud.io
```
```text
INFO:    Remote "SylabsCloud" added.
```
```bash
apptainer remote use SylabsCloud
```
```text
INFO:    Remote "SylabsCloud" now in use.
```
```bash
apptainer remote list
```
```text
Cloud Services Endpoints
========================

NAME           URI                  ACTIVE  GLOBAL  EXCLUSIVE
DefaultRemote  cloud.apptainer.org  NO      YES     NO
SylabsCloud    cloud.sycloud.io     YES     NO      NO
...
```

:::{admonition} Remote Endpoints, Library API and OCI Registries
:class: tip
[Remotes](https://apptainer.org/docs/user/main/endpoint.html) are service endpoints Apptainer interacts with.
These include [Library API Registries](https://apptainer.org/docs/user/main/library_api.html),
[OCI Registries](https://apptainer.org/docs/user/main/docker_and_oci.html), and keyservers.
The first two are used to search, pull and push images.
The [Library](https://singularityhub.github.io/library-api/#/) API, `library://`,
was designed for SIF images, the [Singularity Image Format](https://github.com/apptainer/sif).
The [Docker](https://docs.docker.com/docker-hub/api/latest/)/[ORAS](https://oras.land/) API, `docker://`,
is used for Docker Hub, and other OCI ([Open Containers Initiative](https://opencontainers.org/)) registries.
These include [Quay.io](https://quay.io), [NVIDIA NGC](https://ngc.nvidia.com/),
the [GitHub Container Registry](https://github.com/features/packages),
[AWS ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/registry_auth.html) and many more.
:::

Once you have set up a working registry you can use search and pull.
The command `search` lists containers of interest
and shows information about users (owners or managers of stored containers) and collections (sets of containers).
For example:

```bash
# this command can take around a minute to complete
apptainer search almalinux
```

```text
No users found for 'almalinux'

Found 1 collections for 'almalinux'
        library://dtrudg-sylabs-2/base-2022-07-29

Found 3 containers for 'almalinux'
        library://library/default/almalinux
                Tags: 8 8.4 8.6 9 9.0 latest
...
```

Downloading an image from the Container Library is pretty straightforward:
```bash
apptainer pull library://library/default/almalinux:9
```
and the image is stored locally as a `.sif` file (`almalinux_9.sif`, in this case).

:::{admonition} Docker Images
:class: tip
Fortunately, Apptainer is also compatible with Docker images. There are many more registries with Docker images.
[Docker Hub](https://hub.docker.com/) is one of the largest libraries available,
and any image hosted on the hub can be easily downloaded with the `docker://` URL as reference:
```bash
apptainer pull docker://almalinux:9
```
:::


:::{admonition} Docker Hub limit error
:class: tip
Docker Hub [limits the number of image pulls](https://docs.docker.com/docker-hub/usage/): unauthenticated users get 100 pulls per 6 hours from a single IP address (200 per 6 hours for authenticated users with a free account).
This may happen in workshops, also because a single image may require multiple downloads.
You will see a TOOMANYREQUESTS error like:
```text
FATAL:   While making image from oci registry: error fetching image to cache: while building SIF from layers: conveyor failed to get:
GET https://index.docker.io/v2/library/almalinux/manifests/8:
TOOMANYREQUESTS: You have reached your unauthenticated pull rate limit. https://www.docker.com/increase-rate-limit
```
The solution is to authenticate if you have a Docker Hub account, or to change IP address (i.e. work from another computer), or to find a different image registry.
For example here is the [Ubuntu gallery on AWS](https://gallery.ecr.aws/ubuntu/ubuntu) where you can find the image pull URLs.
In apptainer you'll have to add the server name not to use the default Docker Hub, e.g.
```bash
apptainer pull docker://public.ecr.aws/ubuntu/ubuntu:24.04
```
*Keep this in mind for later if you see the error!*
:::

## Running Containers

There are several ways to interact with images and start containers. Here we will review how to initialize a shell
environment and how to execute directly a command.

### Initializing a shell and exiting it

The `shell` command initializes a new interactive shell inside the container.

```bash
apptainer shell almalinux_9.sif
```

```text
Apptainer>
```
In this case, the container works as a lightweight virtual machine in which you can execute commands.
Remember, inside the container you have the same user and permissions.

```bash
Apptainer> id
```

```text
uid=1001(myuser) gid=1001(myuser) groups=1001(myuser),500(myothergroup)
```

Now quit the container by typing

```bash
Apptainer> exit
```

or hitting `Ctrl + D`.
Note that when exiting from the Apptainer image all the running processes are killed (stopped).
Changes saved into bound directories are preserved. By default anything else in the container is lost (we'll see later about writable images).

### Bound directories

When an outside directory is accessible also inside Apptainer we say it is *bound*, or bind mounted. The path to access it
may differ but anything you do to its content outside is visible inside and vice-versa.
By default, Apptainer binds the home of the user, `/tmp` and `$PWD` into the container. This means your files
at `hostname:~/` are accessible inside the container. You can specify additional bind mounts using the `--bind` option.
For example, let's say `/cvmfs` is available in the host, and you would like to have access to CVMFS inside the
container (here, *host* refers to the computer/server that you are running apptainer on). Then let's do

```bash
apptainer shell --bind /cvmfs:/mnt almalinux_9.sif
```

Here, the colon `:` separates the path to the directory on the host (`/cvmfs/`) from the mounting point (`/mnt/`) inside of the
container.
If you do not have CVMFS, you can try the command with [`/opt`](https://stackoverflow.com/a/12649407/), for example.
More information on binding is provided [later](07-file-sharing.md).

Let's check that this works:

```text
Apptainer> ls /mnt/cms.cern.ch
bin                        etc                  SITECONF           slc7_aarch64_gcc530
bootstrap.sh               external             slc5_amd64_gcc434  slc7_aarch64_gcc700
...
```

:::{admonition} URLs as input
:class: tip
Each of the different commands to set a container from a local `.sif` also accepts the URL of the image
as input. For example, starting a shell with Rocky Linux 9 is as easy as
```bash
apptainer shell docker://rockylinux/rockylinux:9
```
```text
INFO:    Converting OCI blobs to SIF format
INFO:    Starting build...
Getting image source signatures
Copying blob 7ecefaa6bd84 done
Copying config a8f7ea56a4 done
Writing manifest to image destination
Storing signatures
2026/08/04 10:15:30  info unpack layer: sha256:7ecefaa6bd84a24f90dbe7872f28a94e88520a07941d553579434034d9dca399
INFO:    Creating SIF file...
Apptainer>
```
:::

### Executing commands

The command `exec` starts the container from a specified image and executes a command inside it.

Let's use a .sif image created from the official [Docker image of ROOT](https://hub.docker.com/r/rootproject/root) to start [ROOT](https://root.cern/)
inside a container:  
```bash
apptainer exec /cvmfs/belle.sdcc.bnl.gov/containers/root/root_latest.sif root -b
```

And just like that, ROOT can be used in any laptop, large-scale cluster or grid system
with Apptainer available.

:::{admonition} Using the URL
:class: tip
You can also use the URL of the image directly, without downloading it first:
```bash
apptainer exec docker://rootproject/root root -b
```
However, this will take some time to convert the image to SIF format.
:::


::::{admonition} Execute Python with PyROOT available
:class: important
Start a Python session with PyROOT available.

:::{admonition} Solution
:class: dropdown
```bash
apptainer exec --cleanenv /cvmfs/belle.sdcc.bnl.gov/containers/root/root_latest.sif python3
```
`--cleanenv` is optional but makes the command more robust (see the [Building Containers episode](04-building-containers.md)).

```text
Python 3.13.7 (main, Aug  1 2025, 12:00:00) [GCC 15.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import ROOT
>>> # Now you can work with PyROOT, creating a histogram for example
>>> h = ROOT.TH1F("myHistogram", "myTitle", 50, -10, 10)
```
:::
::::

:::{admonition} Key Points
:class: note
- Use `apptainer --version` to know what you are using and to communicate it if asking for support.
- A container can be started from a local `.sif` or directly with the URL of the image.
- Apptainer is also compatible with Docker images, providing access to the large collection of images hosted by Docker Hub.
- Get a shell inside of your container with `apptainer shell <path/URL to image>`.
- Execute a command inside of your container with `apptainer exec <path/URL> <command>`.
- Bind outside directories with `--bind`.
:::
