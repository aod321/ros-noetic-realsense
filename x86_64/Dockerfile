# Use ROS Noetic base image
FROM osrf/ros:noetic-desktop-full

# Switch to Tsinghua mirrors
RUN sed -i 's@http://.*archive.ubuntu.com@https://mirrors.tuna.tsinghua.edu.cn@g' /etc/apt/sources.list && \
    sed -i 's@http://.*security.ubuntu.com@https://mirrors.tuna.tsinghua.edu.cn@g' /etc/apt/sources.list

# Install required packages
RUN apt-get update && apt-get install -y \
    ros-noetic-realsense2-camera \
    ros-noetic-realsense2-description \
    ros-noetic-rgbd-launch \
    openssh-server \
    mesa-utils \
    net-tools \
    tmux \
    vim \
    git \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Setup environment
RUN echo "export DISPLAY=:0" >> /root/.bashrc && \
    echo "source /opt/ros/noetic/setup.bash" >> /root/.bashrc

# Setup workspace and install realsense-ros
RUN mkdir -p /root/catkin_ws/src && \
    cd /root/catkin_ws/src && \
    git clone https://github.com/IntelRealSense/realsense-ros.git && \
    cd realsense-ros && \
    git checkout ros1-legacy && \
    cd .. && \
    /bin/bash -c '. /opt/ros/noetic/setup.bash; catkin_init_workspace'

# Build workspace
RUN cd /root/catkin_ws && \
    /bin/bash -c '. /opt/ros/noetic/setup.bash; catkin_make clean && \
    catkin_make -DCATKIN_ENABLE_TESTING=False -DCMAKE_BUILD_TYPE=Release -j$(nproc) && \
    catkin_make install'

# Source the workspace
RUN echo "source /root/catkin_ws/devel/setup.bash" >> /root/.bashrc

# Create startup script
RUN echo '#!/bin/bash\n\
source /opt/ros/noetic/setup.bash\n\
source /root/catkin_ws/devel/setup.bash\n\
export ROS_MASTER_URI=${ROS_MASTER_URI:-http://localhost:11311}\n\
roslaunch realsense2_camera rs_rgbd.launch align_depth:=false depth_width:=1280 depth_height:=720 depth_fps:=60 color_width:=1920 color_height:=1080 color_fps:=60\n'\
> /root/start_camera.sh && \
chmod +x /root/start_camera.sh

# Set the working directory
WORKDIR /root/catkin_ws

# Command to run when container starts
CMD ["/root/start_camera.sh"]
