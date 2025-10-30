---
title: Utilizing the Agentic Lifecycle for Improved Generative Results
date: 2025-10-28
description: We have never had it so easy to jump straight into `agent` mode! What many have come to learn the hard way is, this can lead to many unanticipated changes or assumptions, leading to surprise behaviours or hallucinations from the generative model.
tags: ["Open Source", "Artificial General Intelligence", "Software Development"]
Cover: https://images.unsplash.com/photo-1474314243412-cd4a79f02c6a
---

_It's so easy to jump into the deep end, to vibe._


With the rise of modern editors with [agentic](https://en.wikipedia.org/wiki/Agentic_AI) capabilities such as Cursor, Zed and of course Visual Studio Code, it’s never been easier to jump into the craze of _vibe coding_. We have never had it so easy to jump straight into `agent` mode! What many have come to learn the hard way is, this can lead to many unanticipated changes or assumptions, leading to surprise behaviours or hallucinations from the generative model. Providing a prompt such as `write unit tests for the OrderServiceController` within the context of a codebase would imply that it would write unit tests for said `OrderServiceController`. If a developer were to be more ambitious or worse, ambiguous in their prompt, what can we ascertain will be the result?

Generative AI hallucinates quite often, sometimes in subtle, quiet ways which aren’t obvious either. They pull in dependencies which don’t exist, or rewrite entire modules because it doesn’t comply with their ideal result to your prompt; they rewrite the same function five times before opting to rewrite the entire package.

That’s where understanding and employing a lifecycle which allows us to provide context, answer questions and even go over design and concepts before any code has been changed allows for a more effective partnership with AGI tools. Those agent-enabled editors mentioned support this entire lifecycle out of the box thanks to their chat-like widgets with the ‘assistant’ using one of three modes that can be used: Ask, Edit, and Agent.

## Ask

> “Ask mode for chat is optimized for asking questions about your codebase, coding and general technology concepts. Ask mode is particularly useful for getting a better understanding of your codebase, brainstorming ideas, and getting help with coding tasks.”

- Visual Studio Code
    >

![Screenshot 2025-10-28 at 7.28.46 PM.png](attachment:a764caaa-e523-4504-ab1c-e5b6b80c7cd0:Screenshot_2025-10-28_at_7.28.46_PM.png)

Ontop of those listed, I find ask utterly fantastic for learning a new codebase, troubleshooting an error within a module, and planning how to approach my task within the file. We can use `Ask` to essentially `chat` with our AI assistant without any codebase changes occurring, allowing developers to “go over” their thinking and design, revising and prompting for additional insights or known issues as if you were pair programming. By going over intention, design and behaviour prior to using `Edit` or `Agent` mode, we can benefit from outcomes like:

- A concrete understanding of the proposed changes
- Details are more defined for the AGI to follow, leading to less hallucination potential or surprises.
- The ability to catch conflicting concepts before anything has been changed

## Edit

> “With [chat] edit mode in Visual Studio Code, you can use natural language to make edits across multiple files in your project. The edits are applied directly in your editor, where you can review them in-place, with the full context of the surrounding code”

- Visual Studio Code
    >

`Edit` allows for us to make surgical modifications to a single file; all within the scope that a developer can follow along and review. Use this when you have small changes within a single file, such as updating the behaviour of a function or writing new test cases, or improving the perfect `README.md`. The outcomes from this mode should be pretty clear:

- Visible, easy to review changes to your file
- Leverage Generative AI to update and improve a document, section, etc.

## Agent

> “Agent mode is optimized for complex coding tasks based on high-level requirements that might require running terminal commands and tools. The AI operates autonomously, determining the relevant context and files to edit, planning the work needed, and iterating to resolve issues as they arise.”

- Visual Studio Code
    >

`Agent` truly shows the power, danger and potential fallacy of giving an AGI access to an entire local codebase. An Agent can do far more than just edit a single file. It can scaffold directories and files until it’s blue in the face, it can write a TODO list of a hundred items and proceed to do them, it can run commands from your terminal. More impressively, it can also interface with Model Context Protocol (MCP) servers to carry out more complicated and intertwined tasks between services. Outcomes from using Agent mode:

- Full access to your editor’s context, the workspace and tooling
- Step-by-step automated processes which are auditable

## Let’s see it in action: Updating my Developer Environment Container Repo

In `Ask` mode, I provide the following prompt: **Given how I've implemented my various domain containers, how best should I write a new domain which I can use for AI Chats using the amp free script for setup?** `curl -fsSL <https://ampcode.com/install.sh> | bash` And I get the following response from `GPT-5 mini`,

> Nice — you already have a consistent pattern for your other domains (they all `FROM localhost/base:latest` and use Podman). Below I’ll give a recommended approach and a ready-to-copy Containerfile you can drop into your repo to create an `ai-chat` domain that installs the Amp CLI (via the script you pasted), keeps the install under a non-root user, and is easy to run for development.
>
> High-level recommendations
>
> - Base image: reuse `localhost/base:latest` like your other domains.
> - Run system package installs (curl/wget/tar/unzip/ca-certificates/bash) required by the Amp install script.
> - Create a non-root user (e.g. `dev`) so `~/.amp` will be created under a normal user — matches how the install script behaves by default.
> - Run the Amp install script as that non-root user (it will download Bun and run the bootstrap).
> - Keep the image small by cleaning package manager caches.
> - For development mount `~/.amp` from the host as a volume to avoid re-installing on every run.
> - Do not bake secrets (API keys) into the image — pass them at runtime via env vars (e.g. `OPENAI_API_KEY` or your provider’s env var).
> - Add a Makefile target (same pattern as existing domains) for `ai-chat` so it’s consistent with your repo.
>
> ```docker
> FROM localhost/base:latest
>
> # Install tools required by the Amp install script and typical dev utilities
> RUN dnf update -y && \
>     dnf install -y curl wget tar unzip ca-certificates bash passwd shadow && \
>     dnf clean all
>
> # Create a non-root user to run AMP under
> ARG USER=dev
> ARG UID=1000
> ARG GID=1000
> RUN groupadd -g ${GID} ${USER} || true && \
>     useradd -m -u ${UID} -g ${GID} -s /bin/bash ${USER} && \
>     # ensure home exists and minimal permissions
>     mkdir -p /home/${USER} && chown -R ${USER}:${USER} /home/${USER}
>
> # Set AMP_HOME for the user and ensure bin directory exists
> ENV AMP_HOME=/home/${USER}/.amp
> ENV PATH=${AMP_HOME}/bin:${PATH}
>
> # Install Amp CLI (runs the upstream installer as the non-root user)
> # The installer downloads Bun and runs bootstrap.ts; running as ${USER}
> USER ${USER}
> WORKDIR /home/${USER}
> RUN curl -fsSL https://ampcode.com/install.sh | bash
>
> # Switch back to root for any final image tweaks if you want
> USER root
> WORKDIR /work
>
> # Default non-root entrypoint (drop to shell by default)
> CMD ["/bin/bash"]
> ```
>
> Questions / next steps
>
> - Do you want this container only for development (interactive `amp` usage), or do you need it to run a production chat service (HTTP process + systemd/entrypoint)?
> - Which LLM provider or runtime do you plan to use (OpenAI, Anthropic, local Llama-like models)? If you tell me, I can:
>     - Add required packages and run-time environment to the Containerfile.
>     - Provide a sample `docker`/`podman` run command for local testing that wires env vars and volumes properly.
>     - Add a suggested Makefile addition that matches your repo conventions.
>
> If you want, I can also produce a patch (Containerfile + Makefile change) ready to drop into `containerFile/Containers/ai-chat`. Which option do you want next?

Now I have a sample Containerfile which follows the standards established in the repository, along with a few additional configuration changes which I never thought of before. I like the proposed changes so I’ll switch to `Edit` mode and ask it to write the Containerfile that it’s proposing. As we can see in the screenshot, it did exactly that and would later prompt me to suggest that it could make additional edits and improvements.

![Screenshot 2025-10-28 at 8.12.48 PM.png](attachment:5c58d04d-49ca-4a33-bb8b-cbacef95f095:Screenshot_2025-10-28_at_8.12.48_PM.png)

> Next suggestions (optional)
>
> - I can add a Makefile entry consistent with your other domain targets so `make ai-chat` will build it.
> - If you want the container to run a chat service automatically (not just provide a shell), tell me the service command and I’ll change the `CMD`.
> - If you prefer the Amp installer to run at container runtime (so images build faster), I can move the install into an entrypoint script that only runs if `$AMP_HOME` is missing.

Now I could make those edits myself, but instead why don’t we dig further into automation and switch to `Agent` mode to let it do the following for me:

1. Validate that the containerfile is valid and builds
2. Update my Makefile to build it as well when I run `make`

We can see that when it wants to run commands, I’m prompted to allow for such interactivity on the system. I’m not bold enough to always allow, but to each their own.

![Screenshot 2025-10-28 at 8.18.11 PM.png](attachment:0b43acec-3316-421a-8cef-2afe8b0d4810:Screenshot_2025-10-28_at_8.18.11_PM.png)

I’ll skip the output of a Docker/Podman build, and show instead what it provided after:

> I validated the Containerfile by building it and updated your Makefile so `make` (and the `all` target) will build the new `ai-chat` domain.
>
> What I did
>
> - Added `containerFile/Containers/ai-chat/Containerfile` (installs Amp as a non-root user).
> - Built the image to validate the file builds.
> - Updated `containerFile/Makefile` to include an `ai-chat` target and add it to the `all` target.
>
> Notes, caveats, and suggestions
>
> - Build time & image size: because the Amp installer downloads Bun and runs the bootstrap at build time, the image is ~1.3 GB and the build takes a while. That’s expected.
> - Option: If you prefer faster rebuilds, move Amp installation into an entrypoint script that runs at container runtime only when `$AMP_HOME` is missing. That keeps image layers smaller and avoids re-downloading during iterative image builds.
> - UID matching: I added build args `USER/UID/GID` so you can pass `-build-arg UID=$(id -u) --build-arg GID=$(id -g)` if you want the container user to match your host user.
> - Secrets: keep API keys as runtime env vars (don’t bake them into the image).
>
> Would you like me to:
>
> - Add an entrypoint script to install Amp only at runtime (faster builds)?
> - Add a Makefile help target or a `make rebuild-no-cache` rule?
> - Add a build rule that passes host UID/GID automatically?

I liked the suggestion, so I asked it then to make the entrypoint run the script instead.

![Screenshot 2025-10-28 at 8.30.42 PM.png](attachment:67330b7a-633b-4ce0-993b-aeeb135c1cf9:Screenshot_2025-10-28_at_8.30.42_PM.png)

And, despite some of the ambiguity of my wording, Copilot didn’t make any edits I didn’t expect. It created a new Containerfile for ai-chat domain within my repo, updated the Makefile and provided additional suggestions. While in this process, I thought why not ask it for further ideas in `Ask` mode:

**How could I improve my process? How could I optimize my images if they are developer environments?**

> Good — here’s a practical, actionable guide to improving your developer-image process and shrinking / speeding up your Containerfiles. I’ll split this into (A) process & workflow improvements, (B) Containerfile optimization techniques you can apply immediately, and (C) concrete examples and commands you can copy into your repo.
>
> A — Process & workflow improvements
>
> - Separate concerns: keep a small, stable `base` image and build language/tool images that inherit from it. This reduces how often base layers change and speeds builds.
> - Build-time vs runtime installs: install only build-time tools during image build; defer large or transient installs (like Bun/amp) to an entrypoint (you already implemented this — good).
> - Use persistent volumes for caches & state: mount `~/.amp`, language caches (cargo, go, npm/pnpm, pip) from the host so you avoid repeated downloads across container runs.
> - Use explicit versioning: pin package/tool versions (or base image tags) so builds are reproducible.
> - Leverage layer caching & CI cache: push intermediate images or use a cache-from mechanism in CI to avoid full rebuilds.
> - Run as non-root and match UIDs: accept build args for UID/GID so files created by the container are owned on the host correctly.
> - Developer ergonomics: mount repo into `/work`, add a `dev` entrypoint that sets up PATH and envs, provide quick run scripts in repo (e.g., `make dev-shell`).
>
> B — Containerfile optimizations (practical rules)
>
> 1. Combine package manager steps into a single RUN to limit layers and remove caches:
>     - e.g. `RUN dnf install -y ... && dnf clean all && rm -rf /var/cache/dnf`
> 2. Don’t install large optional packages unless needed; disable weak deps where supported.
> 3. Use multi-stage builds for compiled artifacts (build toolchain in one stage, copy only runtime files into final image).
> 4. Defer heavy downloads to startup (entrypoint) for faster rebuilds; you already do this for Amp.
> 5. Use BuildKit cache mounts for language package caches when building (npm, pip, cargo, go). If you use BuildKit or docker buildx, add `-mount=type=cache` in a Dockerfile/Containerfile to persist package caches between builds.
> 6. Use an explicit `.dockerignore` / `.containerignore` to exclude large files from build context.
> 7. Keep images small by removing docs and unnecessary locales if applicable.
> 8. Create a “dev-tools” image with heavy tooling (clang, cmake, docker/gcloud CLI, large SDKs) that developers can `podman run` into — instead of baking everything into every language image.
> 9. Use build args for UID/GID & optional features (e.g., `-build-arg WITH_RUST=true`).
> 10. Tag images by semantic version / commit hash; store dev image variants separately from CI/production ones.
>
> C — Concrete examples and commands
>
> 1. Build-time UID/GID args — add to Makefile / build command
>
> - Build passing host UID/GID so the container user matches the host:
>
> ```
> # run from containerFile/
> podman build --build-arg UID=$(id -u) --build-arg GID=$(id -g) -t ai-chat:entrypoint Containers/ai-chat
>
> ```
>
> 1. Use BuildKit cache mounts (if you can enable BuildKit or buildx)
>
> - Example snippet (shows how to cache Python pip wheel cache and npm cache). Put into Containerfile replacing naive `pip install` / `npm ci` layers.
>
> ```
> # Example: using BuildKit mount to cache package manager artifacts (works with BuildKit-enabled builds)
> # pip example
> RUN --mount=type=cache,target=/root/.cache/pip \\
>     python3 -m pip install --no-cache-dir -r requirements.txt
>
> # npm example (assumes node base image)
> RUN --mount=type=cache,target=/root/.npm \\
>     npm ci --prefer-offline
>
> ```
>
> Note: you must enable BuildKit (Docker buildx or BuildKit-enabled podman) to use `--mount=type=cache`. If you don't use BuildKit, use host volumes for persistent caches.
>
> 1. Multi-stage build example for compiled tools (pattern)
>
> ```
> # Stage 1: build artifact
> FROM localhost/base:latest AS builder
> RUN dnf install -y gcc make cmake
> WORKDIR /src
> COPY . .
> RUN make release
>
> # Stage 2: runtime
> FROM localhost/base:latest
> COPY --from=builder /src/bin/myapp /usr/local/bin/myapp
> ENTRYPOINT ["/usr/local/bin/myapp"]
>
> ```
>
> 1. Small improvements for DNF-based Containerfiles
>
> - Consolidate installs and clear caches:
>
> ```
> RUN dnf -y update && \\
>     dnf -y install --setopt=install_weak_deps=False curl wget tar unzip ca-certificates bash && \\
>     dnf clean all && rm -rf /var/cache/dnf
>
> ```
>
> (If `--setopt=install_weak_deps=False` is not desired, omit it; it reduces extra recommended packages.)
>
> 1. .containerignore
>
> - Add a `.containerignore` or `.dockerignore` in `containerFile/` to skip `.git`, local node_modules, models, large artifacts:
>
> ```
> .git
> node_modules
> dist
> *.log
> models/
> .cache/
>
> ```
>
> 1. Entrypoint pattern (you already have one) — options:
>
> - Add retry logic and a short timeout for the network install (helpful on flaky networks).
> - Add an environment variable to skip runtime install when CI builds or for ephemeral containers: e.g. `SKIP_AMP_INSTALL=1`.
>
> 1. CI cache strategy
>
> - In CI, push a cache image (e.g. `ai-chat:cache`) with built dependencies used as `-cache-from` or as a base for subsequent builds. This speeds builds in CI.
>
> Recommendations prioritized (what to do next)
>
> 1. Add `.containerignore` to all container folders.
> 2. Keep the heavy tool installation in a separate dev-tools image if many devs share the same machine.
> 3. Use host-mounted caches for long-lived state (cargo registry, npm, pip cache) while developing.
> 4. If you have CI that builds images frequently, set up a build cache (push interim images or enable BuildKit caching).
> 5. Add `-build-arg UID=$(id -u)` and `GID` to Makefile build lines so images produce correct file ownership by default.
>
> Want me to:
>
> - Add `.containerignore` to the repo?
> - Update your Makefile to pass UID/GID automatically (I can apply that change)?
> - Add BuildKit-ready cache-mount snippets adapted to the languages in your repo (Node, Python, Rust, Go)?
>   Tell me which of those and I’ll make the edits.

So I let it implement some, providing the prompt:

1. **I love the idea of a small base image**
2. **Disable weak dependencies where supported**
3. **Use multi-stage builds where applicable**
4. **Use .containerignore files appropriately**
5. **Run as non-root by default -if you believe it is wise for a local developer env**

**Also, write a @README.md file that explains the current process and usage**

From there, it went on to change most of the files and proceeded to do a sanity check using `make` in the terminal, only for me to see the following build error:

![Screenshot 2025-10-28 at 8.56.44 PM.png](attachment:038157fa-6aca-48e6-afe1-fe65468167d6:Screenshot_2025-10-28_at_8.56.44_PM.png)

```docker
The requested operation requires superuser privileges. Please log in as a user with elevated rights, or use the "--assumeno" or "--downloadonly" options to run the command without modifying the system state.
```

So I wondered, do I really not need root afterall? It’s a localized dev env that I want to use to keep the host machine as clean as possible. So I sent my agent that error, and mentioned at the end, **I think the user should just be root since this is a localized dev env with minimal mounts**

> Done — I changed the base image to run as `root` by default, while keeping the `dev` user present for interactive use. This makes local developer workflows simpler when elevated privileges are required inside the container.
>
> What I changed
>
> - Updated `containerFile/Containers/base-fedora/Containerfile` to:
>     - install minimal tooling in a single consolidated layer (weak deps disabled),
>     - create a `dev` user (configurable with build args),
>     - default to `USER root` (so containers start as root unless overridden).
> - All other changes you asked for (entrypoints, .containerignore, README, other Containerfiles) are already in place.

It proceeded to do a new `make` sanity check and this time, all built without any issue:

```docker
REPOSITORY                         TAG         IMAGE ID      CREATED             SIZE
localhost/ai-chat                  latest      32fb9879da46  About a minute ago  873 MB
localhost/node                     latest      aa8a60dab28a  About a minute ago  867 MB
localhost/kubernetes               latest      20910426593a  About a minute ago  1.77 GB
localhost/python                   latest      a8c28fe319fd  2 minutes ago       861 MB
localhost/zig                      latest      4a34aa08a518  2 minutes ago       1.39 GB
localhost/rust                     latest      1e0027b5f3ff  2 minutes ago       838 MB
localhost/go                       latest      126f3e562a73  3 minutes ago       982 MB
localhost/base                     latest      fb61cbf1a5cb  3 minutes ago       773 MB
<none>                             <none>      49d76f8f257f  6 minutes ago       838 MB
<none>                             <none>      bd952529b508  7 minutes ago       982 MB
registry.fedoraproject.org/fedora  42          1380f3d82dbf  19 hours ago        181 MB
```
