# Local LLaMA Server Setup Documentation

_TL;DR_: End-to-end documentation to set up your own local & fully private LLM server on Debian. Equipped with chat, web search, RAG, model management, MCP servers, image generation, and TTS, along with steps for configuring SSH, firewall, and secure remote access via Tailscale.

Software Stack:

- Inference Engine ([Ollama](https://github.com/ollama/ollama), [llama.cpp](https://github.com/ggml-org/llama.cpp), [vLLM](https://github.com/vllm-project/vllm))
- Search Engine ([SearXNG](https://github.com/searxng/searxng))
- Model Server ([llama-swap](https://github.com/mostlygeek/llama-swap), `systemd` service)
- Chat Platform ([Open WebUI](https://github.com/open-webui/open-webui))
- MCP Proxy Server ([mcp-proxy](https://github.com/sparfenyuk/mcp-proxy), [MCPJungle](https://github.com/mcpjungle/MCPJungle))
- Text-to-Speech Server ([Kokoro FastAPI](https://github.com/remsky/Kokoro-FastAPI))
- Image Generation Server ([ComfyUI](https://github.com/comfyanonymous/ComfyUI))
- VPN/Remote Access ([Tailscale](https://tailscale.com))
- Reverse Proxy ([Caddy](https://github.com/caddyserver/caddy))
- DNS Provider ([Cloudflare](https://www.cloudflare.com))

![Software Stack Architectural Diagram](media/llm-server-architecture.png)

## Table of Contents

- [Local LLaMA Server Setup Documentation](#local-llama-server-setup-documentation)
  - [Table of Contents](#table-of-contents)
  - [About](#about)
  - [Priorities](#priorities)
  - [Prerequisites](#prerequisites)
  - [General](#general)
    - [Allow `sudo` Permissions](#allow-sudo-permissions)
    - [Update System Packages](#update-system-packages)
    - [Schedule Startup Script](#schedule-startup-script)
    - [Configure Script Permissions](#configure-script-permissions)
    - [Configure Auto-Login (optional)](#configure-auto-login-optional)
  - [Docker](#docker)
    - [Add User to Docker Group](#add-user-to-docker-group)
    - [Nvidia Container Toolkit](#nvidia-container-toolkit)
    - [Create a Network](#create-a-network)
    - [Harden Docker Containers](#harden-docker-containers)
    - [Helpful Commands](#helpful-commands)
  - [HuggingFace CLI](#huggingface-cli)
    - [Manage Models](#manage-models)
    - [Download Models](#download-models)
    - [Delete Models](#delete-models)
  - [Search Engine](#search-engine)
    - [SearXNG](#searxng)
    - [Open WebUI Integration](#open-webui-integration)
  - [Inference Engine](#inference-engine)
    - [Ollama](#ollama)
    - [llama.cpp](#llamacpp)
    - [vLLM](#vllm)
  - [Model Server](#model-server)
    - [llama-swap](#llama-swap)
    - [`systemd` Service](#systemd-service)
    - [Open WebUI Integration](#open-webui-integration-1)
      - [llama-swap](#llama-swap-1)
      - [`systemd` Service](#systemd-service-1)
  - [Chat Platform](#chat-platform)
    - [Open WebUI](#open-webui)
  - [MCP Proxy Server](#mcp-proxy-server)
    - [mcp-proxy](#mcp-proxy)
    - [MCPJungle](#mcpjungle)
    - [Comparison](#comparison)
    - [Open WebUI Integration](#open-webui-integration-2)
      - [mcp-proxy](#mcp-proxy-1)
      - [MCPJungle](#mcpjungle-1)
    - [VS Code/Claude Desktop Integration](#vs-codeclaude-desktop-integration)
  - [Text-to-Speech Server](#text-to-speech-server)
    - [Kokoro FastAPI](#kokoro-fastapi)
    - [Open WebUI Integration](#open-webui-integration-3)
  - [Image Generation Server](#image-generation-server)
    - [ComfyUI](#comfyui)
    - [Open WebUI Integration](#open-webui-integration-4)
  - [SSH](#ssh)
  - [Firewall](#firewall)
  - [Remote Access](#remote-access)
  - [Updating](#updating)
    - [General](#general-1)
    - [Nvidia Drivers \& CUDA](#nvidia-drivers--cuda)
    - [Docker Services](#docker-services)
    - [Ollama](#ollama-1)
    - [llama.cpp](#llamacpp-1)
    - [vLLM](#vllm-1)
    - [ComfyUI](#comfyui-1)
  - [Troubleshooting](#troubleshooting)
    - [Docker](#docker-1)
    - [`ssh`](#ssh-1)
    - [Nvidia Drivers](#nvidia-drivers)
    - [Ollama](#ollama-2)
    - [vLLM](#vllm-2)
    - [Open WebUI](#open-webui-1)
  - [Monitoring](#monitoring)
  - [Notes](#notes)
    - [Software](#software)
    - [Hardware](#hardware)
  - [References](#references)
  - [Acknowledgements](#acknowledgements)

## About

This repository outlines the steps to run a server for running local language models. It uses Debian specifically, but most Linux distros should follow a very similar process. It aims to be a guide for Linux beginners like me who are setting up a server for the first time.

The process involves installing the requisite drivers, setting the GPU power limit, setting up auto-login, and scheduling the `init.bash` script to run at boot. All these settings are based on my ideal setup for a language model server that runs most of the day but a lot can be customized to suit your needs.

> [!IMPORTANT]
> No part of this guide was written using AI - any hallucinations are the good old human kind. AI is used to format Markdown elements and vet for any technical inaccuracies. While I've done my absolute best to ensure correctness in every step/command, check **everything** you execute in a terminal. Enjoy!

## Priorities

- **Simplicity**: It should be relatively straightforward to set up the components of the solution.
- **Stability**: The components should be stable and capable of running for weeks at a time without any intervention necessary.
- **Security**: The components should be able to be tightly secured and limited in their capability to damage the system in case of a known vulnerability affecting any of the components.
- **Maintainability**: The components and their interactions should be uncomplicated enough that you know enough to maintain them as they evolve (because they *will* evolve).
- **Aesthetics**: The result should be as close to a cloud provider's chat platform as possible. A homelab solution doesn't necessarily need to feel like it was cobbled together haphazardly.
- **Modularity**: Components in the setup should be able to be swapped out for newer/more performant/better maintained alternatives easily. Standard protocols (OpenAI-compatibility, MCPs, etc.) help with this a lot and, in this guide, they are always preferred over bundled solutions.
- **Open source**: The code should be able to be verified by a community of engineers. Chat platforms and LLMs involve large amounts of personal data conveyed in natural language and it's important to know that data isn't going outside your machine.

## Prerequisites

Any modern CPU and GPU combination should work for this guide. Previously, compatibility with AMD GPUs was an issue but the latest releases of Ollama have worked through this and [AMD GPUs are now supported natively](https://ollama.com/blog/amd-preview). 

For reference, this guide was built around the following system:
- **CPU**: Intel Core i5-12600KF
- **Memory**: 96GB 3200MHz DDR4 RAM
- **Storage**: 1TB M.2 NVMe SSD
- **GPU**: 2x Nvidia RTX 3090 (24GB)

> [!NOTE]
> **AMD GPUs**: Power limiting is skipped for AMD GPUs as [AMD has recently made it difficult to set power limits on their GPUs](https://www.reddit.com/r/linux_gaming/comments/1b6l1tz/no_more_power_limiting_for_amd_gpus_because_it_is/). Naturally, skip any steps involving `nvidia-smi` or `nvidia-persistenced` and the power limit in the `init.bash` script.
> 
> **CPU-only**: You can skip the GPU driver installation and power limiting steps. The rest of the guide should work as expected.

> [!NOTE]
> This guide uses `~/` (or `/home/<username>`) as the base directory. If you're working in different directory, please modify all your commands accordingly.

To begin the process of setting up your server, you will need the following:

- Fresh install of Debian
- Internet connection
- Basic understanding of the Linux terminal
- Peripherals like a monitor, keyboard, and mouse

To install Debian on your newly built server hardware:

- Download the [Debian ISO](https://www.debian.org/distrib/) from the official website.
- Create a bootable USB using a tool like [Rufus](https://rufus.ie/en/) for Windows or [Balena Etcher](https://etcher.balena.io) for MacOS.
- Boot into the USB and install Debian.

For a more detailed guide on installing Debian, refer to the [official documentation](https://www.debian.org/releases/buster/amd64/). For those who aren't yet experienced with Linux, I recommend using the graphical installer - you will be given an option between the text-based installer and graphical installer. 

I also recommend installing a lightweight desktop environment like XFCE for ease of use. Other options like GNOME or KDE are also available - GNOME may be a better option for those using their server as a primary workstation as it is more feature-rich (and, as such, heavier) than XFCE.

## General

### Allow `sudo` Permissions

To do a bunch of things in this guide, we need to be root. But we don't have the ability to act as root until root allows our user to. First, we'll switch to root and grant our user permission to run commands with `sudo`.

> [!TIP]
> `sudo` stands for "superuser doer" - in Linux, it signals to the OS that you want the command you're running to be running on behalf of the root user. This should be used sparingly on a highly secure system (we want user-specific permissions for most processes) and carefully since it can affect the system with the same power as the root user. Don't overthink its use or worry when using it - just know that it **can** be dangerous if used incorrectly.

- Switch to root:
    ```bash
    su root
    ```

- Run the following command to add your user to the `sudo` group (which has the permissions we're looking for):
    ```bash
    sudo usermod -a -G sudo <username>
    ```
    > Replace `<username>` with your username.

    Save and exit (`Ctrl+X`).
- Close your existing terminal and open a new session. This is required to see the changes.
- (Optional) Test your new permissions by running `ls` (command to view the contents of a directory) with `sudo`:
    ```bash
    sudo ls
    ```

### Update System Packages

- Update the system by running the following commands:
    ```
    sudo apt update
    sudo apt upgrade
    ```

Now, we'll install the required GPU drivers that allow programs to utilize their compute capabilities.

**Nvidia GPUs**
- Follow Nvidia's [guide on downloading CUDA Toolkit](https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&Distribution=Debian). The instructions are specific to your machine and the website will lead you to them interactively.
- Run the following commands:
    ```bash
    sudo apt install linux-headers-amd64
    sudo apt install nvidia-driver firmware-misc-nonfree
    ```
- Reboot the server.
- Run the following command to verify the installation:
    ```bash
    nvidia-smi
    ```
  
**AMD GPUs**
- Run the following commands:
    ```bash
    deb http://deb.debian.org/debian bookworm main contrib non-free-firmware
    apt install firmware-amd-graphics libgl1-mesa-dri libglx-mesa0 mesa-vulkan-drivers xserver-xorg-video-all
    ```
- Reboot the server.

We'll also install some packages that are not installed on Debian by default but may be required later:
```
sudo apt install libcurl cmake build-essential
```

### Schedule Startup Script

In this step, we'll create a script called `init.bash`. This script will be run at boot to set the GPU power limit and start the server using Ollama. We set the GPU power limit lower because it has been seen in testing and inference that there is only a 5-15% performance decrease for a 30% reduction in power consumption. This is especially important for servers that are running 24/7.

- Run the following commands:
    ```bash
    touch init.bash
    nano init.bash
    ```
- Add the following lines to the script:
    ```bash
    #!/bin/bash
    sudo nvidia-smi -pm 1
    sudo nvidia-smi -pl <power_limit>
    ```
    > Replace `<power_limit>` with the desired power limit in watts. For example, `sudo nvidia-smi -pl 250`.

    For multiple GPUs, modify the script to set the power limit for each GPU:
    ```bash
    sudo nvidia-smi -i 0 -pl <power_limit>
    sudo nvidia-smi -i 1 -pl <power_limit>
    ```
- Save and exit the script.
- Make the script executable:
    ```bash
    chmod +x init.bash
    ```

Adding the `init.bash` script to the crontab will schedule it to run at boot.

- Run the following command:
    ```bash
    crontab -e
    ```
- Add the following line to the file:
    ```bash
    @reboot /path/to/init.bash
    ```
    > Replace `/path/to/init.bash` with the path to the `init.bash` script.

- (Optional) Add the following line to shutdown the server at 12am:
    ```bash
    0 0 * * * /sbin/shutdown -h now
    ```
- Save and exit the file.

### Configure Script Permissions

We want `init.bash` to run the `nvidia-smi` commands without having to enter a password. This is done by giving `nvidia-persistenced` and `nvidia-smi` passwordless `sudo` permissions, and can be achieved by editing the `sudoers` file.

AMD users can skip this step as power limiting is not supported on AMD GPUs.

- Run the following command to edit the sudoers file:
    ```bash
    sudo visudo
    ```
- Add the following lines to the file:
    ```
    <username> ALL=(ALL) NOPASSWD: /usr/bin/nvidia-persistenced
    <username> ALL=(ALL) NOPASSWD: /usr/bin/nvidia-smi
    ```
    > Replace `<username>` with your username.
- Save and exit the file.

> [!IMPORTANT]
> Ensure that you add these lines AFTER `%sudo ALL=(ALL:ALL) ALL`. The order of the lines in the file matters - the last matching line will be used so if you add these lines before `%sudo ALL=(ALL:ALL) ALL`, they will be ignored.

### Configure Auto-Login (optional)

When the server boots up, we may want it to automatically log in to a user account and run the `init.bash` script. This is done by configuring the `lightdm` display manager.

- Run the following command:
    ```bash
    sudo nano /etc/lightdm/lightdm.conf
    ```
- Find the following commented line. It should be in the `[Seat:*]` section.
    ```
    # autologin-user=
    ```
- Uncomment the line and add your username:
    ```
    autologin-user=<username>
    ```
    > Replace `<username>` with your username.
- Save and exit the file.

## Docker

📖 [**Documentation**](https://docs.docker.com/engine/)

Docker is a containerization platform that allows you to run applications in isolated environments. This subsection follows [Docker's guide](https://docs.docker.com/engine/install/debian/) to install Docker Engine on Debian. The commands are listed below, but visiting the guide is recommended in case instructions have changed.

- If you already have a Docker installation on your system, it's a good idea to re-install so there are no broken/out-of-date dependencies. The command below will iterate through your system's installed packages and remove the ones associated with Docker.
    ```bash
    for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt purge $pkg; done
    ```

- Run the following commands:
    ```bash
    # Add Docker's official GPG key:
    sudo apt update
    sudo apt install ca-certificates curl
    sudo install -m 0755 -d /etc/apt/keyrings
    sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
    sudo chmod a+r /etc/apt/keyrings/docker.asc

    # Add the repository to Apt sources:
    sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
    Types: deb
    URIs: https://download.docker.com/linux/debian
    Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
    Components: stable
    Signed-By: /etc/apt/keyrings/docker.asc
    EOF

    sudo apt update
    ```
- Install the Docker packages:
    ```bash
    sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    ```
- Verify the installation (optional):
    ```bash
    sudo systemctl status docker
    ```

    If it hasn't started, try manually starting the daemon:
    ```bash
    sudo systemctl start docker
    ```

### Add User to Docker Group

So that we can forgo the use of `sudo` for Docker commands and run them directly under our user, we will add our user to the `docker` group. This isn't strictly necessary but convenient. This sub-section uses [Docker's own post-install documentation](https://docs.docker.com/engine/install/linux-postinstall/) as a reference - have a look just in case things have changed since the time of writing.

> [!CAUTION]
> Being in the `docker` group grants significant system access - effectively root-equivalent privileges to the Docker daemon. Only add trusted users to this group.

1. Add your user to the Docker group:
    ```bash
    sudo usermod -aG docker $USER
    ```

2. Apply the changes:
    ```bash
    newgrp docker
    ```
    
3. Verify your new permissions by checking running containers:
    ```bash
    docker ps -a
    ```

If you encounter "permission denied" and Docker commands still require sudo, log out and log back in.

> [!NOTE]
> If you decide not to do this, you will need to append `sudo` in front of any `docker` commands.

### Nvidia Container Toolkit

You will most likely want to use GPUs via Docker - this will require Nvidia Container Toolkit, which allows Docker to allocate/de-allocate memory on Nvidia GPUs. The steps for installing this are listed below, but it is recommended to reference [Nvidia's documentation](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) for the most up-to-date commands.

1. Configure the repository
    ```bash
    curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
    && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
        sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
        sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
    ```

2. Update packages:
    ```bash
    sudo apt update
    ```

3. Install Nvidia Container Toolkit packages:
    ```bash
    sudo apt install -y nvidia-container-toolkit
    ```

4. Restart the Docker daemon:
    ```bash
    sudo systemctl restart docker
    ```

### Create a Network

We'll be running most services via Docker containers. To allow multiple containers to communicate with each other, we can open up ports via UFW (which we'll [configure later](#firewall)) but this is less optimal than creating a Docker network. This way, all containers on the network can securely talk to each other without needing to open ports for the many services we could run via UFW, inherently creating a more secure setup.

We'll call this network `app-net`: you can call it anything you like, just be sure to update the commands that use it later.

Run the following:

```bash
docker network create app-net
```

That's it! Now, when we create containers, we can reference it as follows:

**Docker Run**
```bash
docker run <container> --network app-net
```

**Docker Compose**
```yaml
services:
  <container>:
    # add this
    networks:
      - app-net

# add this
networks:
  app-net:
    external: true
```

> Replace `<container>` with the actual service - don't forget to add other parameters too.

With this configured, we can now call containers by name and port. Let's pretend we're calling the `/health` endpoint in `llama-swap` from `open-webui` (two actual containers we'll be creating later on) to ensure that the containers can see and speak to each other. Run (`CTRL+C` to quit):

```bash
docker exec -i open-webui curl http://llama-swap:8080/health
```

You could also run it the other way to be extra sure:

```bash
docker exec -it llama-swap curl http://open-webui:8080
```

> [!IMPORTANT]
> The port is always the **internal port** the container is running on. If a container runs on 1111:8080, for example, 1111 is the port on the host (where you might access it, like `http://localhost:1111` or `http://<server_ip>:1111`) and 8080 is the internal port the container is running on. Thus, trying to access the container on 1111 via `app-net` will not work. Remembering this when specifying URLs in services will save you a lot of unnecessary "why isn't this working?" pains.

### Harden Docker Containers

Docker security is is **critical** for public-facing services. It involves reducing attack surfaces, eliminating terminal abilities for containers, and limiting resource usage. Even if you only intend to access your services via a private network (preferred if possible), shoring up security is never a bad idea because it limits how much damage can be done to your server and its services if it ever was to become compromised somehow.

📖 See the full guide at [Docker Security](docs/docker-security.md). 

> [!TIP]
> The combined example at the end provides a ready-to-paste YAML block with all hardening directives. You can copy it directly into your existing Compose files and customize as needed.

### Helpful Commands

In the process of setting up this server (or anywhere down the rabbit hole of setting up services), you're likely to use Docker often. For the uninitiated, here are helpful commands that will make navigating and troubleshooting containers easier:

- See available/running containers: `docker ps -a`
- Restart a container: `docker restart <container_name>`
- View a container's logs (live): `docker logs -f <container_name>` (`CTRL+C` to quit)
- Rename a container: `docker rename <container_name> <new_container_name>`
- Sometimes, a single service will spin up multiple containers, e.g. `xyz-server` and `xyz-db`. To restart both simultaneously, run the following command **from inside the directory containing the Compose file**: `docker compose restart`
- Recreate a service: `docker compose down && docker compose up -d`
- Test a container's configuration: `docker compose config --quiet`
- View a container's resolved configuration: `docker compose config`
- List Docker networks: `docker network ls`

> [!TIP]
> There are no set rules when it comes to how you set up your Docker containers/services. However, here are my two cents:  
> It's cleanest to use Docker Compose (`docker compose up -d` with a `docker-compose.yaml` file as opposed to `docker run -d <image_name>`). Unless you take copious notes on your homelab and its setup, this method is almost self-documenting and keeps a neat trail of the services you run via their compose files. One compose file per directory is standard.

> [!TIP]
> Occasionally, restarting a container is not enough to persist changes to configurations. If ever you find that changes you made to a service are not showing up, recreate the service altogether first before continuing to other troubleshooting.

## HuggingFace CLI

📖 [**Documentation**](https://huggingface.co/docs/huggingface_hub/main/en/guides/cli)

HuggingFace is the leading open-source ML/AI platform - it hosts models (including LLMs), datasets, and demo apps that can be used to test models. For the purpose of this guide, we'll be using HuggingFace to download popular open-source LLMs. 

> [!NOTE]
> Only needed for llama.cpp/vLLM.

- Create a new virtual environment:
    ```bash
    python3 -m venv hf-env
    source hf-env/bin/activate
    ```
- Download the `huggingface_hub` package using `pip`:
    ```bash
    pip install -U "huggingface_hub[cli]"
    ```
- Create an authentication token on https://huggingface.com
- Log in to HF Hub:
    ```bash
    hf auth login
    ```
- Enter your token when prompted.
- Run the following to verify your login:
    ```bash
    hf auth whoami
    ```

    The result should be your username.

### Manage Models

Models can be downloaded either to the default location (`.cache/huggingface/hub`) or to any local directory you specify. Where the model is stored can be defined using the `--local-dir` command line flag. Not specifying this will result in the model being stored in the default location. Storing the model in the folder where the packages for the inference engine are stored is good practice - this way, everything you need to run inference on a model is stored in the same place. However, if you use the same models with multiple backends frequently (e.g. using Qwen_QwQ-32B-Q4_K_M.gguf with both llama.cpp and vLLM), either set a common model directory or use the default HF option without specifying this flag.

First, activate the virtual environment that contains `huggingface_hub`:
```
source hf-env/bin/activate
```

### Download Models

Models are downloaded using their HuggingFace tag. Here, we'll use bartowski/Qwen_QwQ-32B-GGUF as an example. To download a model, run:
```
hf download bartowski/Qwen_QwQ-32B-GGUF Qwen_QwQ-32B-Q4_K_M.gguf --local-dir models
```
Ensure that you are in the correct directory when you run this.

### Delete Models

To delete a model in the specified location, run:
```
rm <model_name>
```

To delete a model in the default location, run:
```
hf delete-cache
```

This will start an interactive session where you can remove models from the HuggingFace directory. In case you've been saving models in a different location than `.cache/huggingface`, deleting models from there will free up space but the metadata will remain in the HF cache until it is deleted properly. This can be done via the above command but you can also simply delete the model directory from `.cache/huggingface/hub`.

## Search Engine

> [!NOTE]
> This step is optional but highly recommended for grounding LLMs with relevant search results from reputable sources. Targeted web searches via MCP tool calls make reports generated by LLMs much less prone to random hallucinations.

### SearXNG

🌟 [**GitHub**](https://github.com/searxng/searxng)  
📖 [**Documentation**](https://docs.searxng.org)  

To power our search-based workflows, we don't want to rely on a search provider that can monitor searches. While using any search engine has this problem, metasearch engines like SearXNG mitigate it to a decent degree. SearXNG aggregates results from over 245 [search services](https://docs.searxng.org/user/configured_engines.html#configured-engines) and does not track/profile users. You can use a hosted instance on the Internet but, considering the priorities of this guide and how trivial it is to set one up, we'll be spinning up our own instance on port 5050.

1. Start the container:
    ```bash
    docker pull searxng/searxng
    export PORT=5050
    docker run \
        -d -p ${PORT}:8080 \
        --name searxng \
        --network app-net \
        -v "${PWD}/searxng:/etc/searxng" \
        -e "BASE_URL=http://0.0.0.0:$PORT/" \
        -e "INSTANCE_NAME=searxng" \
        --restart unless-stopped \
        searxng/searxng
    ```

2. Edit `settings.yml` to include JSON format support:
    ```bash
    sudo nano searxng/settings.yml
    ```

    Add the following:
    ```yaml
    search:
      # ...other parameters here...
      formats:
        - html
        - json      # add this line
    ```

3. Restart the container with `docker restart searxng`

### Open WebUI Integration

In case you want a simple web search workflow and want to skip MCP servers/agentic setups, Open WebUI supports web search functionality natively. Navigate to `Admin Panel > Settings > Web Search` and set the following values:

- Enable `Web Search`
- Web Search Engine: `searxng`
- Searxng Query URL: `http://searxng:8080/search?q=<query>`
- API Key: `anything-you-like`

## Inference Engine

The inference engine is the software layer that loads model weights and manages text generation. This guide supports three popular engines: Ollama, llama.cpp, and vLLM. Each offers different trade-offs between ease of use, control, and performance.

If you're unsure where to begin and want some more context, see a detailed comparison in [Comparing Inference Engines](docs/inference-engine-comparison.md). It goes into what inference is, how the engines tackle it differently, and what the best fit for you might be.

### Ollama

🌟 [**GitHub**](https://github.com/ollama/ollama)  
📖 [**Documentation**](https://github.com/ollama/ollama/tree/main/docs)  
🔧 [**Engine Arguments**](https://github.com/ollama/ollama/blob/main/docs/modelfile.md)

Ollama will be installed as a service, so it runs automatically at boot.

- Download Ollama from the official repository:
    ```
    curl -fsSL https://ollama.com/install.sh | sh
    ```

We want our API endpoint to be reachable by the rest of the LAN. For Ollama, this means setting `OLLAMA_HOST=0.0.0.0` in the `ollama.service`.

- Run the following command to edit the service:
    ```
    systemctl edit ollama.service
    ```
- Find the `[Service]` section and add `Environment="OLLAMA_HOST=0.0.0.0"` under it. It should look like this:
    ```
    [Service]
    Environment="OLLAMA_HOST=0.0.0.0"
    ```
- Save and exit.
- Reload the environment.
    ```
    systemctl daemon-reload
    systemctl restart ollama
    ```

> [!TIP]
> If you installed Ollama manually or don't use it as a service, remember to run `ollama serve` to properly start the server. Refer to [Ollama's troubleshooting steps](#ollama-3) if you encounter an error.

### llama.cpp

🌟 [**GitHub**](https://github.com/ggml-org/llama.cpp)  
📖 [**Documentation**](https://github.com/ggml-org/llama.cpp/tree/master/docs)  
🔧 [**Engine Arguments**](https://github.com/ggml-org/llama.cpp/tree/master/examples/server)

> [!TIP]
> This is my recommended inference engine. My biggest reasons are
> 1. the `--fit` flag and its ability to automatically load any model by calculating offload requirements based on available system resources, and
> 2. insanely quick model load times, especially compared to vLLM.

- Clone the llama.cpp GitHub repository:
    ```
    git clone https://github.com/ggml-org/llama.cpp.git
    cd llama.cpp
    ```
- Build the binary:

    **CPU**
    ```
    cmake -B build
    cmake --build build --config Release -j <threads>
    ```

    **CUDA**
    ```
    cmake -B build -DGGML_CUDA=ON
    cmake --build build --config Release -j <threads>
    ```
    > Replace `<threads>` with `[number of threads your CPU has] - 2`. This offers a massive boost to the build process as it leverages your system resources better.

For other systems looking to use Metal, Vulkan and other low-level graphics APIs, view the complete [llama.cpp build documentation](https://github.com/ggml-org/llama.cpp/blob/master/docs/build.md) to leverage accelerated inference.

### vLLM

🌟 [**GitHub**](https://github.com/vllm-project/vllm)  
📖 [**Documentation**](https://docs.vllm.ai/en/stable/index.html)  
🔧 [**Engine Arguments**](https://docs.vllm.ai/en/stable/serving/engine_args.html)

vLLM comes with its own OpenAI-compatible API that we can use just like Ollama. Where Ollama runs GGUF model files, vLLM can run AWQ, GPTQ, GGUF, BitsAndBytes, and safetensors (the default release type) natively.

**Manual Installation (Recommended)**

- Create a directory and virtual environment for vLLM:
    ```
    mkdir vllm
    cd vllm
    python3 -m venv .venv
    source .venv/bin/activate
    ```

- Install vLLM using `pip`:
    ```
    pip install vllm
    ```

- Serve with your desired flags. It uses port 8000 by default, but I'm using port 8556 here so it doesn't conflict with any other services:
    ```
    vllm serve <model> --port 8556
    ```

- To use as a service, add the following block to `init.bash` to serve vLLM on startup:
    ```
    source .venv/bin/activate
    vllm serve <model> --port 8556
    ```
    > Replace `<model>` with your desired model tag, copied from HuggingFace.

**Docker Installation**

- Run:
    ```
    docker run --gpus all \
    -v ~/.cache/huggingface:/root/.cache/huggingface \
    --env "HUGGING_FACE_HUB_TOKEN=<your_hf_hub_token>" \
    -p 8556:8000 \
    --ipc=host \
    vllm/vllm-openai:latest \
    --model <model>
    ```
    > Replace `<your_hf_hub_token>` with your HuggingFace Hub token and `<model>` with your desired model tag, copied from HuggingFace.

To serve a different model:

- First stop the existing container:
    ```
    docker ps -a
    docker stop <vllm_container_ID>
    ```

- If you want to run the exact same setup again in the future, skip this step. Otherwise, run the following to delete the container and not clutter your Docker container environment:
    ```
    docker rm <vllm_container_ID>
    ```

- Rerun the Docker command from the installation with the desired model.
    ```
    docker run --gpus all \
    -v ~/.cache/huggingface:/root/.cache/huggingface \
    --env "HUGGING_FACE_HUB_TOKEN=<your_hf_hub_token>" \
    -p 8556:8000 \
    --ipc=host \
    vllm/vllm-openai:latest \
    --model <model>
    ```

## Model Server

> [!NOTE]
> Only needed for manual installations of llama.cpp/vLLM. Ollama manages model management via its CLI.

While the above steps will help you get up and running with an OpenAI-compatible LLM server, they will not help with this server persisting after you close your terminal window or restart your physical server. They also won't allow a chat platform to reliably reference and swap between the various models you have available - a likely use-case in a landscape where different models specialize in different tasks. Running the inference engine via Docker can achieve this persistence with the `-d` (for "detach") flag but (i) services like llama.cpp and vLLM are usually configured without Docker and (ii) it can't swap models on-demand. This necessitates a server that can manage loading/unloading, swapping, and listing available models.

### llama-swap

🌟 [**GitHub**](https://github.com/mostlygeek/llama-swap)  
📖 [**Documentation**](https://github.com/mostlygeek/llama-swap/wiki)  
📖 [**Configuration**](https://github.com/mostlygeek/llama-swap/wiki/Configuration)

> [!TIP]
> This is my recommended way to run llama.cpp/vLLM models.

llama-swap is a lightweight proxy server for LLMs that solves our pain points from above. It's an extremely configurable tool that allows a single point of entry for models from various backends. Models can be set up in groups, listed/unlisted easily, configured with customized hyperparameters, and monitored using streamed logs in the llama-swap web UI.

In the installation below, we'll use `Qwen3-4B-Instruct-2507-UD-Q4_K_XL.gguf` for llama.cpp and `Qwen/Qwen3-4B-Instruct-2507` for vLLM. We'll also use port 7000 to serve the models on.

1. Create a new directory with the `config.yaml` file:
    ```bash
    sudo mkdir llama-swap
    cd llama-swap
    sudo nano config.yaml
    ```

2. Enter the following and save:

    **llama.cpp**
    ```yaml
    models:
      "qwen3-4b":
        proxy: "http://127.0.0.1:7000"
        cmd: |
          /app/llama-server
          -m /models/Qwen3-4B-Instruct-2507-UD-Q4_K_XL.gguf
          --port 7000 --host 0.0.0.0
    ```

    **vLLM (Docker)**
    ```yaml
    models:
      "qwen3-4b":
        proxy: "http://127.0.0.1:7000"
        cmd: |
          docker run --name qwen-vllm
          --init --rm -p 7000:8080 --ipc=host
          vllm/vllm-openai:latest
          -m /models/Qwen/Qwen3-4B-Instruct-2507
        cmdStop: docker stop qwen-vllm
    ```

    **vLLM (local)**:
    ```yaml
    models:
      "qwen3-4b":
        proxy: "http://127.0.0.1:7000"
        cmd: |
          source /app/vllm/.venv/bin/activate &&
          /app/vllm/.venv/bin/vllm serve
          -m /models/Qwen/Qwen3-4B-Instruct-2507
          --port 7000 --host 0.0.0.0
        cmdStop: pkill -f "vllm serve"
    ```

3. Install the container:
    
    We use the `cuda` tag here, but llama-swap offers `cpu`, `intel`, `vulkan`, and `musa` tags as well. Releases can be found [here](https://github.com/mostlygeek/llama-swap/pkgs/container/llama-swap).

    **llama.cpp**
    ```bash
    docker run -d --gpus all --restart unless-stopped --network app-net --pull=always --name llama-swap -p 9292:8080 \
    -v /path/to/models:/models \
    -v /home/<username>/llama-swap/config.yaml:/app/config.yaml \
    -v /home/<username>/llama.cpp/build/bin/llama-server:/app/llama-server \
    ghcr.io/mostlygeek/llama-swap:cuda
    ```

    **vLLM (Docker/local)**
    ```bash
    docker run -d --gpus all --restart unless-stopped --network app-net --pull=always --name llama-swap -p 9292:8080 \
    -v /path/to/models:/models \
    -v /home/<username>/vllm:/app/vllm \
    -v /home/<username>/llama-swap/config.yaml:/app/config.yaml \
    ghcr.io/mostlygeek/llama-swap:cuda
    ```

    > Replace `<username>` with your actual username and `/path/to/models` with the path to your model files.

> [!NOTE]
> llama-swap prefers Docker-based vLLM due to cleanliness of environments and adherence to SIGTERM signals sent by the server. I've written out both options here.

This should result in a functioning llama-swap instance running at `http://localhost:9292`, which can be confirmed by running `curl http://localhost:9292/health`. It is **highly recommended** that you read the configuration documentation. llama-swap is thoroughly documented and highly configurable - utilizing its capabilities will result in a tailored setup ready to deploy as you need it.

### `systemd` Service

📖 [**llama.cpp Server Documentation**](https://github.com/ggml-org/llama.cpp/tree/master/tools/server)  
📖 [**llama.cpp Preset Documentation**](https://github.com/ggml-org/llama.cpp/blob/master/docs/preset.md)  
📖 [**vLLM Online Serving Documentation**](https://docs.vllm.ai/en/stable/serving/online_serving.html)

The other way to persist a model across system reboots is to start the inference engine in a `.service` file that will run alongside the Linux operating system when booting, ensuring that it is available whenever the server is on.

llama.cpp can now natively allow for model-switching via its `--models-preset` flag. Compared to llama-swap, llama.cpp's in-built switcher naturally has less overhead because it's bundled with the binary but it also doesn't come with a UI where one can switch models, view activity & logs, monitor system performance during inference, etc. llama.cpp and llama-swap are functionally equivalent with respect to model-switching. For vLLM, if you're willing to live with the relative compromise of not being able to swap models/backends and are satisfied with running one model, `systemd` is the lowest overhead solution and works great.

Let's call the service we're about to build `llm-server.service`. We'll assume all models are in the `models` child directory - you can change this as you need to.

Create the `systemd` service file:
```bash
sudo nano /etc/systemd/system/llm-server.service
```

**llama.cpp**

1. Create the `models.ini` file (in your llama.cpp directory):
    ```ini
    version = 1

    [*]
    jinja = true
    no-mmap = true
    no-warmup = true
    flash-attn = true

    [qwen3.6-27b]
    model = models/Qwen3.6-27B/Qwen_Qwen3.6-27B-Q8_0.gguf
    mmproj = models/Qwen3.6-27B/mmproj-BF16.gguf
    ctx-size = 180000
    parallel = 1
    context-shift = true
    fit = off
    tensor-split = 12,11
    spec-type = draft-mtp
    spec-draft-n-max = 2
    temp = 0.6
    min-p = 0.0
    top-p = 0.95
    top-k = 20
    presence-penalty = 0.0
    repeat-penalty = 1.0
    chat-template-kwargs = {"preserve_thinking": true}

    [gemma-4-31b]
    model = models/Gemma4-31B/gemma-4-31B-it-qat-UD-Q4_K_XL.gguf
    mmproj = models/Gemma4-31B/mmproj-BF16.gguf
    model-draft = models/Gemma4-31B/gemma-4-31b-it-qat-q4_0-assistant.gguf
    ctx-size = 130000
    main-gpu = 0
    n-gpu-layers = 99
    parallel = 1
    fit = on
    spec-type = draft-mtp
    spec-draft-n-max = 2
    chat-template-kwargs = {"preserve_thinking": true}
    temp = 1.0
    min-p = 0.0
    top-p = 0.95
    top-k = 64
    cache-ram = 2048
    ctx-checkpoints = 2
    image-min-tokens = 300
    image-max-tokens = 512
    ```

2. Configure the service file:
    ```ini
    [Unit]
    Description=llama.cpp Server
    After=network.target

    [Service]
    Type=simple
    User=<username>
    Group=<username>
    WorkingDirectory=/home/<username>/llama.cpp

    ExecStart=/home/<username>/llama.cpp/build/bin/llama-server \
        --models-preset models.ini \
        --port 9292 \
        --host 0.0.0.0

    # Restart policy
    Restart=on-failure
    RestartSec=5

    [Install]
    WantedBy=multi-user.target
    ```

**vLLM**

1. Configure the service file:
    ```ini
    [Unit]
    Description=LLM Server Service
    After=network.target

    [Service]
    User=<user>
    Group=<user>
    WorkingDirectory=/home/<user>/vllm/
    ExecStart=/bin/bash -c 'source .venv/bin/activate && vllm serve --port <port> --host 0.0.0.0 -m /home/<username>/vllm/models/<model>'
    Restart=always
    RestartSec=10s

    [Install]
    WantedBy=multi-user.target
    ```

> Replace `<username>`, `<port>`, and `<model>` with your Linux username, desired port for serving, and desired model respectively.

Reload the `systemd` daemon:
```bash
sudo systemctl daemon-reload
```
Run the service:

- If `llm-server.service` doesn't exist:

    ```bash
    sudo systemctl enable llm-server.service
    sudo systemctl start llm-server
    ```

- If `llm-server.service` does exist:

    ```bash
    sudo systemctl restart llm-server
    ```

(Optional) Check the service's status:
```bash
sudo systemctl status llm-server
```

### Open WebUI Integration

#### llama-swap

Navigate to `Admin Panel > Settings > Connections` and set the following values:

- Enable OpenAI API
- API Base URL: `http://llama-swap:8080/v1`
- API Key: `anything-you-like`

#### `systemd` Service

Follow the same steps as above.

- Enable OpenAI API
- API Base URL: `http://localhost:<port>/v1`
- API Key: `anything-you-like`

> Replace `<port>` with your desired port.

> [!WARNING]
> If you use the `systemd` service, ensure the port is open to the local network via `ufw`. Refer to [Firewall](#firewall).

## Chat Platform

### Open WebUI

🌟 [**GitHub**](https://github.com/open-webui/open-webui)  
📖 [**Documentation**](https://docs.openwebui.com)

Open WebUI is a web-based interface for managing models and chats, and provides a beautiful, performant UI for communicating with your models. You will want to do this if you want to access your models from a web interface. If you're fine with using the command line or want to consume models through a plugin/extension, you can skip this step.

To install without Nvidia GPU support, run the following command:
```bash
docker run -d -p 3000:8080 --network app-net --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main
```

For Nvidia GPUs, run the following command:
```bash
docker run -d -p 3000:8080 --network app-net --gpus all --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:cuda
```

You can access it by navigating to `http://localhost:3000` in your browser or `http://<server_ip>:3000` from another device on the same network. There's no need to add this to the `init.bash` script as Open WebUI will start automatically at boot via Docker Engine.

Read more about Open WebUI [here](https://github.com/open-webui/open-webui).

> [!NOTE]
> `host.docker.internal` is a magic hostname that resolves to the internal IP address assigned to the host by Docker. This allows containers to communicate with services running on the host, such as databases or web servers, without needing to know the host's IP address. It simplifies communication between containers and host-based services, making it easier to develop and deploy applications.

## MCP Proxy Server

Model Context Protocol (MCP) is a protocol that tools (functions/scripts written in code) to LLMs in a standardized way. Generally, models are being trained more and more with the ability to natively call tools in order to power agentic tasks - think along the lines of having a model use sequential thinking to formulate multiple thoughts, execute multiple targeted web searches, and provide a response leveraging real-time information. MCP also, probably more importantly for most people, enables models to call third-party tools like those for GitHub, Azure, etc. A complete list, curated and maintained by Anthropic, can be found [here](https://github.com/modelcontextprotocol/servers).

Most guides on the Internet concerning MCP will have you spin up an MCP server via a client like VS Code, Cline, etc. since most agentic uses are for coding or Claude Desktop, which is a proprietary app by Anthropic and not at all what we're aiming to achieve in this guide with respect to privacy. There *are* other chat clients that support MCP server management from the UI itself (LobeChat, Cherry Studio, etc.), but we want to be able to manage MCP servers in a central and modular way. That way, (i) they aren't tied to a specific client and are available to every client you may use them with and (ii) if you switch chat platforms in the future, your MCP servers require zero changes because they run as a decoupled service - little bit more maintenance for a lot more flexibility down the line. We can do this by setting up an MCP proxy server.

This proxy server will take the MCP servers running via stdio (standard IO) protocol (that can only be accessed by an application running on that device) and make them compatible with streamable HTTP. Any MCP-enabled client can use streamable HTTP so they'll also be able to use all the servers we install on our physical server. This centralizes the management of your MCP servers: create/edit/delete servers in one place, use any of them from your various clients (Open WebUI, VS Code, etc.).

We'll use the [fetch](https://github.com/zcaceres/fetch-mcp), [sequential-thinking](https://github.com/arben-adm/mcp-sequential-thinking), and [searxng](https://github.com/ihor-sokoliuk/MCP-searxng) MCP servers to get started. The process for adding more servers will be identical.

### mcp-proxy

🌟 [**GitHub**](https://github.com/sparfenyuk/mcp-proxy)  

mcp-proxy is a server proxy that allows switching between transports (stdio to streamable HTTP and vice versa). I'll be using port 3131 to avoid conflicts - feel free to change this as you need to. I will also be extending mcp-proxy to include `uv`: most MCP servers use either `npx` or `uv` and setting mcp-proxy up without `uv` is going to hamper your ability to run the MCP servers you'd like. If you don't require `uv`, (i) don't add the `build` section in the compose file and (ii) skip step 4.

1. Create a compose file
    ```bash
    mkdir mcp-proxy
    cd mcp-proxy
    sudo nano docker-compose.yaml
    ```

2. Enter the following:
    ```yaml
    services:
      mcp-proxy:
        container_name: mcp-proxy
        build:
          context: .
          dockerfile: Dockerfile
        networks:
          - app-net
        volumes:
          - .:/config
          - /:/<server_hostname>:ro
        restart: unless-stopped
        ports:
          - 3131:3131
        command: "--pass-environment --port=3131 --host 0.0.0.0 --transport streamablehttp --named-server-config /config/servers.json"

    networks:
      app-net:
        external: true
    ```

    > Replace `<server_hostname>` with your actual server's hostname (or whatever else). This is primarily useful when adding `filesystem` or similar MCP servers that read from and write files to the file system. Feel free to skip if that isn't your goal.

3. Create a `servers.json` file:
    ```json
    {
        "mcpServers": {
            "fetch": {
                "disabled": false,
                "timeout": 60,
                "command": "uvx",
                "args": [
                    "mcp-server-fetch"
                ],
                "transportType": "stdio"
            },
            "sequential-thinking": {
                "command": "npx",
                "args": [
                    "-y",
                    "@modelcontextprotocol/server-sequential-thinking"
                ]
            },
            "searxng": {
                "command": "npx",
                "args": ["-y", "mcp-searxng"],
                "env": {
                    "SEARXNG_URL": "http://searxng:8080/search?q=<query>"
                }
            }
        }
    }
    ```

4. Create a `Dockerfile`:
    ```bash
    sudo nano Dockerfile
    ```
    Enter the following:
    ```Dockerfile
    FROM ghcr.io/sparfenyuk/mcp-proxy:latest

    # Install dependencies for nvm and Node.js
    RUN apk add --update npm

    # Install the 'uv' package
    RUN python3 -m ensurepip && pip install --no-cache-dir uv

    ENV PATH="/usr/local/bin:/usr/bin:$PATH" \
        UV_PYTHON_PREFERENCE=only-system

    ENTRYPOINT ["catatonit", "--", "mcp-proxy"]
    ```

5. Start the container with `docker compose up -d`

Your mcp-proxy container should be up and running! Adding servers is simple: add the relevant server to `servers.json` (you can use the same configuration that the MCP server's developer provides for VS Code, it's identical) and then restart the container with `docker restart mcp-proxy`.

### MCPJungle

🌟 [**GitHub**](https://github.com/mcpjungle/MCPJungle?tab=readme-ov-file)  

MCPJungle is another MCP proxy server with a different focus. It focuses on providing more of a "production-grade" experience, a lot of which is disabled by default in the development mode of the application. We'll use the standard development version of the container here on port 4141.

1. Create a compose file:
    ```bash
    mkdir mcpjungle
    cd mcpjungle
    sudo nano docker-compose.yaml
    ```

   Enter the following and save:

    ```yaml
    # MCPJungle Docker Compose configuration for individual users.
    # Use this compose file if you want to run MCPJungle locally for your personal MCP management & Gateway.
    # The mcpjungle server runs in development mode.
    services:
      db:
        image: postgres:latest
        container_name: mcpjungle-db
        environment:
          POSTGRES_USER: mcpjungle
          POSTGRES_PASSWORD: mcpjungle
          POSTGRES_DB: mcpjungle
        ports:
          - "5432:5432"
        networks:
          - app-net
        volumes:
          - db_data:/var/lib/postgresql/data
        healthcheck:
          test: ["CMD-SHELL", "PGPASSWORD=mcpjungle pg_isready -U mcpjungle"]
          interval: 10s
          timeout: 5s
          retries: 5
        restart: unless-stopped

      mcpjungle:
        image: mcpjungle/mcpjungle:${MCPJUNGLE_IMAGE_TAG:-latest-stdio}
        container_name: mcpjungle-server
        environment:
          DATABASE_URL: postgres://mcpjungle:mcpjungle@db:5432/mcpjungle
          SERVER_MODE: ${SERVER_MODE:-development}
          OTEL_ENABLED: ${OTEL_ENABLED:-false}
        ports:
          - "4141:8080"
        networks:
          - app-net
        volumes:
          # Mount host filesystem current directory to enable filesystem MCP server access
          - .:/host/project:ro
          - /home/<username>:/host:ro
          # Other options:
          # - ${HOME}:/host/home:ro
          # - /tmp:/host/tmp:rw
        depends_on:
          db:
            condition: service_healthy
        restart: always

    volumes:
      db_data:

    networks:
      app-net:
        external: true
    ```

2. Start the container with `docker compose up -d`

3. Create a tool file:
    ```bash
    sudo nano fetch.json
    ```

    Enter the following and save:
    ```json
    {
        "name": "fetch",
        "transport": "stdio",
        "command": "npx",
        "args": ["mcp-server-fetch"]
    }
    ```

4. Register the tool:
    ```bash
    docker exec -i mcpjungle-server /mcpjungle register -c /host/project/fetch.json
    ```

Repeat steps 3 and 4 for every tool mentioned. Commands for `sequential-thinking` and `searxng` can be found below.

**sequential-thinking**
```json
{
    "name": "sequential-thinking",
    "transport": "stdio",
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-sequential-thinking"]
}
```

**searxng**
```json
{
    "name": "searxng",
    "transport": "stdio",
    "command": "npx",
    "args": ["-y", "mcp-searxng"],
    "env": {
        "SEARXNG_URL": "http://searxng:8080/search?q=<query>"
    }
}
```

### Comparison

The choice between the two services is yours entirely: I use mcp-proxy because I find the workflow slightly less cumbersome than MCPJungle. Here's a comparison with the stengths of each service.

**mcp-proxy > MCPJungle**

- Servers can just be added to `servers.json` and will be registered automatically on container restart - MCPJungle requires manual registration of tools via the CLI
- Uses the standard MCP syntax that most clients accept for configuration
- Lighter footprint in that it doesn't need to spin up a separate database container
- Uses stateful connections - MCPJungle spins up a new connection per tool call, which can lead to some performance overhead

**MCPJungle > mcp-proxy**

- Combines all tools under one endpoint, making it very easy to integrate into a chat frontend
- Capable of creating a very configurable setup with tool groups, access control, selective tool enabling/disabling
- Supports enterprise features like telemetry

### Open WebUI Integration

Open WebUI recently added support for streamable HTTP - where once you may have had to use [mcpo](https://github.com/open-webui/mcpo), Open WebUI's way of automatically generating an OpenAPI-compatible HTTP server, you can use the MCP servers you've set up as-is with no changes.

#### mcp-proxy

Navigate to `Admin Panel > Settings > External Tools`. Click the `+` button to add a new tool and enter the following information:

- URL: `http://mcp-proxy:<port>/servers/<tool_name>/mcp`
- API Key: `anything-you-like`
- ID: `<tool_name>`
- Name: `<tool_name>`

> Replace `<port>` with the port of the MCP service and `<tool_name>` with the specific tool you're adding.

#### MCPJungle

Follow the same steps as above. By design, MCPJungle exposes all tools via one endpoint, so you should only have to add once:

- URL: `http://mcpjungle-server:8080/mcp`
- API Key: `anything-you-like`
- ID: `<tool_name>`
- Name: `<tool_name>`

> [!IMPORTANT]
> When configuring models in Open WebUI (via `Admin Panel > Settings > Models > my-cool-model > Advanced Params`), change the `Function Calling` parameter from `Default` to `Native`. This step will allow the model to use multiple tool calls to formulate a single response instead of just one.

### VS Code/Claude Desktop Integration

The steps for integrating your MCP proxy server in another client such as VS Code (Claude Desktop, Zed, etc.) will be similar, if not exactly the same.

Add the following key and value to your `mcp.json` file:

```json
"your-mcp-proxy-name": {
    "timeout": 60,
    "type": "stdio",
    "command": "npx",
    "args": [
    "mcp-remote",
    "http://<your-server-url>/mcp",
    "--allow-http"
    ]
}
```

## Text-to-Speech Server

### Kokoro FastAPI

🌟 [**GitHub**](https://github.com/remsky/Kokoro-FastAPI)

Kokoro FastAPI is a text-to-speech server that wraps around and provides OpenAI-compatible API inference for [Kokoro-82M](https://huggingface.co/hexgrad/Kokoro-82M), a state-of-the-art TTS model. The documentation for this project is fantastic and covers most, if not all, of the use cases for the project itself.

To install Kokoro-FastAPI, run
```bash
git clone https://github.com/remsky/Kokoro-FastAPI.git
cd Kokoro-FastAPI
docker compose up --build
```

The server can be used in two ways: an API and a UI. By default, the API is served on port 8880 and the UI is served on port 7860.

### Open WebUI Integration

Navigate to `Admin Panel > Settings > Audio` and set the following values:

- Text-to-Speech Engine: `OpenAI`
- API Base URL: `http://host.docker.internal:8880/v1`
- API Key: `anything-you-like`
- Set Model: `kokoro`
- Response Splitting: None (this is crucial - Kokoro uses a novel audio splitting system)

The server can be used in two ways: an API and a UI. By default, the API is served on port 8880 and the UI is served on port 7860.

## Image Generation Server

### ComfyUI

🌟 [**GitHub**](https://github.com/comfyanonymous/ComfyUI)  
📖 [**Documentation**](https://docs.comfy.org)

ComfyUI is a popular open-source graph-based tool for generating images using image generation models such as Stable Diffusion XL, Stable Diffusion 3, and the Flux family of models.

- Clone and navigate to the repository:
    ```
    git clone https://github.com/comfyanonymous/ComfyUI
    cd ComfyUI
    ```
- Set up a new virtual environment:
    ```
    python3 -m venv comfyui-env
    source comfyui-env/bin/activate
    ```
- Download the platform-specific dependencies:
  - Nvidia GPUs
    ```
    pip install torch torchvision torchaudio --extra-index-url https://download.pytorch.org/whl/cu121
    ```
  - AMD GPUs
    ```
    pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/rocm6.0
    ```
  - Intel GPUs
  
    Read the installation instructions from [ComfyUI's GitHub](https://github.com/comfyanonymous/ComfyUI?tab=readme-ov-file#intel-gpus).
    
- Download the general dependencies:
    ```
    pip install -r requirements.txt
    ```

Now, we have to download and load a model. Here, we'll use FLUX.1 [dev], a new, state-of-the-art medium-tier model by Black Forest Labs that fits well on an RTX 3090 24GB. Since we want this to be set up as easily as possible, we'll use a complete checkpoint that can be loaded directly into ComfyUI. For a completely customized workflow, CLIPs, VAEs, and models can be downloaded separately. Follow [this guide](https://comfyanonymous.github.io/ComfyUI_examples/flux/#simple-to-use-fp8-checkpoint-version) by ComfyUI's creator to install the FLUX.1 models in a fully customizable way.

> [!NOTE]
> [FLUX.1 [schnell] HuggingFace](https://huggingface.co/Comfy-Org/flux1-schnell/blob/main/flux1-schnell-fp8.safetensors) (smaller, ideal for <24GB VRAM)
> 
> [FLUX.1 [dev] HuggingFace](https://huggingface.co/Comfy-Org/flux1-dev/blob/main/flux1-dev-fp8.safetensors) (larger, ideal for 24GB VRAM)

- Download your desired model into `/models/checkpoints`.

- If you want ComfyUI to be served at boot and effectively run as a service, add the following lines to `init.bash`:
    ```
    cd /path/to/comfyui
    source comfyui/bin/activate
    python main.py --listen
    ```
    > Replace `/path/to/comfyui` with the correct relative path to `init.bash`.

    Otherwise, to run it just once, simply execute the above lines in a terminal window.

### Open WebUI Integration

Navigate to `Admin Panel > Settings > Images` and set the following values:

- Image Generation Engine: `ComfyUI`
- API Base URL: `http://localhost:8188`

> [!TIP]
> You'll either need more than 24GB of VRAM or to use a small language model mostly on CPU to use Open WebUI with FLUX.1 [dev]. FLUX.1 [schnell] and a small language model, however, should fit cleanly in 24GB of VRAM, making for a faster experience if you intend to regularly use both text and image generation together.

## SSH

Enabling SSH allows you to connect to the server remotely. After configuring SSH, you can connect to the server from another device on the same network using an SSH client like PuTTY or the terminal. This lets you run your server headlessly without needing a monitor, keyboard, or mouse after the initial setup.

On the server:
- Run the following command:
    ```
    sudo apt install openssh-server
    ```
- Start the SSH service:
    ```
    sudo systemctl start ssh
    ```
- Enable the SSH service to start at boot:
    ```
    sudo systemctl enable ssh
    ```
- Find the server's IP address:
    ```
    ip a
    ```

On the client:
- Connect to the server using SSH:
    ```
    ssh <username>@<ip_address>
    ```
    > Replace `<username>` with your username and `<ip_address>` with the server's IP address.

> [!NOTE]
> If you expect to tunnel into your server often, I highly recommend following [this guide](https://www.raspberrypi.com/documentation/computers/remote-access.html#configure-ssh-without-a-password) to enable passwordless SSH using `ssh-keygen` and `ssh-copy-id`. It worked perfectly on my Debian system despite having been written for Raspberry Pi OS.

## Firewall

Setting up a firewall is essential for securing your server. The Uncomplicated Firewall (UFW) is a simple and easy-to-use firewall for Linux. You can use UFW to allow or deny incoming and outgoing traffic to and from your server.

- Install UFW:
    ```bash
    sudo apt install ufw
    ```

- Allow SSH, HTTPS, and HTTP to your local network:
    ```bash
    # Allow all <ip_range> hosts access to port <port>
    sudo ufw allow from <ip_range> to any port <port> proto tcp
    ```
    
    Start with running the above command to open ports 22 (SSH), 80 (HTTP), and 443 (HTTPS) to our local network. Since we use the `app-net` Docker network for our containers, there's no need to open anything else up. Open ports up carefully, ideally only to specific IPs or to your local network. To allow a port for a specific IP, you can replace the IP range with a single IP and it'll work exactly the same way.

> [!TIP]
> You can find your local network's IP address range by running `ip route show`. The result will be something like this:
> ```
> me@my-cool-server:~$ ip route show
> default via <router_ip> dev enp3s0 proto dhcp src <server_ip> metric 100
> <network_ip_range> dev enp3s0 proto kernel scope link src <server_ip> metric 100
> # more routes
> ```

- Enable UFW:
    ```bash
    sudo ufw enable
    ```

- Check the status of UFW:
    ```bash
    sudo ufw status verbose
    ```

> [!WARNING]
> Enabling UFW without allowing access to port 22 will disrupt your existing SSH connections. If you run a headless setup, this means connecting a monitor to your server and then allowing SSH access through UFW. Be careful to ensure that this port is allowed when making changes to UFW's configuration.

Refer to [this guide](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-debian-10) for more information on setting up UFW.

## Remote Access

This section covers the software required to allow secure and efficient access to the homelab environment. It involves leveraging Caddy as a reverse proxy engine, Cloudflare as a DNS provider, and Tailscale for the mesh VPN that provides built-in security due to firewall hole punching. It is highly recommended if you're looking at building a system that will be used professionally or even by family and friends.

📖 See the full guide at [Remote Access](docs/remote-access.md).

## Updating

Updating your system is a good idea to keep software running optimally and with the latest security patches. Updates to Ollama allow for inference from new model architectures and updates to Open WebUI enable new features like voice calling, function calling, pipelines, and more.

I've compiled steps to update these "primary function" installations in a standalone section because I think it'd be easier to come back to one section instead of hunting for update instructions in multiple subsections.

### General

Upgrade Debian packages by running the following commands:
```
sudo apt update
sudo apt upgrade
```

### Nvidia Drivers & CUDA

Follow Nvidia's guide [here](https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&Distribution=Debian) to install the latest CUDA drivers.

> [!WARNING]
> Don't skip this step. Not installing the latest drivers after upgrading Debian packages will throw your installations out of sync, leading to broken functionality. When updating, target everything important at once. Also, rebooting after this step is a good idea to ensure that your system is operating as expected after upgrading these crucial drivers.

### Docker Services

Most services in this guide run via Docker and can be updated using the same general pattern. Navigate to the directory containing the service's compose file or container and use the appropriate commands below.

**Docker Compose services**:

```bash
cd <service-directory>
docker compose down
docker compose pull
docker compose up -d
```

> Replace `<service-directory>` with the directory containing the `docker-compose.yaml` file (e.g., `mcp-proxy`, `mcpjungle`, `Kokoro-FastAPI`, `searxng`).

**Standalone containers** (llama-swap, Open WebUI):

For llama-swap, stop and remove the existing container, then re-run the `docker run` command from the [llama-swap section](#llama-swap):
```bash
docker stop llama-swap
docker rm llama-swap
# Re-run the docker run command from the installation section
```

For Open WebUI, use Watchtower to update:
```bash
# One-time update
docker run --rm --volume /var/run/docker.sock:/var/run/docker.sock containrrr/watchtower --run-once open-webui

# Or set up automatic updates
docker run -d --name watchtower --volume /var/run/docker.sock:/var/run/docker.sock containrrr/watchtower open-webui
```

**vLLM (Docker installation)**:

Simply re-run your `docker run` command and the latest Docker image will be pulled automatically.

### Ollama

Rerun the command that installs Ollama - it acts as an updater too:
```
curl -fsSL https://ollama.com/install.sh | sh
```

### llama.cpp

Enter your llama.cpp folder and run the following commands:
```
cd llama.cpp
git pull
# Rebuild according to your setup - uncomment `-DGGML_CUDA=ON` for CUDA support
cmake -B build # -DGGML_CUDA=ON
cmake --build build --config Release
```

### vLLM

For a manual installation, enter your virtual environment and update via `pip`:
```
source vllm/.venv/bin/activate
pip install vllm --upgrade
```

### ComfyUI

Navigate to the directory, pull the latest changes, and update dependencies:
```
cd ComfyUI
git pull
source comfyui-env/bin/activate
pip install -r requirements.txt
```

## Troubleshooting

### Docker

- Since we aim to use our user instead of root to run services, you may encounter permissions issues when trying to mount a volume that is not technically owned by our user. To fix this, you can do one of two things:
  - Change the ownership of the directory:
    ```bash
    sudo chown -R $(id -u):$(id -g) /path/to/volume
    ```

  - Grant access to our user with access control lists (ACLs, in short):
    ```bash
    # Grant read/write to specific user
    sudo setfacl -R -m u:$(id -u):rw /path/to/volume

    # Grant to group
    sudo setfacl -R -m g:$(id -g):rw /path/to/volume
    ```

    In case you don't have the `acl` package, install with `sudo apt update && sudo apt install acl`. The official package page can be found [here](https://packages.debian.org/bookworm/acl).

    > Replace `/path/to/volume` with the desired path.

    I like ACLs better - it solves the access issue cleanly without changing ownership unnecessarily and, in my experience, tends to break things less. However, if the resource is clearly meant to be owned by a specific user, changing ownership is preferred purely for separation of concerns. Gauge for yourself, both are completely acceptable options. Avoid changing permissions if your container is already up and running, though.

### `ssh`
- If you encounter an issue using `ssh-copy-id` to set up passwordless SSH, try running `ssh-keygen -t rsa` on the client before running `ssh-copy-id`. This generates the RSA key pair that `ssh-copy-id` needs to copy to the server.

### Nvidia Drivers
- Disable Secure Boot in the BIOS if you're having trouble with the Nvidia drivers not working. For me, all packages were at the latest versions and `nvidia-detect` was able to find my GPU correctly, but `nvidia-smi` kept returning the `NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver` error. [Disabling Secure Boot](https://askubuntu.com/a/927470) fixed this for me. Better practice than disabling Secure Boot is to sign the Nvidia drivers yourself but I didn't want to go through that process for a non-critical server that can afford to have Secure Boot disabled.
- If you run into `docker: Error response from daemon: unknown or invalid runtime name: nvidia.`, you probably have `--runtime nvidia` in your Docker statement. This is meant for `nvidia-docker`, [which is deprecated now](https://stackoverflow.com/questions/52865988/nvidia-docker-unknown-runtime-specified-nvidia). Removing this flag from your command should get rid of this error.

### Ollama
- If you receive the `could not connect to ollama app, is it running?` error, your Ollama instance wasn't served properly. This could be because of a manual installation or the desire to use it at-will and not as a service. To run the Ollama server once, run:
    ```
    ollama serve
    ```
    Then, **in a new terminal**, you should be able to access your models regularly by running:
    ```
    ollama run <model>
    ```
    For detailed instructions on _manually_ configuring Ollama to run as a service (to run automatically at boot), read the official documentation [here](https://github.com/ollama/ollama/blob/main/docs/linux.md). You shouldn't need to do this unless your system faces restrictions using Ollama's automated installer.
    
- If you receive the `Failed to open "/etc/systemd/system/ollama.service.d/.#override.confb927ee3c846beff8": Permission denied` error from Ollama after running `systemctl edit ollama.service`, simply creating the file works to eliminate it. Use the following steps to edit the file. 
  - Run:
    ```
    sudo mkdir -p /etc/systemd/system/ollama.service.d
    sudo nano /etc/systemd/system/ollama.service.d/override.conf
    ```
  - Retry the remaining steps.
- If you still can't connect to your API endpoint, check your firewall settings. [This guide to UFW (Uncomplicated Firewall) on Debian](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-debian-10) is a good resource.

### vLLM
- If you encounter ```RuntimeError: An error occurred while downloading using `hf_transfer`. Consider disabling HF_HUB_ENABLE_HF_TRANSFER for better error handling.```, add `HF_HUB_ENABLE_HF_TRANSFER=0` to the `--env` flag after your HuggingFace Hub token. If this still doesn't fix the issue -
  - Ensure your user has all the requisite permissions for HuggingFace to be able to write to the cache. To give read+write access over the HF cache to your user (and, thus, `huggingface-cli`), run:
    ```
    sudo chmod 777 ~/.cache/huggingface
    sudo chmod 777 ~/.cache/huggingface/hub
    ```
  - Manually download a model via the HuggingFace CLI and specify `--download-dir=~/.cache/huggingface/hub` in the engine arguments. If your `.cache/huggingface` directory is being troublesome, specify another directory to the `--download-dir` in the engine arguments and remember to do the same with the `--local-dir` flag in any `huggingface-cli` commands.

### Open WebUI
- If you encounter `Ollama: llama runner process has terminated: signal: killed`, check your `Advanced Parameters`, under `Settings > General > Advanced Parameters`. For me, bumping the context length past what certain models could handle was breaking the Ollama server. Leave it to the default (or higher, but make sure it's still under the limit for the model you're using) to fix this issue.

## Monitoring

To monitor GPU usage, power draw, and temperature, you can use the `nvidia-smi` command. To monitor GPU usage, run:
```
watch -n 1 nvidia-smi
```
This will update the GPU usage every second without cluttering the terminal environment. Press `Ctrl+C` to exit.

## Notes

This is my first foray into setting up a server and ever working with Linux so there may be better ways to do some of the steps. I will update this repository as I learn more.

### Software

- I chose Debian because it is, apparently, one of the most stable Linux distros. I also went with an XFCE desktop environment because it is lightweight and I wasn't yet comfortable going full command line.
- Use a user for auto-login, don't log in as root unless for a specific reason.
- To switch to root in the command line without switching users, run `sudo -i`.
- If something using a Docker container doesn't work, try running `docker ps -a` to see if the container is running. If it isn't, try running `docker compose up -d` again. If it is and isn't working, try running `docker restart <container_id>` to restart the container.
- If something isn't working no matter what you do, try rebooting the server. It's a common solution to many problems. Try this before spending hours troubleshooting. Sigh.
- While it takes some time to get comfortable with, using an inference engine like llama.cpp and vLLM (as compared to Ollama) is really the way to go to squeeze the maximum performance out of your hardware. If you're reading this guide in the first place and haven't already thrown up your hands and used a cloud provider, it's a safe assumption that you care about the ethos of hosting all this stuff locally. Thus, get your experience as close to a cloud provider as it can be by optimizing your server.

### Hardware

- The power draw of my EVGA FTW3 Ultra RTX 3090 was 350W at stock settings. I set the power limit to 250W and the performance decrease was negligible for my use case, which is primarily code completion in VS Code and the Q&A via chat. 
- Using a power monitor, I measured the power draw of my server for multiple days - the running average is ~60W. The power can spike to 350W during prompt processing and token generation, but this only lasts for a few seconds. For the remainder of the generation time, it tended to stay at the 250W power limit and dropped back to the average power draw after the model wasn't in use for about 20 seconds. 
- Ensure your power supply has enough headroom for transient spikes (particularly in multi GPU setups) or you may face random shutdowns. Your GPU can blow past its rated power draw and also any software limit you set for it based on the chip's actual draw. I usually aim for +50% of my setup's estimated total power draw.

## References

Adding user to `sudo` group:
- https://askubuntu.com/questions/168280/how-do-i-grant-sudo-privileges-to-an-existing-user

Downloading Nvidia drivers:
- https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&Distribution=Debian
- https://wiki.debian.org/NvidiaGraphicsDrivers

Downloading AMD drivers:
- https://wiki.debian.org/AtiHowTo

Secure Boot:
- https://askubuntu.com/a/927470

Monitoring GPU usage, power draw: 
- https://unix.stackexchange.com/questions/38560/gpu-usage-monitoring-cuda/78203#78203

Passwordless `sudo`:
- https://stackoverflow.com/questions/25215604/use-sudo-without-password-inside-a-script
- https://www.reddit.com/r/Fedora/comments/11lh9nn/set_nvidia_gpu_power_and_temp_limit_on_boot/
- https://askubuntu.com/questions/100051/why-is-sudoers-nopasswd-option-not-working

Auto-login:
- https://forums.debian.net/viewtopic.php?t=149849
- https://wiki.archlinux.org/title/LightDM#Enabling_autologin

Expose Ollama to LAN:
- https://github.com/ollama/ollama/blob/main/docs/faq.md#setting-environment-variables-on-linux
- https://github.com/ollama/ollama/issues/703

Firewall:
- https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-debian-10

Passwordless `ssh`:
- https://www.raspberrypi.com/documentation/computers/remote-access.html#configure-ssh-without-a-password

Adding CUDA to PATH:
- https://askubuntu.com/questions/885610/nvcc-version-command-says-nvcc-is-not-installed

Docs:

- [Debian](https://www.debian.org/releases/buster/amd64/)
- [Docker](https://docs.docker.com/engine/install/debian/)
- [Ollama](https://github.com/ollama/ollama/blob/main/docs/api.md)
- [vLLM](https://docs.vllm.ai/en/stable/index.html)
- [Open WebUI](https://github.com/open-webui/open-webui)
- [ComfyUI](https://github.com/comfyanonymous/ComfyUI)

## Acknowledgements

Cheers to all the fantastic work done by the open-source community. This guide wouldn't exist without the effort of the many contributors to the projects and guides referenced here. To stay up-to-date on the latest developments in the field of machine learning, LLMs, and other vision/speech models, check out [r/LocalLLaMA](https://www.reddit.com/r/LocalLLaMA/). To stay in touch with (or fall down the rabbit hole of) the world of self-hosted apps out there for your new server, check out [r/selfhosted](https://www.reddit.com/r/selfhosted).

> [!NOTE]
> Please star any projects you find useful and consider contributing to them if you can. Stars on this guide would also be appreciated if you found it helpful, as it helps others find it too. 
