FROM ubuntu:24.04

LABEL org.opencontainers.image.authors="wesr@uoregon.edu; clem.hardy@pm.me; achubaty@for-cast.ca;"

################################################################
## PREREQUISITES
################################################################

ENV C_INCLUDE_PATH=/usr/include/gdal
ENV CPLUS_INCLUDE_PATH=/usr/include/gdal
ENV DEBIAN_FRONTEND=noninteractive
ENV LC_ALL=C.UTF-8

USER root

## Install additional dependencies
## - sudo, ca-certificates, and gnupg for key management;
## - add 'universe' repository;
## - `mono-utils` provides `monodis` used to debug ext/lib builds;
## - python needed for Magic Harvest;
## - `xmlstarlet` used to edit xml in .csproj files;
RUN apt-get update && apt-get upgrade -y \
  && apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    dotnet-sdk-8.0 \
    gdal-bin \
    git \
    gnupg \
    libgdal-dev \
    libjpeg62 \
    libpng16-16 \
    python3 \
    python3-pip \
    software-properties-common \
    sudo \
    unzip \ 
    wget \
    xmlstarlet \
  && add-apt-repository universe \
  && apt-get clean && rm -rf /var/lib/apt/lists/*

## install yq (used to parse yaml files) from source
RUN wget https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 -O /usr/local/bin/yq \
  && chmod +x /usr/local/bin/yq

## Add GitHub CLI GPG key and repository
RUN curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | dd of=/etc/apt/trusted.gpg.d/githubcli-archive-keyring.gpg && \
  echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/trusted.gpg.d/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" > /etc/apt/sources.list.d/github-cli.list && \
  apt-get update && \
  apt-get install -y gh && \
  apt-get clean && \
  rm -rf /var/lib/apt/lists/*

## configure git (be sure to unset anything changed here at the end):
## - `advice.detachedHead false` suppresses 'detached HEAD' messages during checkout;
## - `http.*` settings help with slow/high-latency network connections;
RUN  git config --global advice.detachedHead false \
  && git config --global http.lowSpeedLimit 0 \
  && git config --global http.lowSpeedTime 999999 \
  && git config --global http.postBuffer 524288000 \
  && git config --global http.version HTTP/1.1

## set variables to aid in readability and reuse:
## - ARG variable are set during build only;
## - ENV variables are set in the image and can be used e.g., to run LANDIS;
##
ARG LANDIS_GITHUB="https://github.com/LANDIS-II-Foundation"

ARG LANDIS_DIR="/opt/landis-ii" ## use /opt for non-OS-bundled software
ARG LANDIS_CORE_DIR="$LANDIS_DIR/Core-Model-v8-LINUX"
ARG LANDIS_EXT_DIR="$LANDIS_CORE_DIR/build/extensions"
ARG LANDIS_REL_DIR="$LANDIS_CORE_DIR/build/Release"

ARG EXT_LOG_FILE="$LANDIS_DIR/build_exts.log"
ARG LIB_LOG_FILE="$LANDIS_DIR/build_libs.log"
ARG TESTS_DIR="$LANDIS_DIR/tests"

ENV LANDIS_CONSOLE="$LANDIS_REL_DIR/Landis.Console.dll"
ENV LANDIS_EXT_TOOL="$LANDIS_REL_DIR/Landis.Extensions.dll"

COPY scripts $LANDIS_DIR/scripts
RUN chmod a+x $LANDIS_DIR/scripts/*.sh

################################################################
## COMPILING LANDIS-II AND EXTENSIONS
################################################################

WORKDIR $LANDIS_DIR

## get and build the core model
RUN git clone $LANDIS_GITHUB/Core-Model-v8-LINUX.git \
  && dotnet build $LANDIS_CORE_DIR/Tool-Console/src -c Release

RUN git clone $LANDIS_GITHUB/Support-Library-Dlls-v8.git \
  && mv $LANDIS_DIR/Support-Library-Dlls-v8/* $LANDIS_EXT_DIR/ \
  && rm -r $LANDIS_DIR/Support-Library-Dlls-v8

## recompile some support libraries to resolve library errors
##
##  - necessary files are downloaded and built using `scripts/install_libraries_v8.sh`;
##  - the script parses `libraries-v8-latest.yaml` to checkout specific commits;
##  - it cleans up afterwards, removing the library source files to save space in the image;

COPY libraries-v8-latest.yaml $LANDIS_DIR/libraries-v8-latest.yaml

RUN scripts/install_libraries_v8.sh libraries-v8-latest.yaml $LANDIS_DIR

## DEBUG: library install and dependencies
# RUN echo "$(cat $LIB_LOG_FILE)" 1>&2 && exit 1

## get, build, and register all available extensions
##
##  - necessary files are downloaded and built using `scripts/install_extensions_v8.sh`;
##  - the script parses `extensions-v8-latest.yaml` to checkout specific commits;
##  - it uses sparse checkout of specific directories to avoid downloading the entire repo;
##  - it cleans up afterwards, removing the extension source files to save space in the image;

COPY extension_files $LANDIS_DIR/extension_files
COPY extensions-v8-latest.yaml $LANDIS_DIR/extensions-v8-latest.yaml

RUN scripts/install_extensions_v8.sh extensions-v8-latest.yaml $LANDIS_DIR

## DEBUG: extension install and dependencies
# RUN echo "$(cat $EXT_LOG_FILE)" 1>&2 && exit 1

## (TODO) resume here
## SCRPPLE test DGS compliant (this is the version within ML's personal repo- WORKS)
RUN cd /opt/Core-Model-v8-LINUX && \ 
  git clone $LANDIS_GITHUB/Extension-Social-Climate-Fire.git && \ 
  cd Extension-Social-Climate-Fire && \ 
  #git checkout b463ea378f1bcde4369907a408dfe64b9cc52c7a && \  
  cd /opt/Core-Model-v8-LINUX/Extension-Social-Climate-Fire/src && \
  python3 /opt/Core-Model-v8-LINUX/editing_csproj_LANDIS-II_files.py ./SocialClimateFire.csproj && \
  dotnet build -c Release && \
  dotnet /opt/Core-Model-v8-LINUX/build/Release/Landis.Extensions.dll add "/opt/Core-Model-v8-LINUX/Extension-Social-Climate-Fire/deploy/installer/Scrapple 4.txt" 

## Output Biomass (WORKS)
RUN wget -O master.zip $LANDIS_GITHUB/Extension-Output-Biomass/archive/refs/heads/master.zip && \
  unzip master.zip -d /opt/Core-Model-v8-LINUX/ && \
  rm master.zip && \
  cd /opt/Core-Model-v8-LINUX/Extension-Output-Biomass-master/src && \
  #cp /opt/Core-Model-v8-LINUX-main/build/extensions/Support-Library-Dlls-v8-main/*.dll /opt/Core-Model-v8-LINUX-main/build/extensions/Extension-Output-Biomass-master/src/lib/ && \
  python3 /opt/Core-Model-v8-LINUX/editing_csproj_LANDIS-II_files.py ./output-biomass.csproj && \
  dotnet build -c Release && \
  dotnet /opt/Core-Model-v8-LINUX/build/Release/Landis.Extensions.dll add "/opt/Core-Model-v8-LINUX/Extension-Output-Biomass-master/deploy/installer/Output Biomass 4.txt"
  
# Output Biomass Community 
RUN wget -O master.zip $LANDIS_GITHUB/Extension-Output-Biomass-Community/archive/refs/heads/master.zip && \
  unzip master.zip -d /opt/Core-Model-v8-LINUX/ && \
  rm master.zip && \
  cd /opt/Core-Model-v8-LINUX/Extension-Output-Biomass-Community-master/src && \
  python3 /opt/Core-Model-v8-LINUX/editing_csproj_LANDIS-II_files.py ./output-biomass-community.csproj && \
  dotnet build -c Release && \
  dotnet /opt/Core-Model-v8-LINUX/build/Release/Landis.Extensions.dll add "/opt/Core-Model-v8-LINUX/Extension-Output-Biomass-Community-master/deploy/installer/Output Biomass Community 3.txt"
  
# Output Biomass Reclass
RUN wget -O master.zip $LANDIS_GITHUB/Extension-Output-Biomass-Reclass/archive/refs/heads/master.zip && \
  unzip master.zip -d /opt/Core-Model-v8-LINUX/ && \
  rm master.zip && \
  cd /opt/Core-Model-v8-LINUX/Extension-Output-Biomass-Reclass-master/src && \ 
  python3 /opt/Core-Model-v8-LINUX/editing_csproj_LANDIS-II_files.py ./output-biomass.csproj && \
  dotnet build -c Release && \
  dotnet /opt/Core-Model-v8-LINUX/build/Release/Landis.Extensions.dll add "/opt/Core-Model-v8-LINUX/Extension-Output-Biomass-Reclass-master/deploy/installer/Output Biomass Reclass 4.txt"

# Output Max Spp Age
RUN wget -O master.zip $LANDIS_GITHUB/Extension-Output-Max-Species-Age.git && \
  unzip master.zip -d /opt/Core-Model-v8-LINUX/ && \
  rm master.zip && \
  cd /opt/Core-Model-v8-LINUX/Extension-Output-Max-Species-Age-master/src && \ 
  python3 /opt/Core-Model-v8-LINUX/editing_csproj_LANDIS-II_files.py ./max-species-age.csproj && \
  dotnet build -c Release && \
  dotnet /opt/Core-Model-v8-LINUX/build/Release/Landis.Extensions.dll add "/opt/Core-Model-v8-LINUX/Extension-Output-Max-Species-Age-master/deploy/installer/Output MaxSpeciesAge 4.txt"

# GIPL (WORKS)
RUN wget $LANDIS_GITHUB/Library-GIPL/archive/master.zip && \
  unzip master.zip -d /opt/Core-Model-v8-LINUX/ && \
  rm master.zip && \
  mv /opt/Core-Model-v8-LINUX/Library-GIPL-master/ /opt/Core-Model-v8-LINUX/Library-GIPL/ && \
  cd /opt/Core-Model-v8-LINUX/Library-GIPL/src && \ 
  python3 /opt/Core-Model-v8-LINUX/editing_csproj_LANDIS-II_files.py ./GiplDamm.csproj && \
  dotnet build -c Release && \
  cp /opt/Core-Model-v8-LINUX/Library-GIPL/src/bin/Debug/netstandard2.0/Landis.Extension.GiplDamm.dll /opt/Core-Model-v8-LINUX/build/extensions/.  

# SHAW (WORKS)
RUN wget $LANDIS_GITHUB/Library-Shaw/archive/refs/heads/master.zip && \
  unzip master.zip -d /opt/Core-Model-v8-LINUX && \
  rm master.zip && \
  mv /opt/Core-Model-v8-LINUX/Library-Shaw-master/ /opt/Core-Model-v8-LINUX/Library-Shaw/ && \
  cd /opt/Core-Model-v8-LINUX/Library-Shaw/src && \
  python3 /opt/Core-Model-v8-LINUX/editing_csproj_LANDIS-II_files.py ./ShawDammNewInputs.csproj && \
  dotnet build -c Release && \
  cp /opt/Core-Model-v8-LINUX/Library-Shaw/src/bin/Debug/netstandard2.0/Landis.Extension.ShawDamm.dll /opt/Core-Model-v8-LINUX/build/extensions/. 

# Biomass Succession
  RUN cd /opt/Core-Model-v8-LINUX/ && \
  git clone $LANDIS_GITHUB/Extension-Biomass-Succession.git && \ 
  cd /opt/Core-Model-v8-LINUX/Extension-Biomass-Succession && \
  #git checkout 58ad3673e02abe82f437a6b68c44220c51351091 && \ 
  cd /opt/Core-Model-v8-LINUX/Extension-Biomass-Succession/src && \
  python3 /opt/Core-Model-v8-LINUX/editing_csproj_LANDIS-II_files.py ./biomass-succession.csproj && \
  dotnet build -c Release && \
  dotnet /opt/Core-Model-v8-LINUX/build/Release/Landis.Extensions.dll add "/opt/Core-Model-v8-LINUX/Extension-Biomass-Succession/deploy/installer/Biomass Succession 7.txt"

# NECN (WORKS)
RUN cd /opt/Core-Model-v8-LINUX/ && \
  git clone $LANDIS_GITHUB/Extension-NECN-Succession.git && \
  #cd /opt/Core-Model-v8-LINUX/Extension-NECN-Succession && \ 
  #git checkout 37ce246c37bab3448e3db134373deb56063e14ac && \
  cd /opt/Core-Model-v8-LINUX/Extension-NECN-Succession/src && \   
  python3 /opt/Core-Model-v8-LINUX/editing_csproj_LANDIS-II_files.py ./NECN-succession.csproj && \
  dotnet build -c Release && \
  dotnet /opt/Core-Model-v8-LINUX/build/Release/Landis.Extensions.dll add "/opt/Core-Model-v8-LINUX/Extension-NECN-Succession/deploy/installer/NECN_Succession8.txt"

# DGS (WORKS)
RUN wget $LANDIS_GITHUB/Extension-DGS-Succession/archive/refs/heads/v8.zip && \
  unzip v8.zip -d /opt/Core-Model-v8-LINUX/ && \
  rm v8.zip && \
  mv /opt/Core-Model-v8-LINUX/Extension-DGS-Succession-8/ /opt/Core-Model-v8-LINUX/Extension-DGS-Succession/ && \
  #best bet is to remove the main support dlls from DGS repo then push latest
  rm /opt/Core-Model-v8-LINUX/Extension-DGS-Succession/src/lib/*.dll && \
  #cp /opt/Core-Model-v8-LINUX/Extension-NECN-Succession/src/lib/*.dll /opt/Core-Model-v8-LINUX/Extension-DGS-Succession/src/lib/ && \
  cd /opt/Core-Model-v8-LINUX/Extension-DGS-Succession/src && \
  python3 /opt/Core-Model-v8-LINUX/editing_csproj_LANDIS-II_files.py ./DGS-succession.csproj && \
  dotnet build -c Release && \
  dotnet /opt/Core-Model-v8-LINUX/build/Release/Landis.Extensions.dll add "/opt/Core-Model-v8-LINUX/Extension-DGS-Succession/deploy/installer/DGS_Succession 1.102.txt"

RUN cd /opt/Core-Model-v8-LINUX && \
  rm /opt/Core-Model-v8-LINUX/build/extensions/Landis.Library.InitialCommunity.Universal.dll && \
  wget $LANDIS_GITHUB/Library-Initial-Community/archive/refs/heads/master.zip && \
  mkdir /opt/Core-Model-v8-LINUX/Initial-Community-main/ && \
  unzip master.zip -d /opt/Core-Model-v8-LINUX/Initial-Community-main/ && \
  rm master.zip && \
  cd /opt/Core-Model-v8-LINUX/Initial-Community-main/Library-Initial-Community-master/ && \
  python3 /opt/Core-Model-v8-LINUX/editing_csproj_LANDIS-II_files.py ./initial-community.csproj && \ 
  dotnet build -c Release

# Add a hint path for all of the extensions to the Tool-Console csproj file and then rebuild it 
# need to add max species age just not sure on syntax
RUN sed -i '/<\/Project>/i \
<ItemGroup>\n\
  <Reference Include="Landis.Extension.Succession.Biomass-v7.dll">\n\
    <HintPath>../../build/extensions/Landis.Extension.Succession.Biomass-v7.dll</HintPath>\n\
  </Reference>\n\
  <Reference Include="Landis.Library.Metadata-v2">\n\
    <HintPath>../../build/extensions/Landis.Library.Metadata-v2.dll</HintPath>\n\
  </Reference>\n\
  <Reference Include="Landis.Library.Succession-v9.dll">\n\
    <HintPath>../../build/extensions/Landis.Library.Succession-v9.dll</HintPath>\n\
  </Reference>\n\
  <Reference Include="Landis.Extension.Succession.NECN-v8.dll">\n\
    <HintPath>../../build/extensions/Landis.Extension.Succession.NECN-v8.dll</HintPath>\n\
  </Reference>\n\
  <Reference Include="Landis.Extension.SocialClimateFire-v4.dll">\n\
    <HintPath>../../build/extensions/Landis.Extension.SocialClimateFire-v4.dll</HintPath>\n\
  </Reference>\n\
  <Reference Include="Landis.Extension.GiplDamm.dll">\n\
    <HintPath>../../build/extensions/Landis.Extension.GiplDamm.dll</HintPath>\n\
  </Reference>\n\
  <Reference Include="Landis.Extension.Output.Biomass-v4.dll">\n\
    <HintPath>../../build/extensions/Landis.Extension.Output.Biomass-v4.dll</HintPath>\n\
  </Reference>\n\
  <Reference Include="Landis.Extension.Output.BiomassCommunity-v3.dll">\n\
    <HintPath>../../build/extensions/Landis.Extension.Output.BiomassCommunity-v3.dll</HintPath>\n\
  </Reference>\n\
  <Reference Include="Landis.Library.UniversalCohorts-v1.dll">\n\
    <HintPath>../../build/extensions/Landis.Library.UniversalCohorts-v1.dll</HintPath>\n\
  </Reference>\n\
  <Reference Include="Landis.Extension.Output.BiomassReclass-v4.dll">\n\
    <HintPath>../../build/extensions/Landis.Extension.Output.BiomassReclass-v4.dll</HintPath>\n\
  </Reference>\n\
  <Reference Include="Landis.Extension.ShawDamm.dll">\n\
    <HintPath>../../build/extensions/Landis.Extension.ShawDamm.dll</HintPath>\n\
  </Reference>\n\
  <Reference Include="Landis.Extension.Succession.DGS.dll">\n\
    <HintPath>../../build/extensions/Landis.Extension.Succession.DGS.dll</HintPath>\n\
  </Reference>\n\
  <Reference Include="Landis.Library.BiomassHarvest-v3.dll">\n\
    <HintPath>../../build/extensions/Landis.Library.BiomassHarvest-v3.dll</HintPath>\n\
  </Reference>\n\
  <Reference Include="Landis.Library.Climate-v5.dll">\n\
    <HintPath>../../build/extensions/Landis.Library.Climate-v5.dll</HintPath>\n\
  </Reference>\n\
  <Reference Include="Landis.Library.HarvestManagement-v4.dll">\n\
    <HintPath>../../build/extensions/Landis.Library.HarvestManagement-v4.dll</HintPath>\n\
  </Reference>\n\
  <Reference Include="Landis.Library.HarvestManagement-v4.dll">\n\
    <HintPath>../../build/extensions/Landis.Library.HarvestManagement-v4.dll</HintPath>\n\
  </Reference>\n\
  <Reference Include="log4net.dll">\n\
    <HintPath>../../build/extensions/log4net.dll</HintPath>\n\
  </Reference>\n\
  <Reference Include="Landis.Library.UniversalCohorts-v1.dll">\n\
    <HintPath>../../build/extensions/Landis.Library.UniversalCohorts-v1.dll</HintPath>\n\
  </Reference>\n\
</ItemGroup>' /opt/Core-Model-v8-LINUX/Tool-Console/src/Console.csproj

# Move Support Libraries to the correct place
#RUN mv /opt/Core-Model-v8-LINUX-main/build/extensions/Support-Library-Dlls-v8-main/*.dll /opt/Core-Model-v8-LINUX-main/build/extensions/

RUN chown -R 1000:1000 /opt/Core-Model-v8-LINUX
  
ARG LOCAL_USER=user
ARG PRIV_CMDS='/bin/ch*,/bin/cat,/bin/gunzip,/bin/tar,/bin/mkdir,/bin/ps,/bin/mv,/bin/cp,/usr/bin/apt*,/usr/bin/pip*,/bin/yum'

RUN adduser --disabled-password --gecos "VICE_User" --uid 1000 user  && \
  usermod -aG sudo user && \
  echo "$LOCAL_USER ALL=NOPASSWD: $PRIV_CMDS" >> /etc/sudoers

RUN apt-get update && \
  apt-get install -y wget bzip2 ca-certificates \
  libglib2.0-0 libxext6 libsm6 libxrender1 \
  gettext-base git mercurial subversion \
  tmux && \
  apt-get clean && \
  rm -rf /var/lib/apt/lists/*

RUN wget --quiet https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O ~/miniconda.sh && \
  /bin/bash ~/miniconda.sh -b -p /opt/conda && \
  rm ~/miniconda.sh && \
  ln -s /opt/conda/etc/profile.d/conda.sh /etc/profile.d/conda.sh && \
  echo ". /opt/conda/etc/profile.d/conda.sh" >> /home/user/.bashrc && \
  echo ". /opt/conda/etc/profile.d/conda.sh" >> /home/user/.zshrc && \
  chown -R user:user /opt/conda

# install ttyd
RUN curl -L "https://github.com/tsl0922/ttyd/releases/download/1.6.3/ttyd.x86_64" > ttyd && \
  chmod a+x ttyd && \
  mv ttyd /usr/local/bin/ttyd

RUN apt-get update && \
  apt-get install -y curl grep sed dpkg && \
  curl -L "https://github.com/krallin/tini/releases/download/v0.19.0/tini_0.19.0-amd64.deb" > tini.deb && \
  dpkg -i tini.deb && \
  rm tini.deb && \
  apt-get clean && \
  rm -rf /var/lib/apt/lists/*

# Install a few dependencies for iCommands, text editing, and monitoring instances
RUN apt-get update && \
  apt-get install -y lsb-release apt-transport-https curl gnupg2 libfuse2 gcc less nodejs software-properties-common apt-utils glances htop nano  && \
  apt-get clean && \
  rm -rf /var/lib/apt/lists/*

RUN apt-get update && \
  apt-get install -y vim-nox && \
  apt-get clean && \
  rm -rf /var/lib/apt/lists/*

RUN apt-get update && \
  apt-get install -y emacs-nox && \
  apt-get clean && \
  rm -rf /var/lib/apt/lists/*

RUN wget -qO - https://packages.irods.org/irods-signing-key.asc | apt-key add - && \
  echo "deb [arch=amd64] https://packages.irods.org/apt/ $(lsb_release -sc) main" >> /etc/apt/sources.list.d/renci-irods.list && \
  apt-get update && \
  apt install -y irods-icommands 

USER user

# set shell as bash and terminal as linux
ENV SHELL=bash
ENV TERM=xterm

# open port 7681 for ttyd
EXPOSE 7681

# changes tmux layout while running
COPY entry.sh /bin
RUN echo 'set-option -g status off' >> ~/.tmux.conf

# add iRODS iCommands to user profile as JSON
RUN mkdir /home/user/.irods 

#RUN cd /opt/Core-Model-v8-LINUX-main/Tool-Extension-Admin/src && dotnet build -c Release
RUN echo "alias console='dotnet /opt/Core-Model-v8-LINUX/build/Release/Landis.Console.dll'" >> ~/.bashrc
#RUN echo "cd /opt/Core-Model-v8-LINUX-main/Tool-Console/src && dotnet build -c Release && cd" >> ~/.bashrc
RUN cd /opt/Core-Model-v8-LINUX/Tool-Console/src && dotnet build -c Release
ENTRYPOINT ["bash", "/bin/entry.sh"]

CMD ["ttyd", "bash"]

WORKDIR /home/user
