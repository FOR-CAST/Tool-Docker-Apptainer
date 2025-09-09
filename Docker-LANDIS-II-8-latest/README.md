# LANDIS-II v8 Docker Image (Linux Build)

This repository contains a Dockerfile and supporting scripts to build a fully functioning **LANDIS-II v8** environment with selected extensions compiled for **Linux**.
This setup enables reproducible and portable forest landscape modeling, ideal for high-performance or cloud-based simulations.
This build uses the latest commits of each extension.

---
add new image 'Docker-LANDIS-II-v8-latest'

This image closely follows the original `Clean_Docker_LANDIS-II_8_Latest_Commits` image with the following enhancements:

- uses Ubuntu 24.04 (noble) as the base image instead of 22.04 (jammy);
- install `dotnet` from the `apt` repositories rather than installing and configuring manually;
- uses more recent extension commits for several other extensions;
- places the LANDIS-II code at `/opt/landis-ii` instead of `/opt/Core-Model-v8-LINUX`,
  `/opt` is the standard location for software not provided via the OS package manager;
- extension repos and commit SHAs are recorded in a shared `extensions-v8-latest.yaml` file,
  rather than in the Dockerfile, which should be easier to update and maintain going forward;
- likewise, library repos and commit SHAs are recorded in a shared `libraries-v8-latest.yaml` file,
  rather than in the Dockerfile, which should be easier to update and maintain going forward;
- uses shared versions of the Forest Roads and Magic Harvest extensions' `.csproj` files,
  located in the `extension_files/` directory;
- rewrote and simplified various installation scripts to use `bash`;
  these are shared in the `scripts/` directory and can be used directly with other image 'flavours';
  - uses `yq` to parse yaml files in bash;
  - uses `xmlstarlet` to parse and edit XML in the `.csproj` files;
- shared tests are copied from the `tests/` directory and run as part of the build process;
---

## Included Extensions

- Core Model v8
- Biomass Succession
- NECN Succession
- DGS Succession (GIPL + SHAW Libraries)
- Social-Climate-Fire
- Output Biomass
- Output Biomass Community
- Output Biomass Reclass
- Output Max Spp Age


## Build Instructions

To build the Docker image locally:

```bash
git clone https://github.com/LANDIS-II-Foundation/Tool-Docker-Apptainer.git
cd Clean_Docker_LANDIS-II_8_Latest_Commits
docker build -t landisii-v8 .
```

## Starting the container in detached mode and mount your input files
```bash
docker run -d --platform linux/amd64 -v path/to/inputs:/home/user/inputs landisii-v8

#list container names
docker container ls

#enter the container
docker exec -it container_name /bin/bash
```

## Running landis (assuming your scenario file is named "scenario.txt"
```bash
dotnet $console scenario.txt
```
