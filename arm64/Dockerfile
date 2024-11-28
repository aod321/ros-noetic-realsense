# Use ROS Noetic base image
FROM ros:noetic-ros-base

# Avoid prompts from apt
ENV DEBIAN_FRONTEND=noninteractive

# Set environment variables
ENV LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH
ENV PYTHONPATH=$PYTHONPATH:/usr/local/lib
ENV PROTOCOL_BUFFERS_PYTHON_IMPLEMENTATION=cpp
ENV PROTOCOL_BUFFERS_PYTHON_IMPLEMENTATION_VERSION=3

# Switch to USTC mirrors and install required packages
RUN sed -i -e 's@//archive.ubuntu.com/@//mirrors.ustc.edu.cn/@g' \
           -e 's@//security.ubuntu.com/@//mirrors.ustc.edu.cn/@g' \
           -e 's@//ports.ubuntu.com/@//mirrors.ustc.edu.cn/@g' \
           /etc/apt/sources.list && \
    apt-get update && \
    apt-get dist-upgrade -y && \
    apt-get install -y \
        automake \
        libtool \
        vim \
        cmake \
        libusb-1.0-0-dev \
        libx11-dev \
        xorg-dev \
        libglu1-mesa-dev \
        python3-dev \
        python3-pip \
        git \
        wget \
        python-opengl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Install Python packages
RUN pip3 install pyopengl && \
    pip3 install pyopengl_accelerate==3.1.3rc1

# Prepare for protobuf build
WORKDIR /tmp
RUN git clone --depth=1 -b v3.10.0 https://github.com/google/protobuf.git

# Build protobuf - configure phase
WORKDIR /tmp/protobuf
RUN ./autogen.sh && \
    ./configure

# Build protobuf - make phase
RUN make -j8

# Install protobuf core
RUN make install && \
    ldconfig

# Build and install protobuf python bindings
WORKDIR /tmp/protobuf/python
RUN export LD_LIBRARY_PATH=../src/.libs && \
    python3 setup.py build --cpp_implementation

# Install protobuf python bindings
RUN export LD_LIBRARY_PATH=../src/.libs && \
    python3 setup.py install --cpp_implementation && \
    ldconfig

# Clean up protobuf files
RUN rm -rf /tmp/protobuf

# Install TBB for arm64
RUN apt-get update && \
    apt-get install -y libtbb-dev && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Install OpenCV and cv_bridge
RUN apt-get update && \
    apt-get install -y \
        libopencv-dev \
        python3-opencv \
        ros-noetic-cv-bridge && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Install ros-noetic-image-transport
RUN apt-get update && \
    apt-get install -y \
        ros-noetic-image-transport && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Build and install librealsense
# Clone librealsense
WORKDIR /tmp
RUN git clone https://github.com/IntelRealSense/librealsense.git

# Setup build directory and copy udev rules
WORKDIR /tmp/librealsense
RUN mkdir -p /etc/udev/rules.d && \
    cp config/99-realsense-libusb.rules /etc/udev/rules.d/ && \
    mkdir build

# Configure librealsense
WORKDIR /tmp/librealsense/build
RUN cmake .. -DBUILD_EXAMPLES=true -DCMAKE_BUILD_TYPE=Release -DFORCE_LIBUVC=true

# Build librealsense
RUN make -j8

# Install librealsense
RUN make install

# Configure Python bindings
RUN cmake .. -DBUILD_PYTHON_BINDINGS=bool:true -DPYTHON_EXECUTABLE=$(which python3)

# Build Python bindings
RUN make -j8

# Install Python bindings and update library cache
RUN make install && \
    ldconfig

# Clean up
RUN rm -rf /tmp/librealsense

# Handle udev rules
RUN udevadm control --reload-rules && udevadm trigger || true

# Create catkin workspace

# Source ROS in every RUN command
SHELL ["/bin/bash", "-c"]
WORKDIR /root
RUN mkdir -p ~/catkin_ws/src

# Clone realsense-ros
WORKDIR /root/catkin_ws/src
RUN git clone https://github.com/IntelRealSense/realsense-ros.git && \
    cd realsense-ros && \
    git checkout ros1-legacy

# Install dependencies
RUN apt-get update && \
    apt-get install -y \
    ros-noetic-ddynamic-reconfigure \
    ros-noetic-tf \
    ros-noetic-diagnostic-updater \
    ros-noetic-rgbd-launch \
    libeigen3-dev \
    dos2unix && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Initialize catkin workspace
WORKDIR /root/catkin_ws/src
RUN source /opt/ros/noetic/setup.bash && \
    catkin_init_workspace

# Clean build
WORKDIR /root/catkin_ws
RUN source /opt/ros/noetic/setup.bash && \
    catkin_make clean

# Build workspace
RUN source /opt/ros/noetic/setup.bash && \
    catkin_make -DCATKIN_ENABLE_TESTING=False -DCMAKE_BUILD_TYPE=Release

# Install
RUN source /opt/ros/noetic/setup.bash && \
    catkin_make install

# Setup environment
RUN echo "source ~/catkin_ws/devel/setup.bash" >> ~/.bashrc

# Source ROS setup in bashrc
RUN echo "source /opt/ros/noetic/setup.bash" >> /root/.bashrc

# Create startup script with proper line endings
RUN echo '#!/bin/bash' > /root/start_camera.sh && \
    echo 'source /opt/ros/noetic/setup.bash' >> /root/start_camera.sh && \
    echo 'source /root/catkin_ws/devel/setup.bash' >> /root/start_camera.sh && \
    echo 'export ROS_MASTER_URI=${ROS_MASTER_URI}' >> /root/start_camera.sh && \
    echo 'trap "exit 1" TERM INT' >> /root/start_camera.sh && \
    echo 'roslaunch --required realsense2_camera rs_rgbd.launch align_depth:=false depth_width:=1280 depth_height:=720 depth_fps:=60 color_width:=1920 color_height:=1080 color_fps:=60 & ROSLAUNCH_PID=$!' >> /root/start_camera.sh && \
    echo 'wait $ROSLAUNCH_PID' >> /root/start_camera.sh && \
    echo 'exit 1' >> /root/start_camera.sh && \
    chmod +x /root/start_camera.sh && \
    dos2unix /root/start_camera.sh

# Set working directory
WORKDIR /ros_ws

# Install and configure SSH server
RUN apt-get update && \
    apt-get install -y openssh-server dos2unix && \
    mkdir /var/run/sshd && \
    echo 'root:root' | chpasswd && \
    sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# SSH login fix
RUN sed 's@session\s*required\s*pam_loginuid.so@session optional pam_loginuid.so@g' -i /etc/pam.d/sshd

# Expose SSH port
EXPOSE 22

# Command to run when container starts
ENTRYPOINT ["/ros_entrypoint.sh"]
CMD ["/root/start_camera.sh"]