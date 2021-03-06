ARG VERSION_ID
ARG DEPENDENCIES_IMAGE
ARG IMAGE_NAME
FROM ${DEPENDENCIES_IMAGE} as dependencies

ARG VERSION_ID
# using a smaller image doesn't save us any space as the layers are used by the other images.
FROM ubuntu:${VERSION_ID} as qemu

# install qemu for the support of building containers on host
RUN apt-get update && apt-get install -y --no-install-recommends qemu-user-static && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# start of the real image base
ARG VERSION_ID
FROM arm64v8/ubuntu:${VERSION_ID}

LABEL maintainer "Ian Davis"

# TODO: Set up conditional QEMU support for those that haven't configured binfmt.service.
# This is small overhead (3-4 MiB) and helps people fall into the pit of success.
COPY --from=qemu /usr/bin/qemu-aarch64-static /usr/bin/qemu-aarch64-static

RUN apt-get update && apt-get install -y --no-install-recommends \
    bzip2 \
    lbzip2 \
    ca-certificates \
    gnupg \
    sudo

# Copy the driver pack and verify
ARG DEPENDENCIES_IMAGE
COPY --from=dependencies /data/Jetson-210_Linux_R32.4.3_aarch64.tbz2 Jetson-210_Linux_R32.4.3_aarch64.tbz2
RUN echo "048ed5e9f225896b86ed2d5b0ca803fa *./Jetson-210_Linux_R32.4.3_aarch64.tbz2" | md5sum -c -
RUN tar -xpj --overwrite -f ./Jetson-210_Linux_R32.4.3_aarch64.tbz2
RUN sed -i '/.*tar -I lbzip2 -xpmf \"${LDK_NV_TEGRA_DIR}\/config\.tbz2\".*/c\tar -I lbzip2 -xpm --overwrite -f \"${LDK_NV_TEGRA_DIR}\/config.tbz2\"' ./Linux_for_Tegra/apply_binaries.sh
# pre-req for Linux_for_Tegra/nv_tegra/nv-apply-debs.sh as it tries to chroot
COPY --from=qemu /usr/bin/qemu-aarch64-static /Linux_for_Tegra/nv_tegra/qemu-aarch64-static
RUN sed -i '/.*	LC_ALL=C chroot . dpkg -i --path-include=\"\/usr\/share\/doc\/\*\" \"${pre_deb_list\[\@\]}\".*/c\	apt install -y --no-install-recommends \"${pre_deb_list[@]}\"' ./Linux_for_Tegra/nv_tegra/nv-apply-debs.sh
RUN sed -i '/.*LC_ALL=C chroot . dpkg -i --path-include=\"\/usr\/share\/doc\/\*\" \"${deb_list\[\@\]}\".*/c\apt install -y --no-install-recommends \"${deb_list[@]}\"' ./Linux_for_Tegra/nv_tegra/nv-apply-debs.sh
RUN sed -i '/.*rm -f "${L4T_ROOTFS_DIR}\/usr\/bin\/qemu-aarch64-static\".*/c\#rm -f "${L4T_ROOTFS_DIR}\/usr\/bin\/qemu-aarch64-static\"' ./Linux_for_Tegra/nv_tegra/nv-apply-debs.sh

# In 32.4.3, this will use deb packages instead of the archive overlay
RUN ./Linux_for_Tegra/apply_binaries.sh -r /

# Patch OTA apt source
RUN sed -i "s/<SOC>/t210/g" "/etc/apt/sources.list.d/nvidia-l4t-apt-source.list"

# Cleanup so --squash will be effective
RUN rm -rf ./Linux_for_Tegra && \
    rm ./Jetson-210_Linux_R32.4.3_aarch64.tbz2 \
    && \
    apt-get purge --autoremove -y bzip2 lbzip2 && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

ENV LD_LIBRARY_PATH=/usr/lib/aarch64-linux-gnu/tegra:/usr/lib/aarch64-linux-gnu/tegra-egl:${LD_LIBRARY_PATH}

RUN ln -s /usr/lib/aarch64-linux-gnu/tegra/libnvidia-ptxjitcompiler.so.32.4.3 /usr/lib/aarch64-linux-gnu/tegra/libnvidia-ptxjitcompiler.so
RUN ln -s /usr/lib/aarch64-linux-gnu/tegra/libnvidia-ptxjitcompiler.so.32.4.3 /usr/lib/aarch64-linux-gnu/tegra/libnvidia-ptxjitcompiler.so.1
RUN ln -sf /usr/lib/aarch64-linux-gnu/tegra/libGL.so /usr/lib/aarch64-linux-gnu/libGL.so
RUN ln -s /usr/lib/aarch64-linux-gnu/libcuda.so /usr/lib/aarch64-linux-gnu/libcuda.so.1
RUN ln -sf /usr/lib/aarch64-linux-gnu/tegra-egl/libEGL.so /usr/lib/aarch64-linux-gnu/libEGL.so
RUN ldconfig