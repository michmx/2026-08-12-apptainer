# Containers from definition files

:::{admonition} Overview
:class: note
**Teaching:** 20 min | **Exercises:** 20 min

**Questions**
- How to easily build and deploy containers from a single definition file?

**Objectives**
- Create a container from a definition file.
:::

As shown in the previous chapter, building containers with an interactive session may take several steps,
and it can become as complicated as the required setup.
An Apptainer definition file provides an easy way to build and deploy containers.


## Hello World Apptainer

The following recipe shows how to build a hello-world container, and run the container on your local computer.

- Step 1: Open a text editor (e.g., nano, vim, or gedit in a graphical environment)

  ```bash
  nano hello-world.def
  ```

- Step 2: Include the following script in the `hello-world.def` file to define the environment

  ```text
  Bootstrap: docker
  From: ubuntu:24.04

  %runscript
    echo "Hello World"
  # Print Hello world when the image is loaded
  ```

    In the above script, the first line - `Bootstrap: docker` indicates that Apptainer will use the Docker protocol to retrieve the base OS to start the image.
The `From: ubuntu:24.04` is given to Apptainer to start from a specific image/OS in Docker Hub.
Any content within the  `%runscript` will be written to a file that is executed when one runs the apptainer image.
The `echo "Hello World"` command will print the `Hello World` on the terminal.
Finally the `#` hash is used to include comments within the definition file.

- Step 3: Build the image

  ```bash
  apptainer build hello-world.sif hello-world.def
  ```

    The `hello-world.sif` file specifies the name of the output file that is built when using the `apptainer build` command.

- Step 4: Run the image

  ```bash
  ./hello-world.sif
  ```

### Deleting Apptainer image
To delete the hello-world Apptainer image, simply delete the `hello-world.sif` file.

:::{admonition} `apptainer delete`
:class: tip
Note that there is also an `apptainer delete` command, used to delete an image from a remote library.
To learn more about using remote endpoints and pulling and pushing images from or to libraries, read
[Remote Endpoints](https://apptainer.org/docs/user/main/endpoint.html) and [Library API Registries](https://apptainer.org/docs/user/main/library_api.html).
:::


## Example of a more elaborated definition file

Let's look at the structure of the definition file with another example. Let's prepare a container from an [official
Ubuntu image](https://hub.docker.com/_/ubuntu), but this time we will install ROOT with RooFit and Python integration.


Following the ROOT instructions to
[download a pre-compiled binary distribution](https://root.cern/install/#download-a-pre-compiled-binary-distribution),
the definition file will look like

```text
Bootstrap: docker
From: ubuntu:24.04

# NOTE: This section is only if building the container in the JupyterHub at CompHEP 2026
%setup
    mkdir -p ${APPTAINER_ROOTFS}/cvmfs
    mkdir -p ${APPTAINER_ROOTFS}/direct/u0b
    mkdir -p ${APPTAINER_ROOTFS}/u0b/software

%post
    apt-get update -y
    export DEBIAN_FRONTEND=noninteractive
    apt-get install wget -y
    apt-get install binutils cmake dpkg-dev g++ gcc libssl-dev git libx11-dev \
        libxext-dev libxft-dev libxpm-dev python3 libtbb-dev libvdt-dev libgif-dev \
        libgsl-dev -y
    cd /opt
    wget https://root.cern/download/root_v6.38.04.Linux-ubuntu24.04-x86_64-gcc13.3.tar.gz
    tar -xzvf root_v6.38.04.Linux-ubuntu24.04-x86_64-gcc13.3.tar.gz

%environment
    export PATH=/opt/root/bin:$PATH
    export LD_LIBRARY_PATH=/opt/root/lib:$LD_LIBRARY_PATH
    export PYTHONPATH=/opt/root/lib

%runscript
    python3 /opt/root/tutorials/roofit/roofit/rf101_basics.py

%labels
    Author HEPTraining
    Version v0.0.1

%help
    Example container running the RooFit tutorial and producing the rf101_basics.png image.
    The container provides ROOT with RooFit and Python integration running on Ubuntu.
```

:::{admonition} What is `export DEBIAN_FRONTEND=noninteractive` for?
:class: tip
Some Debian/Ubuntu packages pause during installation to ask configuration questions
(for example, `tzdata` asks for your geographic region and time zone).
During `apptainer build` there is no terminal to type answers into, so a prompt like this would hang the build.
Setting `DEBIAN_FRONTEND=noninteractive` tells `apt-get` to skip all prompts and accept the default answers,
letting the build run unattended.
It only affects the commands that follow it inside `%post`; it does not change the environment of the final container.
:::

Let's take a look at the [definition file](https://apptainer.org/docs/user/main/definition_files.html):
* The first two lines define the base image. In this case, the image `ubuntu:24.04` from Docker Hub is used.
* `%post` are lines to execute inside the container after the OS has been set. In this example, we are listing the
steps that we would follow to install ROOT with a precompiled binary in an interactive session.
Notice that the binary used corresponds with the Ubuntu version defined at the second line.
* `%environment` is used to define environment variables available inside the container. Here we are setting the env
variables required to execute ROOT and PyROOT.
* Apptainer containers can be executable. `%runscript` define the actions to take when the container is executed.
To illustrate the functionality, we will just run [rf101_basics.py](https://root.cern/doc/master/rf101__basics_8py.html)
from the RooFit tutorial.
* `%labels` add custom metadata to the container.
* `%help` is the container documentation: what it is and how to use it. It can be displayed using `apptainer run-help`.

Save this definition file as `rootInUbuntu.def`. To build the container, just provide the definition file as argument.
Modern Apptainer versions build images without any special privileges; with older versions you may need `sudo` or the `--fakeroot` option:
```bash
apptainer build rootInUbuntu.sif rootInUbuntu.def
```


Then, an interactive shell inside the container can be initialized with `apptainer shell`, or
a command executed with `apptainer exec`. A third option is to execute the actions defined inside `%runscript`
simply by calling the container as an executable

```bash
./rootInUbuntu.sif
```

```text
...
 PARAMETER  CORRELATION COEFFICIENTS
       NO.  GLOBAL      1      2
        1  0.02723   1.000  0.027
        2  0.02723   0.027  1.000
[#1] INFO:Minimization -- RooMinimizer::optimizeConst: deactivating const optimization
RooRealVar::mean = 1.01746 +/- 0.0300144  L(-10 - 10)
RooRealVar::sigma = 2.9787 +/- 0.0219217  L(0.1 - 10)
Info in <TCanvas::Print>: png file rf101_basics.png has been created
```

You will find the output file `rf101_basics.png` in the location where the container was executed.
The exact output depends on the ROOT version used in the container.
If you don't have a `DISPLAY` set up, ROOT may complain. Ignore the error messages; the image will be created anyway.

Here we have covered the basics with a few examples focused on HEP software.
Check the [Apptainer docs](https://apptainer.org/docs/user/main/build_a_container.html) to see all the available
options and more details related to the container creation.

A few [best practices for your containers](https://apptainer.org/docs/user/main/definition_files.html#best-practices-for-build-recipes) to make them more usable, portable, and secure:
1. Always install packages, programs, data, and files into operating system locations (e.g. not `/home`, `/tmp` , or any other directories that might get commonly binded on).
1. Document your container. If your runscript doesn’t supply help, write a `%help` or `%apphelp` section. A good container tells the user how to interact with it.
1. If you require any special environment variables to be defined, add them to the `%environment` and `%appenv` sections of the build recipe.
1. Files should always be owned by a system account (UID less than 500).
1. Ensure that sensitive files like `/etc/passwd`, `/etc/group`, and `/etc/shadow` do not contain secrets.
1. Build production containers from a definition file instead of a sandbox that has been manually changed. This ensures the greatest possibility of reproducibility and mitigates the “black box” effect.

:::{admonition} Deploying your containers
:class: tip
Keep in mind that, while building a container may be time consuming, the execution can be immediate and anywhere your image is available.
Once your container is built with the requirements of your analysis, you can deploy it in a large cluster and execute it
as far as Apptainer is available on the site.

Libraries like [Sylabs Cloud Library](https://cloud.sylabs.io/library) ease the distribution of images.
Your institution (e.g. Fermilab or CERN) may provide a [Harbor](https://goharbor.io/) registry.
GitHub has a [Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
that Apptainer can access via the ORAS API.
Organizations like OSG provide instructions to [use available images](https://portal.osg-htc.org/documentation/htc_workloads/using_software/containers/)
and [distribute custom images via CVMFS](https://portal.osg-htc.org/documentation/htc_workloads/using_software/containers-docker/).

Be smart, and this will open endless possibilities in your workflow.
:::


::::{admonition} Write a definition file to build a container with Pythia8 available in Python
:class: important
Following the example of the first section in which a container is built with an interactive session
(see the previous episode),
write a definition file to deploy a container with Pythia8 available.

Take a look at
[`/opt/pythia/pythia8310/examples/main01.py`](https://gitlab.com/Pythia8/releases/-/blob/pythia8310/examples/main01.py)
and define the `%runscript` to execute it using `python3`.

(Tip: notice that main01.py requires `Makefile.inc`).

:::{admonition} Solution
:class: dropdown
```text
Bootstrap: docker
From: almalinux:9

%post
    dnf -y groupinstall 'Development Tools'
    dnf -y install python3-devel
    mkdir /opt/pythia && cd /opt/pythia
    curl -o pythia8310.tgz https://pythia.org/download/pythia83/pythia8310.tgz
    tar xvfz pythia8310.tgz
    cd pythia8310
    ./configure --with-python-include=/usr/include/python3.9
    make

%environment
    export PYTHONPATH=/opt/pythia/pythia8310/lib:$PYTHONPATH
    export LD_LIBRARY_PATH=/opt/pythia/pythia8310/lib:$LD_LIBRARY_PATH

%runscript
    cp /opt/pythia/pythia8310/Makefile.inc .
    python3 /opt/pythia/pythia8310/examples/main01.py

%labels
    Author HEPTraining
    Version v0.0.2

%help
    Container providing Pythia 8.310. Execute the container to run an example.
    Open it in a shell to use the Pythia installation with Python 3.9
```

Save this definition file as `myPythia8.def` and build your container executing

```bash
apptainer build pythiaInAlma9.sif myPythia8.def
```

And finally, execute the container to run [`main01.py`](https://gitlab.com/Pythia8/releases/-/blob/pythia8310/examples/main01.py)

```bash
./pythiaInAlma9.sif
```

This solution is building Pythia from scratch and may take several minutes to build the container.
In the previous episode we saw that binary packages of Pythia are available in EPEL.
Use them to build a similar container much faster.
:::
::::

:::{admonition} Key Points
:class: note
- An Apptainer definition file provides an easy way to build and deploy containers.
:::
