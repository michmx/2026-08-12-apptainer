# Bonus Episode: Building and deploying an Apptainer container to GitHub Packages

:::{admonition} Overview
:class: note
**Teaching:** 40 min | **Exercises:** 0 min

**Questions**
- How to build an Apptainer container for Python packages?
- How to share Apptainer images?

**Objectives**
- To be able to build an Apptainer container and share it via GitHub Packages
:::

:::{admonition} Prerequisites
:class: caution
The previous episode ended the Introduction to Apptainer/Singularity.
This bonus episode is an optional extension mixing knowledge from different courses.
For this lesson, you will also need:
* Knowledge of Git [SW Carpentry Git-Novice Lesson](https://swcarpentry.github.io/git-novice/) (for simplified authentication with the `gh` CLI, see [this version of the git training](https://mambelli.github.io/git-novice/07-github.html))
* Knowledge of GitHub CI/CD [HSF Github CI/CD Lesson](https://hsf-training.github.io/hsf-training-cicd-github/)
:::

## Apptainer Container for Python packages

Python packages can be installed using an Apptainer image. The following example illustrates how to write a definition file for building an image containing Python packages.

```text
Bootstrap: docker
From: ubuntu:24.04

%post
    apt-get update -y
    apt-get install -y python3 python3-pip
    pip3 install --break-system-packages numpy awkward uproot particle hepunits \
        matplotlib hist mplhep vector fastjet iminuit
```

As we see, several Python packages from the [Scikit-HEP](https://scikit-hep.org/) ecosystem are installed.
The `--break-system-packages` option is needed to install packages with `pip` outside a virtual environment
on modern Ubuntu versions, as seen in the [instances episode](08-instances.md).


## Publish Apptainer images with GitHub Packages and share them!

It is possible to publish Apptainer images with [GitHub Packages](https://github.com/features/packages).
To do so, one needs to use GitHub CI/CD. A step-by-step guide is presented here.

* **Step 1**: Create a GitHub repository and clone it locally.
* **Step 2**: In the empty repository, make a folder called `.github/workflows`. In this folder we will store the file containing the YAML script for a GitHub workflow, named `apptainer-build-deploy.yml` (the name doesn't really matter).
* **Step 3**: In the top directory of your GitHub repository, create a file named `Apptainer`.
* **Step 4**: Copy the definition file content shown above into the `Apptainer` file. (In principle it is possible to build this image locally, but we will not do that here, as we wish to build it with GitHub CI/CD).
* **Step 5**: In the `apptainer-build-deploy.yml` file, add the following content:

```yaml
name: Apptainer Build Deploy

on:
  pull_request:
  push:
    branches: [master, main]

jobs:
  build-test-container:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    container:
        image: ghcr.io/apptainer/apptainer:1.4.5
        options: --privileged

    name: Build Container
    steps:

      - name: Check out code for the container builds
        uses: actions/checkout@v7

      - name: Build Container
        run: |
           apptainer build container.sif Apptainer

      - name: Login and Deploy Container
        # Deploy only on pushes, not on pull requests (the GITHUB_TOKEN of a PR cannot write packages)
        if: github.event_name == 'push'
        # Use default registry user ${{ github.repository_owner }} , or set a secret ${{ secrets.GHCR_USERNAME }}
        # GHCR image names must be lowercase, so the repository name is converted with `tr`
        run: |
           echo ${{ secrets.GITHUB_TOKEN }} | apptainer registry login -u ${{ github.repository_owner }} --password-stdin oras://ghcr.io
           apptainer push container.sif oras://ghcr.io/$(echo "${GITHUB_REPOSITORY}" | tr '[:upper:]' '[:lower:]'):latest
```

The above script is designed to build and publish an Apptainer image with [GitHub Packages](https://github.com/features/packages).
Note that the deploy step only runs on pushes to the `main` (or `master`) branch; for pull requests the workflow
only tests that the image builds successfully.


* **Step 6**: Add LICENSE and README as recommended in the [SW Carpentry Git-Novice Lesson](https://swcarpentry.github.io/git-novice/), and then the repository is good to go.

:::{admonition} Key Points
:class: note
- Python packages can be installed in Apptainer images along with Ubuntu packages.
- It is possible to publish and share Apptainer images via GitHub Packages.
:::
