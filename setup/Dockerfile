FROM jupyter/minimal-notebook:lab-4.0.7

# Set variables
ARG JUPYTER_KERNEL_NAME=omics-genomika \
    NOTEBOOKS_REPO=https://github.com/IMID/edugen_pub \
    USER_NAME=jovyan \
    #JAVA_VERSION=8.0.292.hs-adpt \
    PIP_VERSION=21.0.1 \
    IGV_JUPYTER_VERSION=2.0.4 \
    GATK_VERSION=4.1.9.0 \
    BWA_VERSION=0.7.18 \
    PYTHON_VERSION=3.9 \
    HTSLIB_VERSION=1.21 \
    BEDTOOLS_VERSION=2.31.1 \
    GATK_VERSION=4.6.0.0

# Install packages
USER root
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
    wget && \
    rm -rf /var/lib/apt/lists/*

# Install gsutil
RUN echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] http://packages.cloud.google.com/apt cloud-sdk main" | tee -a /etc/apt/sources.list.d/google-cloud-sdk.list && \
    curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key --keyring /usr/share/keyrings/cloud.google.gpg add - && \
    apt-get update && apt-get install -y google-cloud-sdk && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /tmp
    
# Install vt
RUN git clone https://github.com/atks/vt.git && \
    cd vt && \
    git checkout 0.57721 && \
    make && \
    mv vt /usr/bin

# Install BWA
RUN git clone https://github.com/lh3/bwa.git && \
    cd bwa && \
    make && \
    mv bwa /usr/bin

# Install HTSlib
RUN wget https://github.com/samtools/htslib/releases/download/$HTSLIB_VERSION/htslib-$HTSLIB_VERSION.tar.bz2 && \
    tar --bzip2 -xf htslib-$HTSLIB_VERSION.tar.bz2 && \
    cd htslib-$HTSLIB_VERSION && \
    ./configure --prefix=/opt/htslib && \
    make && \
    make install
ENV PATH="/opt/htslib/bin:${PATH}"

# Install Samtools
RUN wget https://github.com/samtools/samtools/releases/download/$HTSLIB_VERSION/samtools-$HTSLIB_VERSION.tar.bz2 && \
    tar --bzip2 -xf samtools-$HTSLIB_VERSION.tar.bz2 && \
    cd samtools-$HTSLIB_VERSION && \
    ./configure --prefix=/opt/samtools && \
    make && \
    make install
ENV PATH="/opt/samtools/bin:${PATH}"

# Install BCFtools
RUN wget https://github.com/samtools/bcftools/releases/download/$HTSLIB_VERSION/bcftools-$HTSLIB_VERSION.tar.bz2 && \
    tar --bzip2 -xf bcftools-$HTSLIB_VERSION.tar.bz2 && \
    cd bcftools-$HTSLIB_VERSION && \
    ./configure --prefix=/opt/bcftools && \
    make && \
    make install
ENV PATH="/opt/bcftools/bin:${PATH}"

# Install BEDtools2
RUN wget https://github.com/arq5x/bedtools2/releases/download/v$BEDTOOLS_VERSION/bedtools-$BEDTOOLS_VERSION.tar.gz && \
	  tar zxvf bedtools-$BEDTOOLS_VERSION.tar.gz && \
	  cd bedtools2 && \
	  make && \
    mv /tmp/bedtools2 /opt/
ENV PATH="/opt/bedtools2/bin/:${PATH}"

# Install FastQC
RUN wget https://www.bioinformatics.babraham.ac.uk/projects/fastqc/fastqc_v0.12.1.zip && \
    unzip fastqc_v0.12.1.zip && \
    mv /tmp/FastQC /opt/
ENV PATH="/opt/FastQC/:${PATH}"

# Install MultiQC
RUN pip install multiqc==1.25.1

# Install GATK
RUN wget https://github.com/broadinstitute/gatk/releases/download/$GATK_VERSION/gatk-$GATK_VERSION.zip && \
    unzip gatk-$GATK_VERSION.zip && \
    mv /tmp/gatk-$GATK_VERSION /opt/gatk
ENV PATH="/opt/gatk/:${PATH}"

# Clean-up
RUN rm -rf /tmp/*
RUN rm /bin/sh && ln -s /bin/bash /bin/sh

# Jupyter user setup
ENV HOME=/tmp/$USER_NAME
RUN mkdir -p $HOME && chown -R $USER_NAME $HOME && \
    mkdir -p $HOME/work/git && \
    cd $HOME/work/git && \
    git clone $NOTEBOOKS_REPO && \
    chown -R jovyan $HOME/work/git

WORKDIR $HOME
USER $NB_USER

# Create venv and install kernel
RUN source /opt/conda/etc/profile.d/conda.sh && \
    conda create python=$PYTHON_VERSION -p $HOME/venv/$JUPYTER_KERNEL_NAME -y && \
    conda activate $HOME/venv/$JUPYTER_KERNEL_NAME && \
    pip install -U ipykernel==6.29.5 \
        numpy==2.0.2 \
        pandas==2.2.3 \
        pysam==0.22.1 \
        biothings_client==0.3.1 \
        matplotlib==3.9.2 \
        igv_jupyterlab==1.0.1 \
        gsutil==5.30 && \
    python -m ipykernel install --user --name $JUPYTER_KERNEL_NAME --display-name $JUPYTER_KERNEL_NAME && \
    conda deactivate
