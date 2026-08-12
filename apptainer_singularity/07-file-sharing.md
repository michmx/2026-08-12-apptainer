# Sharing files between host and container

:::{admonition} Overview
:class: note
**Teaching:** 30 min | **Exercises:** 0 min

**Questions**
- How to read and write files on the host system from within the container?

**Objectives**
- Map directories on your host system to directories within your container.
- Learn about the bind paths included automatically in all containers.
:::

One of the key features about containers is the isolation of the processes running inside them. This means that
files on the host system are not accessible within the container.
However, it is very common that some files on the host system are needed inside the container,
or you want to write files from the container to some directory in the host.

We have already used the option `--bind` earlier in the module when exploring the options available to run Apptainer
containers. In this chapter we will explore further options to bind directories from your host system to directories
within your container.

Remember that in Apptainer, your user outside is the same inside the container (except when using fakeroot).
And the same happens with permissions and ownership for files in bind directories.

## Bind paths included by default

For each container executed,
[Apptainer binds automatically some directories by default](https://apptainer.org/docs/user/main/bind_paths_and_mounts.html#system-defined-bind-paths),
and others defined by the system admin in the Apptainer configuration. By default, Apptainer binds:
* The user's home directory ($HOME)
* The current directory when the container is executed ($PWD)
* System-defined paths: `/tmp`, `/proc`, `/dev`, etc.

Since this is defined in the configuration, it may vary from site to site.

Let's use for example the container built during the last chapter called `rootInUbuntu.sif`. Take a look at your
current directory
```bash
pwd
```
```text
/home/myuser/somedirectory
```
Open a shell inside the container and try to use `pwd` again
```bash
apptainer shell rootInUbuntu.sif

Apptainer> pwd
```
```text
/home/myuser/somedirectory
```
you will notice that the files stored on the host are located inside the container! As we explained above, Apptainer
mounts automatically your `$HOME` inside the container.

:::{admonition} Disabling system binds
:class: tip
If for any reason you want to execute a container removing the default binds, the command-line option `--no-mount`
is available. For example, to disable the bind of `/tmp`
```bash
apptainer run --no-mount tmp my_container.sif
```
:::

Try this time with
```bash
apptainer shell --no-mount home,cwd rootInUbuntu.sif
```
and you will notice that `$HOME` is not mounted anymore
```bash
ls /home/myuser
```
```text
ls: cannot access '/home/myuser': No such file or directory
```

Note how we disabled both `home` and `cwd` (current working directory). This is because if you are running the apptainer
command from your home directory, even if you use `--no-mount home` the home directory may still be mounted
because it is also your current directory.

## User-defined bind paths

Apptainer provides mechanisms to specify additional binds when executing a container via command-line
or environment variables. Apptainer offers a complex set of mechanisms for binds or other mounts.
Here we present the main points, refer to the
[Bind Paths and Mounts documentation](https://apptainer.org/docs/user/main/bind_paths_and_mounts.html) for more.

### Bind with command-line options

The command-line option `--bind (-B)` will specify the directories that must be linked between the
host and the container. It is available for `run`, `exec`, and `shell` (as well for `instance` that is
not covered yet).

The syntax for using the bind option is `"source:destination"`, and the paths must be absolute (relative
paths will be rejected). For example, let's create a directory in the host containing a constant that can be useful
for your analysis
```bash
mkdir $HOME/mydata
echo "MUONMASS=105.66 MeV" > $HOME/mydata/muonMass.txt
```
It is very, very important in your analysis workflow to know the mass of the muon, right? It may make sense to put the data
in a high-level directory within the container, like `/data`
```bash
apptainer shell --bind $HOME/mydata:/data rootInUbuntu.sif
```
This will bind the directory `mydata/` from the host as `/data` inside the container:
```bash
ls -l /data
```
```text
-rw-rw-r-- 1 myuser myuser 20 Jan  2 12:46 muonMass.txt
```

Now you can use the mass of the muon from a root-level directory!

If multiple directories must be available in the container, you can repeat the option or they can be defined with a comma between each pair of directories,
i.e. using the syntax `source1:destination1,source2:destination2`.

Also, if the destination is not specified, it is set equal to the source. For example
```bash
apptainer shell --bind /cvmfs rootInUbuntu.sif
```
will mount `/cvmfs` inside the container. Try it!

:::{admonition} Binding directories with Docker-like syntax using `--mount`
:class: tip
The flag `--mount` provides a method to bind directories using the syntax of Docker.
The bind is specified with the format `type=bind,src=<source>,dst=<dest>`.
Currently, only `type=bind` is supported. Check the
[documentation](https://apptainer.org/docs/user/main/bind_paths_and_mounts.html#mount-examples) for
additional options available.
:::

### Bind with environment variables

If the environment variable `$APPTAINER_BIND` is defined, apptainer will bind inside ANY container
the directories specified in the format `source:destination`, with the destination being optional (in the same way as using
`--bind`). For example:
```bash
export APPTAINER_BIND="/cvmfs"
```
will bind CVMFS to all your Apptainer containers (`/cvmfs` must be available in the host, of course).
The `$SINGULARITY_BIND` variable is also honored, for compatibility with Singularity.

You can also bind multiple directories using commas between each `source:destination`.

:::{admonition} Key Points
:class: note
- Bind mounts allow reading and writing files within the container.
- In Apptainer, you have same owner and permissions for files inside and outside the container.
- Some paths are mounted by default by Apptainer.
- Additional directories to bind can be defined using the `--bind` option or the environment variable `$APPTAINER_BIND`.
:::
