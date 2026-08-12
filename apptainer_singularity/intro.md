# Introduction to Apptainer

Apptainer (formerly known as Singularity) is a free and open-source container platform that allows you
to create and run applications in isolated environments (also called "containers") in a simple, portable, fast, and secure manner.

Many container platforms are available, but Apptainer is designed to bring containers and reproducibility to the scientific community and High-Performance Computing (HPC) use cases.
Using Apptainer, developers can work in reproducible environments of their choice and design, and these complete environments can be easily copied and executed on other platforms.

This is an introduction to Apptainer, its motivations and applications in HEP.

Based on the [Apptainer user guide](https://apptainer.org/docs/).

:::{admonition} The HSF Training Center
:class: seealso
This is a condensed version of the HSF Training on Apptainer for [CompHEP 2026](https://indico.cern.ch/event/1672591/). The original lesson can be found [here](https://hsf-training.github.io/hsf-training-singularity-webpage/).
:::

:::{admonition} Prerequisites
:class: caution
* Basic knowledge of the Unix Shell, e.g., from the [Software Carpentry course](https://swcarpentry.github.io/shell-novice/).
* Access to a computing system with Apptainer/Singularity available. It can either be installed locally, or the machine can have user namespaces enabled and access to CVMFS.
* This training concludes with Episode 8. The bonus episodes are optional extensions: Episode 9 also requires Git and CI/CD knowledge and will allow you to integrate that knowledge with the use of Apptainer/Singularity, while Episode 10 shows how to sandbox an AI coding agent and requires a free account with a model provider.
:::

## Table of Contents

```{tableofcontents}
```
