# Copyright 2021 DeepMind Technologies Limited
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

ARG CUDA=11.1.1
#FROM nvidia/cuda:${CUDA}-cudnn8-devel-ubuntu18.04
FROM nvcr.io/nvidia/jax:23.10-py3
# FROM directive resets ARGS, so we specify again (the value is retained if
# previously set).
ARG CUDA

# Use bash to support string substitution.
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN apt-get update 

RUN DEBIAN_FRONTEND=noninteractive apt-get install --no-install-recommends -y \
        build-essential \
        cmake \
        git \
        kalign \
        tzdata \
        wget \
        doxygen \
        cython3 \
        swig \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get autoremove -y \
    && apt-get clean


# compile hmmer from source
RUN mkdir /hmmer_src
WORKDIR /hmmer_src
RUN wget http://eddylab.org/software/hmmer/hmmer-3.4.tar.gz
RUN tar xzf hmmer-3.4.tar.gz
WORKDIR /hmmer_src/hmmer-3.4
RUN ./configure
RUN make
RUN make install

# Compile HHsuite from source.
RUN git clone --branch v3.3.0 https://github.com/soedinglab/hh-suite.git /tmp/hh-suite \
    && mkdir /tmp/hh-suite/build \
    && pushd /tmp/hh-suite/build \
    && cmake -DCMAKE_INSTALL_PREFIX=/opt/hhsuite .. \
    && make -j 4 && make install \
    && ln -s /opt/hhsuite/bin/* /usr/bin \
    && popd \
    && rm -rf /tmp/hh-suite

# Install Miniconda package manager.
#RUN wget -q -P /tmp \
#  https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-aarch64.sh \
#    && bash /tmp/Miniconda3-latest-Linux-aarch64.sh -b -p /opt/conda \
#    && rm /tmp/Miniconda3-latest-Linux-aarch64.sh

# Install conda packages.
ENV PATH="/opt/conda/bin:$PATH"
ENV LD_LIBRARY_PATH="/opt/conda/lib:$LD_LIBRARY_PATH"
#RUN conda install -qy conda==23.5.2 \
#      openmm=7.7.0 \
#      pdbfixer \
#      && conda clean --all --force-pkgs-dirs --yes

#RUN conda install -y -c conda-forge pdbfixer openmm

RUN wget -q -P /app/alphafold/alphafold/common/ \
  https://git.scicore.unibas.ch/schwede/openstructure/-/raw/7102c63615b64735c4941278d92b554ec94415f8/modules/mol/alg/src/stereo_chemical_props.txt




# Add SETUID bit to the ldconfig binary so that non-root users can run it.
RUN chmod u+s /sbin/ldconfig.real

# We need to run `ldconfig` first to ensure GPUs are visible, due to some quirk
# with Debian. See https://github.com/NVIDIA/nvidia-docker/issues/1399 for
# details.
# ENTRYPOINT does not support easily running multiple commands, so instead we
# write a shell script to wrap them up.


#copile openmm from source
RUN mkdir /openmm_build
WORKDIR /openmm_build
RUN wget https://github.com/openmm/openmm/archive/refs/tags/7.7.0.tar.gz
RUN tar xzf 7.7.0.tar.gz
WORKDIR /openmm_build/openmm-7.7.0
RUN mkdir build
WORKDIR /openmm_build/openmm-7.7.0/build
RUN cmake ..
RUN make
RUN make install
RUN make PythonInstall

#compile pdbfixer
RUN mkdir /pdbfixer_build
WORKDIR /pdbfixer_build
RUN git clone https://github.com/openmm/pdbfixer.git
WORKDIR /pdbfixer_build/pdbfixer
RUN python setup.py install

COPY . /app/alphafold
# Install pip packages.
RUN pip3 install --upgrade pip --no-cache-dir \
    && pip3 install -r /app/alphafold/requirements.txt --no-cache-dir 

WORKDIR /app/alphafold
RUN echo $'#!/bin/bash\n\
ldconfig\n\
python /app/alphafold/run_alphafold.py "$@"' > /app/run_alphafold.sh \
  && chmod +x /app/run_alphafold.sh
ENTRYPOINT ["/app/run_alphafold.sh"]
