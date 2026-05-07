# Segway RMP Library

[![Ubuntu 22.04](https://img.shields.io/badge/Ubuntu-22.04%20LTS-orange)](https://releases.ubuntu.com/22.04/)
[![ROS 2 Humble](https://img.shields.io/badge/ROS%202-Humble%20Hawksbill-blue)](https://docs.ros.org/en/humble/index.html)
[![C++](https://img.shields.io/badge/Language-C%2B%2B-blue.svg)](https://isocpp.org/)
[![Calibrated For](https://img.shields.io/badge/Calibrated%20For-RMP50%20%2F%20RMP100-9cf)](https://www.segway.com/)

This repository contains the `libsegwayrmp` low-level driver.

## Usage

This library is intended to use the `segway_rmp_ros2` driver.

1.  **Remove Old Version (Important):**
    Before cloning, ensure any existing `libsegwayrmp` or `libsegwayrmp_ros2` directories are removed from your workspace's `src` folder to avoid build conflicts.
    ```bash
    # Navigate to your workspace source directory
    cd <path_to_your_ros2_workspace>/src
    
    # Remove any old versions that may exist
    rm -rf libsegwayrmp
    rm -rf libsegwayrmp_ros2
    ```

2.  **Clone the Library:**
    Clone this repository into your `src` directory.
    ```bash
    cd <path_to_your_ros2_workspace>/src
    git clone https://github.com/utexas-bwi/libsegwayrmp_ros2.git
    ```
    *(Note: The repository is named `libsegwayrmp_ros2`, but the ROS 2 package it contains is `libsegwayrmp`.)*

3.  **Build the Packages:**
    Build the library and any packages that depend on it, such as `segway_rmp_ros2`.
    ```bash
    # This command rebuilds the library and ensures any dependent nodes are re-linked
    colcon build --packages-select libsegwayrmp
    ```

4.  **Update High-Level Node Parameters:**
    With the calibration handled at the low level, the odometry scaling factor in the high-level ROS 2 driver is no longer necessary. In your launch file for `segway_rmp_ros2`, ensure the odometry scale is set back to `1.0`.
    ```xml
    <param name="linear_odom_scale" value="1.0" />
    <param name="angular_odom_scale" value="1.0" />
    ```

This ensures your entire stack operates with accurate, calibrated data directly from the source, eliminating the need for software workarounds.
