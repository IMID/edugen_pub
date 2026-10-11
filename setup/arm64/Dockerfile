# omics-genomika — natywny obraz arm64 (Apple Silicon)
# Oparty na https://github.com/IMID/edugen_pub/blob/main/setup/Dockerfile
# Zmiany wobec originalu sa oznaczone [FIX].
FROM jupyter/minimal-notebook:lab-4.0.7

ARG JUPYTER_KERNEL_NAME=omics-genomika
ARG NOTEBOOKS_REPO=https://github.com/IMID/edugen_pub
ARG USER_NAME=jovyan
ARG PYTHON_VERSION=3.9
ARG HTSLIB_VERSION=1.21
ARG BEDTOOLS_VERSION=2.31.1
ARG GATK_VERSION=4.6.0.0
ARG FASTQC_VERSION=0.12.1

USER root
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# [FIX] bwa i vt z repozytorium Ubuntu (arm64) — oba uzywaja intrinsics SSE2,
# wiec "make" z originalnego Dockerfile'a nie kompiluje sie na ARM.
RUN apt-get update && apt-get -qq -y install --no-install-recommends \
    apt-transport-https \
    ca-certificates \
    curl \
    default-jre \
    g++ \
    gcc \
    git \
    gnupg \
    libbz2-dev \
    libcurl4-gnutls-dev \
    libdeflate-dev \
    libffi-dev \
    liblzma-dev \
    libncurses5-dev \
    libssl-dev \
    libz-dev \
    lsb-release \
    make \
    openssl \
    unzip \
    zlib1g-dev \
    zip \
    wget \
    bwa \
    vt && \
    rm -rf /var/lib/apt/lists/* && \
    { bwa 2>&1 || true; } | head -3 && { vt 2>&1 || true; } | head -3

# Install gsutil ([FIX] repo z jawna architektura + nowy keyring zamiast apt-key;
# Google przeniosl pakiet "google-cloud-sdk" -> "google-cloud-cli", stara nazwa juz nie istnieje)
RUN curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg \
      | gpg --dearmor -o /usr/share/keyrings/cloud.google.gpg && \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/cloud.google.gpg] http://packages.cloud.google.com/apt cloud-sdk main" \
      > /etc/apt/sources.list.d/google-cloud-sdk.list && \
    apt-get update && apt-get install -y --no-install-recommends google-cloud-cli && \
    rm -rf /var/lib/apt/lists/* && \
    gsutil version

WORKDIR /tmp

# Install HTSlib (kompilacja natywna arm64)
RUN wget -q https://github.com/samtools/htslib/releases/download/$HTSLIB_VERSION/htslib-$HTSLIB_VERSION.tar.bz2 && \
    tar --bzip2 -xf htslib-$HTSLIB_VERSION.tar.bz2 && \
    cd htslib-$HTSLIB_VERSION && \
    ./configure --prefix=/opt/htslib && \
    make -j"$(nproc)" && make install
ENV PATH="/opt/htslib/bin:${PATH}"

# Install Samtools
RUN wget -q https://github.com/samtools/samtools/releases/download/$HTSLIB_VERSION/samtools-$HTSLIB_VERSION.tar.bz2 && \
    tar --bzip2 -xf samtools-$HTSLIB_VERSION.tar.bz2 && \
    cd samtools-$HTSLIB_VERSION && \
    ./configure --prefix=/opt/samtools && \
    make -j"$(nproc)" && make install
ENV PATH="/opt/samtools/bin:${PATH}"

# Install BCFtools
RUN wget -q https://github.com/samtools/bcftools/releases/download/$HTSLIB_VERSION/bcftools-$HTSLIB_VERSION.tar.bz2 && \
    tar --bzip2 -xf bcftools-$HTSLIB_VERSION.tar.bz2 && \
    cd bcftools-$HTSLIB_VERSION && \
    ./configure --prefix=/opt/bcftools && \
    make -j"$(nproc)" && make install
ENV PATH="/opt/bcftools/bin:${PATH}"

# Install BEDtools2
RUN wget -q https://github.com/arq5x/bedtools2/releases/download/v$BEDTOOLS_VERSION/bedtools-$BEDTOOLS_VERSION.tar.gz && \
    tar zxf bedtools-$BEDTOOLS_VERSION.tar.gz && \
    cd bedtools2 && \
    make -j"$(nproc)" && \
    mv /tmp/bedtools2 /opt/
ENV PATH="/opt/bedtools2/bin/:${PATH}"

# Install FastQC (Java — niezalezne od architektury)
RUN wget -q https://www.bioinformatics.babraham.ac.uk/projects/fastqc/fastqc_v${FASTQC_VERSION}.zip && \
    unzip -q fastqc_v${FASTQC_VERSION}.zip && \
    chmod +x /tmp/FastQC/fastqc && \
    mv /tmp/FastQC /opt/
ENV PATH="/opt/FastQC/:${PATH}"

# Install MultiQC
RUN pip install --no-cache-dir multiqc==1.25.1

# Install GATK (Java)
RUN wget -q https://github.com/broadinstitute/gatk/releases/download/$GATK_VERSION/gatk-$GATK_VERSION.zip && \
    unzip -q gatk-$GATK_VERSION.zip && \
    mv /tmp/gatk-$GATK_VERSION /opt/gatk
ENV PATH="/opt/gatk/:${PATH}"

# Clean-up
RUN rm -rf /tmp/*
RUN rm /bin/sh && ln -s /bin/bash /bin/sh

# [FIX] ipywidgets w srodowisku serwera (potrzebne do widgetow IGV)
RUN pip install --no-cache-dir ipywidgets==8.1.5

# Jupyter user setup
# [FIX] HOME zostaje /home/jovyan (original ustawial /tmp/jovyan — czyszczone przy
# restarcie i niezgodne z montowaniem -v ...:/home/jovyan/work)
ENV HOME=/home/${USER_NAME}
RUN mkdir -p $HOME/work/git && \
    cd $HOME/work/git && \
    git clone --depth 1 $NOTEBOOKS_REPO && \
    cp -r $HOME/work/git/edugen_pub /opt/edugen_pub && \
    chown -R $USER_NAME:users $HOME

WORKDIR $HOME
USER $USER_NAME

# Create venv and install kernel
# [FIX] srodowisko kernela trzyma TYLKO ipykernel + biblioteki naukowe.
# W originale "igv_jupyterlab" i "gsutil" wciagaly tu stary stos Jupytera
# (jupyterlab 3.6.8 / notebook 6.5.7 / jupyter_client 7.4.9), po czym kernel
# bindowal gniazda ZMQ i nigdy nie odpowiadal na kernel_info.
# gsutil jest dostarczany przez google-cloud-sdk (CLI), wiec nie jest tu potrzebny.
RUN source /opt/conda/etc/profile.d/conda.sh && \
    conda create -y -q python=$PYTHON_VERSION -p $HOME/venv/$JUPYTER_KERNEL_NAME && \
    conda activate $HOME/venv/$JUPYTER_KERNEL_NAME && \
    pip install --no-cache-dir -U \
        ipykernel==6.29.5 \
        numpy==2.0.2 \
        pandas==2.2.3 \
        pysam==0.22.1 \
        biothings_client==0.3.1 \
        matplotlib==3.9.2 \
        ipywidgets==8.1.5 \
        igv-notebook==0.5.2 && \
    pip install --no-cache-dir --no-deps igv_jupyterlab==1.0.1 && \
    python -m ipykernel install --user --name $JUPYTER_KERNEL_NAME --display-name $JUPYTER_KERNEL_NAME && \
    python -c "import numpy,pandas,pysam,matplotlib,biothings_client,igv_notebook,igv_jupyterlab;print('kernel deps OK')" && \
    pip list | grep -iE "jupyterlab|notebook|jupyter.client" || echo "kernel env clean: brak stosu serwera Jupyter" && \
    conda deactivate

WORKDIR $HOME/work
