# Hardening Docker Containers

To "harden" something in software terms is to make it more secure and more resilient to cyberattacks. Usually, this involves reducing the surface area by which an attacker can gain access to the system but it also includes mitigating what an attacker can do assuming they have successfully gotten access. Essentially, we will set up both prevention and cure - although, as they say, an ounce of prevention is worth a pound of cure. This sub-section leverages knowledge from [this Reddit comment](https://www.reddit.com/r/selfhosted/comments/1pr74r4/comment/nv07sp4/), courtesy of [u/arnedam](https://www.reddit.com/user/arnedam/). Another good reference is the [OWASP Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html).

To a large extent, the security objectives we're aiming towards can be achieved by applying the [Principle of Least Privilege (PoLP)](https://en.wikipedia.org/wiki/Principle_of_least_privilege) - in other words, giving our containers exactly and **only** what they need to perform their function. For each guideline, I've provided an existing risk and a mitigation strategy (along with how it fixes the problem); while not strictly necessary, learning *how* things are at risk is beneficial to start thinking securely about the services you host.

> [!NOTE]
> This sub-section is **critical** for public-facing services. However, even if you only intend to access your services via a private network (**preferred if possible**), doing this is quite literally never a bad idea because it limits how much damage can be done to your server and its services if it ever was to become compromised somehow.

## Guidelines

### 1. Run the service under your user, not root

```yaml
user: "<your_user_id>:<your_group_id>"
```

To find `<your_user_id>`, run `id -u`. To find `<your_group_id>`, run `id -g`.

**RISK**: Unless specified otherwise, containers run under the root user. Compromised containers running under root can do all sorts of damage, ranging anywhere from executing malicious code to prying on/deleting system files.

**MITIGATION**: By specifying a user that the container runs under, the container's privilege is limited to the user's privilege, which is almost definitively lower than the root user's privilege to modify the system.

### 2. Make the file system read-only

```yaml
read_only: true
```

**RISK**: By default, containers are allowed to read and write the files they have access to. This means an attacker who has successfully compromised your container could write bogus files/malicious code and spy on/delete critical information.

**MITIGATION**: Adding this will prevent attackers from being able to write malicious code files or delete important data from your system that are accessible to the container.

> [!WARNING]
> Apply the `read_only: true` directive with care: if your service leverages file writes, like the many services that write to `/tmp`, those subprocesses will fail. If those subprocesses are critical, you may even see the container fail to start. Expect this **not** to work in an all-encompassing way for all containers - you will most likely need to surgically apply the `:ro` directive to volume mounts in most places.

### 3. Limit the physical resources your container can access

```yaml
# maximum number of processes the container can execute
pids_limit: 512
# maximium memory for the container
mem_limit: 3g
# maximum number of CPU cores the container can utilize - can be fractional
cpus: 3
```

**RISK**: Containers are, by default, given unrestricted physical resource allocation capabilities. This means that if a container was to be compromised, it would be able to allocate as much of your CPU clock cycles, RAM capacity, and VRAM capacity to executing malicious code.

**MITIGATION**: Allocating fixed limits to resources will prevent attackers from using the entirety of your server's physical resources to do damage if your container was to be compromised, e.g. running a botnet via your machine.

**Docker Swarm**

If you're using Docker Swarm (not used in this guide), you need to format it slightly differently:

```yaml
deploy:
  resources:
    limits:
      pids: 512
      memory: 3g
      cpus: 3
```

### 4. Disable `tty` and `stdin` in the container

```yaml
tty: false
stdin_open: false
```

**RISK**: `stdin` gives the container the ability for commands to be written to it (input injection) and `tty` gives the container an active shell environment - together, they grant the container complete interactive shell capability such that potentially malicious commands can be run from it.

**MITIGATION**: Disabling these will prevent attackers from being able to execute code via the containers, severely minimizing [arbitrary code execution](https://en.wikipedia.org/wiki/Arbitrary_code_execution) (ACE) vulnerabilities.

### 5. Disallow the container from elevating its own privileges

```yaml
security_opt:
  - "no-new-privileges=true"
```

**RISK**: Containers are able to elevate their own access given an interactive shell environment. This could even override the `user` directive provided earlier, giving root-level access to the system via the container.

**MITIGATION**: Adding this line will prevent attackers from being able to override the `user` directive we provided earlier and stop them from being able to grant the now-compromised container root-level permissions.

### 6. Drop default container capabilities

```yaml
cap_drop:
  - ALL
```

**RISK**: Containers are given a lot of permissions by default. Most of these permissions are not required by most containers. Giving extra access increases the attack vector surface area in case a container was to become compromised - especially since these capabilities can impact the host kernel.

**MITIGATION**: Adding this will drop the widely-permissive default permissions granted to containers. This ensures that we only give containers the system capabilities to do the things they are required to do.

`cap_drop` with a value of `ALL` can be too aggressive for containers that require multiple capabilities. In this case, right after the above block, consider reviewing the following commonly-used capabilities. They are **not all required and should not all be added** - they are just a starting point for some capabilities sometimes required by containers.

```yaml
cap_add:
  # LOWER RISK
  # Make arbitrary changes to file UIDs and GIDs (see chown(2))
  - CHOWN
  # Make arbitrary manipulations of process GIDs and supplementary GID list
  - SETGID
  # Make arbitrary manipulations of process UIDs
  - SETUID
  # Bind a socket to internet domain privileged ports (port numbers less than 1024)
  - NET_BIND_SERVICE

  # HIGHER RISK - CAN AFFECT IMPORTANT SYSTEM-LEVEL CONFIGURATIONS
  # Perform various network-related operations
  - NET_ADMIN
  # Use RAW and PACKET sockets
  - NET_RAW
  # Perform a range of system administration operations
  - SYS_ADMIN
```

This will, in order, drop all capabilities and then surgically re-add the ones that are required for the functionality the container achieves. A complete list of capabilities can be found [here](https://docs.docker.com/engine/containers/run/#runtime-privilege-and-linux-capabilities).

Containers with inadequate capabilities will fail to run. If you drop one that is required, you could:
1. Ideally, take a look at the list of capabilities referenced above, and perform some trial-and-error addition of capabilities. LLMs can help greatly with this endeavor, especially when equipped with tools that can make them capable of reading documentation or the source code of the service you're looking to run. You can skip this directive until you have everything configured and come back to it later.
2. Less ideally, give up and remove the `cap_drop` directive entirely. I wouldn't recommend it but, with private services, this is hardly the most insecure setup in the world if you do everything else.

### 7. Prevent excessive logging

```yaml
logging:
  driver: "json-file"
  options:
    max-file: "10"
    max-size: "20m"
```

**RISK**: Much like a [zip bomb](https://en.wikipedia.org/wiki/Zip_bomb), a logging bomb can recursively clutter and eventually render the system unusable by overwhelming the system with an ever-increasing number of large log files.

**MITIGATION**: By capping the maximum number of files (10) and sizes of the individual files (20MB), we can limit the effectiveness of an attack like this.

Feel free to change the limits (reasonably) as you wish if you run into issues.

### 8. Limit the temporary file directory's privileges

```yaml
tmpfs:
  - /tmp:rw,noexec,nosuid,nodev,size=512m
```

**RISK**: If allowed to download files, compromised containers could both install malicious files to the filesystem and execute them.

**MITIGATION**: Setting up a temporary file area with very limited permissions stops this potential attack vector. Containers are limited to downloading files here, in a little sandbox with a fixed small size and without script execution privileges.

### 9. Mount read-only volumes

```yaml
volumes:
  - /path/to/mount1:/mount1:ro
  - /path/to/mount2:/mount2:ro
```

**RISK**: To access the file system of the host, Docker containers need directories "mounted" as volumes. This lets them read and write from the host as if the container was the host itself. Allowing write access to every mount is usually not necessary and can allow a compromised container to delete or overwrite important information in a cyber-attack.

**MITIGATION**: Mounting volumes as read-only, wherever possible, eliminates the container's ability to destroy the data in those volumes via writes. This wouldn't prevent spyware, but it would limit data from being lost in an attack.

> Replace `/path/to/mount1` and `/path/to/mount2` with actual directory paths.

> [!IMPORTANT]
> For containers where free allocation of CPU cores and memory is crucial - llama-swap, for example - you will not want to limit the maximum resources the container can access as shown in step 3.

## Combined Example

Here's a combined chunk to copy + paste into your existing services' Compose files:

```yaml
services:
  <service_name>:
    # 1
    user: "<your_user_id>:<your_group_id>"
    # 2 - Uncomment only if the container does not write
    # read_only: true
    # 3
    pids_limit: 512
    mem_limit: 3g
    cpus: 3
    # 4
    tty: false
    stdin_open: false
    # 5
    security_opt:
      - "no-new-privileges=true"
    # 6
    cap_drop:
      - ALL
    # Add cap_add section if required
    # 7
    logging:
      driver: "json-file"
      options:
        max-file: "10"
        max-size: "20m"
    # 8
    tmpfs:
      - /tmp:rw,noexec,nosuid,nodev,size=512m
    # 9
    volumes:
      - /path/to/mount1:/mount1:ro
      - /path/to/mount2:/mount2:ro
```
