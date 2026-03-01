##########################################################################
# Created from template athackst/dockerfiles/ros2/galactic-cuda.Dockerfile
##########################################################################

###########################################
# Base image 
###########################################
FROM nvidia/cuda:11.8.0-cudnn8-devel-ubuntu20.04 AS base

ENV DEBIAN_FRONTEND=noninteractive

# Install language
RUN apt-get update && apt-get install -y \
  locales \
  && locale-gen en_US.UTF-8 \
  && update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8 \
  && rm -rf /var/lib/apt/lists/*
ENV LANG=en_US.UTF-8

# Install timezone
RUN ln -fs /usr/share/zoneinfo/UTC /etc/localtime \
  && export DEBIAN_FRONTEND=noninteractive \
  && apt-get update \
  && apt-get install -y tzdata \
  && dpkg-reconfigure --frontend noninteractive tzdata \
  && rm -rf /var/lib/apt/lists/*

ARG USERNAME=ros
ARG USER_UID=1000
ARG USER_GID=$USER_UID

# Create a non-root user
RUN groupadd --gid $USER_GID $USERNAME \
  && useradd -s /bin/bash --uid $USER_UID --gid $USER_GID -m $USERNAME \
  # [Optional] Add sudo support for the non-root user
  && apt-get update \
  && apt-get install -y sudo \
  && echo $USERNAME ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/$USERNAME\
  && chmod 0440 /etc/sudoers.d/$USERNAME \
  # Cleanup
  && rm -rf /var/lib/apt/lists/* \
  && echo "source /usr/share/bash-completion/completions/git" >> /home/$USERNAME/.bashrc

# install cmake for building opencv
RUN apt-get update && apt-get install -y cmake git

USER $USERNAME

ARG OPENCV_DIR=/home/$USERNAME/opecv_build

# Install Opencv
RUN set -eux; \
  mkdir -p $OPENCV_DIR; \
  git clone --depth 1 --branch 4.5.4 https://github.com/opencv/opencv.git $OPENCV_DIR/opencv; \
  git clone --depth 1 --branch 4.5.4 https://github.com/opencv/opencv_contrib.git $OPENCV_DIR/opencv_contrib;

RUN set -eux; \
  mkdir -p "$OPENCV_DIR/opencv/build"; \
  cd "$OPENCV_DIR/opencv/build"; \
  cmake -D CMAKE_BUILD_TYPE=RELEASE \
    -D CMAKE_INSTALL_PREFIX=/usr/local \
    -D WITH_TBB=ON \
    -D ENABLE_FAST_MATH=1 \
    -D WITH_FFMPEG=ON \
    -D CUDA_FAST_MATH=1 \
    -D WITH_CUBLAS=1 \
    -D WITH_CUDA=ON \
    -D BUILD_opencv_cudacodec=OFF \
    -D WITH_CUDNN=ON \
    # -D CUDNN_LIBRARY=/usr/local/cuda/targets/aarch64-linux/lib/libcudnn.so \
    -D CUDNN_LIBRARY=/lib/x86_64-linux-gnu/libcudnn.so \
    # -D CUDNN_INCLUDE_DIR=/usr/local/cuda/targets/aarch64-linux/include \
    -D CUDNN_INCLUDE_DIR=/usr/include \
    -D OPENCV_DNN_CUDA=ON \
    -D BUILD_opencv_world=ON \
    -D CUDA_ARCH_BIN=7.2 \
    -D WITH_V4L=ON \
    -D WITH_QT=OFF \
    -D WITH_OPENGL=ON \
    -D WITH_GSTREAMER=ON \
    -D OPENCV_GENERATE_PKGCONFIG=ON \
    -D OPENCV_PC_FILE_NAME=opencv.pc \
    -D INSTALL_C_EXAMPLES=OFF \
    -D INSTALL_PYTHON_EXAMPLES=OFF \
    -D OPENCV_GENERATE_PKGCONFIG=ON \
    -D OPENCV_EXTRA_MODULES_PATH=$OPENCV_DIR/opencv_contrib/modules \
    -D BUILD_EXAMPLES=OFF \
    ..; \
  make -j$(nproc); \
  sudo make install; \
  sudo ldconfig

###########################################
#  Develop image 
###########################################
FROM base AS development

# Include the package of ros
RUN sudo apt-get update && sudo apt-get install -y \
  curl \
  gnupg2 \
  lsb-release \
  sudo \
  && sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg \
  && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null

ENV ROS_DISTRO=galactic

RUN sudo apt-get update && sudo apt-get install -y \
  bash-completion \
  build-essential \
  gdb \
  git \
  python3-colcon-common-extensions \
  python3-flake8 \
  python3-pip \
  python3-pytest-cov \
  python3-rosdep \
  python3-setuptools \
  python3-vcstool \
  wget

ENV PYTHONNOUSERSITE=1

RUN set -eux; \
  rm -rf /home/ros/.local/lib/python3.8/site-packages/setuptools* \
         /home/ros/.local/lib/python3.8/site-packages/importlib_metadata* \
         /home/ros/.local/lib/python3.8/site-packages/pkg_resources* || true

# Update pydocstyle
# RUN python3 -m pip install -U \
RUN set -eux; \
  sudo python3 -m pip install --no-cache-dir --no-user -U pip; \
  sudo python3 -m pip install --no-cache-dir --no-user -U \
  flake8-blind-except \
  flake8-builtins \
  flake8-class-newline \
  flake8-comprehensions \
  flake8-deprecated \
  flake8-docstrings \
  flake8-import-order \
  flake8-quotes \
  pytest-repeat \
  pytest-rerunfailures \
  pytest

# Install ROS2

ARG ROS2_DIR=/home/$USERNAME/ros2_galactic

RUN mkdir -p $ROS2_DIR/src; \
  cd $ROS2_DIR; \
  curl -fsSL https://raw.githubusercontent.com/ros2/ros2/galactic/ros2.repos -o /tmp/ros2.repos; \
  sed -i "s/version: 2.3.x/version: v2.3.6/" /tmp/ros2.repos; \
  vcs import --recursive "${ROS2_DIR}/src" --input /tmp/ros2.repos; \
  sudo apt update; \
  sudo apt upgrade -y; \
  sudo rosdep init || echo "rosdep already initialized"; \
  rosdep update --include-eol-distros; \
  rosdep install --from-paths src --ignore-src --rosdistro galactic -y --skip-keys "fastcdr rti-connext-dds-5.3.1 urdfdom_headers"; \
  sudo apt remove libopencv*; \
  touch $ROS2_DIR/src/ros-visualization/qt_gui_core/qt_gui_cpp/COLCON_IGNORE; \
  touch $ROS2_DIR/src/ros-visualization/rqt/rqt_gui_cpp/COLCON_IGNORE; \
  sed -i "/version='1.1.2'/a\    py_modules=[]," $ROS2_DIR/src/ros-visualization/rqt/rqt/setup.py

RUN sudo apt-get install -y \
  unzip \
  libgflags-dev \
  libflatbuffers-dev \
  flatbuffers-compiler \
  libboost-python1.71-dev \
  libboost-python-dev \
  ros-galactic-ament-cmake-python \
  ros-galactic-ament-cmake-ros \
  ros-galactic-rclcpp \
  ros-galactic-message-filters \
  ros-galactic-sensor-msgs \
  ros-galactic-pluginlib \
  ros-galactic-ament-lint-auto \
  ros-galactic-tf2 \
  ros-galactic-yaml-cpp-vendor \
  ros-galactic-ament-cmake-cppcheck \
  ros-galactic-eigen3-cmake-module \
  ros-galactic-ament-cmake-cpplint \
  ros-galactic-ament-cmake-lint-cmake \
  ros-galactic-ament-cmake-uncrustify \
  ros-galactic-ament-cmake-auto \
  ros-galactic-ros-environment \
  ros-galactic-image-transport

# RUN source /opt/ros/galactic/setup.sh

RUN cd $ROS2_DIR/src/ros-perception; \
  git clone https://github.com/ros-perception/vision_opencv.git; \
  cd vision_opencv; \
  git checkout galactic; \
  git pull origin galactic; \
  cd $ROS2_DIR; \
  colcon build --symlink-install

RUN echo "source ~/ros2_galactic/install/setup.bash" >> /home/$USERNAME/.bashrc \
  && echo "export PATH=/usr/local/cuda/bin:$PATH" >> /home/$USERNAME/.bashrc \
  && echo "export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH" >> /home/$USERNAME/.bashrc \
  && echo "RMW_IMPLEMENTATION=rmw_cyclonedds_cpp" >> /home/$USERNAME/.bashrc \
  && echo "export ROS_LOCALHOST_ONLY=1" >> /home/$USERNAME/.bashrc \
  && echo "if [ -f /opt/ros/$ROS_DISTRO/setup.bash ]; then source /opt/ros/$ROS_DISTRO/setup.bash; fi" >> /home/$USERNAME/.bashrc

COPY dockerfiles/entrypoint.sh /usr/local/bin/entrypoint.sh
RUN sudo chmod +x /usr/local/bin/entrypoint.sh

SHELL ["/bin/bash", "-c"]
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
CMD ["/bin/bash"]
