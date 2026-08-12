# Bonus Episode: Sandboxing an AI coding agent

:::{admonition} Overview
:class: note

**Questions**
- How to let an AI agent read, write, and execute code without giving it access to the whole system?
- How to run a container in full isolation, except for the current directory?

**Objectives**
- Run an agentic AI tool from a container image.
- Achieve full isolation with the `--containall` option.
- Expose a single directory inside an isolated container with `--bind`.
:::

:::{admonition} Prerequisites
:class: caution
This bonus episode is an optional extension.
For this lesson, you will also need:
* The episodes [Running containers](02-running-containers.md) and [Sharing files](07-file-sharing.md).
* An URL and API key for a LLM service.
* Internet access from the machine running the container, if the model itself runs remotely.
:::

## Why sandbox an AI agent?

Agentic AI tools like [opencode](https://opencode.ai/) do not just answer questions: given a task, they read your files,
write code, and execute shell commands in a loop until the task is done. This makes them useful assistants for analysis
work, and it is also exactly why you should think twice before running one directly on your laptop or on the login node
of your institute's cluster. A misunderstood prompt, or a plain wrong answer, can end with the agent modifying files far
outside your project: your `$HOME`, your SSH keys, your other analyses.

Containers are a natural sandbox for this. However, remember from the [Sharing files episode](07-file-sharing.md) that
by default Apptainer mounts your `$HOME`, the current directory, and several system paths inside every container.
That default is convenient for analysis work, but it is *not* isolation: an agent running in such a container can still
touch everything in your home directory. 

In this episode we will turn the logic around and run a container in full
isolation, then deliberately expose one single directory: the project we want the agent to work on.

## Getting opencode

opencode is an open-source AI coding agent that runs in the terminal and can be used with open models.
Official container images are distributed in DockerHub. Pull the image with

```bash
apptainer pull opencode.sif docker://ghcr.io/anomalyco/opencode
```

and check that it works:
```bash
apptainer exec opencode.sif opencode --version
```

:::{admonition} Building your own agent image
:class: tip
The official image contains the agent plus common developer tools (`git`, `node`, `python3`), but not the
scientific Python stack. 

If you want additional tools like Scikit-HEP, you can build your own image
with a [definition file](05-definition-files.md). Either:

 - Start from the definition file of the
previous episodes (which installs the Scikit-HEP packages) and install opencode in the
`%post` section with `curl -fsSL https://opencode.ai/install | bash`
 - Or start from the official 
opencode image and install the packages you need
:::

## Using an open model

Opencode can talk to many model providers. Here we will use the open-weights models provided by SCDF at BNL. 

First, create a fresh working directory for the agent. This will be the *only* directory visible inside the sandbox:
```bash
mkdir $HOME/agent-project
cd $HOME/agent-project
```

The model is selected in a file named `opencode.json`, placed in the working directory itself. Create it with the
following content:
```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "sdcc": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "SDCC",
      "options": {
        "baseURL": "https://inference0-api.sdcc.bnl.gov/v1",
        "apiKey": "sk-hs4g2BTNuuaI1XpVtSt4OA"
      },
      "models": {
        "nemotron-3-super-120b": {
          "name": "nemotron-3-super-120b",
          "limit": {
            "context": 131072,
            "output": 32768
          }
        }
      }
    }
  }
}
```


## Full isolation with `--containall`

The option `--containall` runs a container with the strongest isolation Apptainer offers as an unprivileged user:
* No directories are bound from the host, not even `$HOME` or the current directory. Instead, a temporary in-memory
  home directory is created, whose contents vanish when the container exits.
* The environment is cleaned, as with `--cleanenv` from the
  [Building containers episode](04-building-containers.md).
* The processes inside get their own PID and IPC namespaces, so the agent cannot even see the other processes
  running on the host.

Starting from full isolation, we add back exactly what we want to share, and nothing more:
```bash
apptainer shell --containall --bind $PWD --pwd $PWD opencode.sif
```

Let's dissect the command:
* `--containall` gives the fully isolated container.
* `--bind $PWD` exposes the current directory (and only it), since `--containall` disables the automatic binds.
* `--pwd $PWD` makes the shell start in that directory inside the container.

And if needed, API keys or other variables can be passed into the otherwise clean environment with `--env`.

Take a look around from inside:
```bash
Apptainer> ls $HOME
```
```text
agent-project
```
Your home directory *appears* to exist, but it is the temporary in-memory one: none of your real files are there,
and the only entry is the project directory we explicitly bound. Writes to the project directory reach the host as
usual (same user, same permissions); writes anywhere else are discarded when the container exits.

Note that Apptainer does not isolate the network by default. This is what allows opencode to reach the remote model
while the filesystem stays sealed.

## Running the agent

opencode can run non-interactively with `opencode run`, which is a good first test. Exit the container shell
(or use `apptainer exec` directly from the host):
```bash
apptainer exec --containall --bind $PWD --pwd $PWD opencode.sif \
    opencode run "Write a Python script pt_histogram.py that opens data.root with uproot, reads the branch Muon_pt from the tree Events, and saves a histogram of it to pt_histogram.png"
```

The agent will report what it is doing and finish by creating the script in the working directory:
```bash
ls
```
```text
opencode.json  pt_histogram.py
```

For longer sessions, start the interactive terminal interface instead: open the sandboxed shell as before and simply
run `opencode` inside it. There you can converse with the agent, for example
```text
Explain what pt_histogram.py does, and add the CMS label with mplhep
```
and review each change it proposes before accepting it.

The agent *writes* code; to execute `pt_histogram.py` you still need Python with `uproot` available, which the
opencode image does not include. You can run the generated script with the Scikit-HEP container built in the
previous episodes, or build a single image containing both the agent and the analysis stack.

:::{admonition} Keeping the agent's memory
:class: tip
opencode stores its session history under `$HOME`, which in our sandbox is the temporary in-memory directory:
every conversation is forgotten when the container exits. If you want sessions to survive while staying isolated,
give the container a persistent home *inside* the project directory:
```bash
mkdir -p $PWD/agent-home
apptainer shell --containall --bind $PWD --pwd $PWD \
    --home $PWD/agent-home:/home/myuser opencode.sif
```
Everything the agent remembers is then stored in `agent-home/`, still within the single directory you chose to share.
:::

::::{admonition} Prove that the sandbox holds
:class: important
Create a (fake) secret file in your home directory on the host:
```bash
echo "MYTOKEN=abc123" > $HOME/secret.txt
```
Now enter the sandboxed container shell and try to read it. Can you? Which option of the command is responsible
for protecting the file?

:::{admonition} Solution
:class: dropdown
From inside the sandbox, the file is simply not there:
```bash
Apptainer> cat /home/myuser/secret.txt
```
```text
cat: /home/myuser/secret.txt: No such file or directory
```
The option `--containall` is the one responsible: it disables the default bind of `$HOME` (and everything else),
replacing it with an empty temporary directory. Only `$HOME/agent-project` is visible, because we bound it
explicitly with `--bind`. Without `--containall`, the default binds would have exposed `secret.txt` to the agent.
:::
::::


:::{admonition} Key Points
:class: note
- Agentic AI tools edit files and execute commands autonomously, so they should run in a sandbox.
- The `--containall` option runs a container with no default binds, a clean environment, a temporary home directory, and isolated PID/IPC namespaces.
- Combining `--containall` with `--bind $PWD --pwd $PWD` exposes exactly one directory to the container.
- The `--env` option passes selected variables, such as API keys, into the clean environment.
- The network is shared by default, which lets the agent reach a remote model while the filesystem stays isolated.
:::
