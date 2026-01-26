# Segway RMP Library

[![Ubuntu 22.04](https://img.shields.io/badge/Ubuntu-22.04%20LTS-orange)](https://releases.ubuntu.com/22.04/)
[![ROS 2 Humble](https://img.shields.io/badge/ROS%202-Humble%20Hawksbill-blue)](https://docs.ros.org/en/humble/index.html)
[![C++](https://img.shields.io/badge/Language-C%2B%2B-blue.svg)](https://isocpp.org/)
[![Based On](https://img.shields.io/badge/Based%20On-utexas--bwi%2Flibsegwayrmp__ros2-lightgrey)](https://github.com/utexas-bwi/libsegwayrmp_ros2)
[![Calibrated For](https://img.shields.io/badge/Calibrated%20For-RMP50%20%2F%20RMP100-9cf)](https://www.segway.com/)

This repository contains a modified and calibrated version of the `libsegwayrmp` low-level driver, specifically tuned for the **Segway RMP 50/100** mobile base used by the Vizzy robot. 
It builds upon the original `libsegwayrmp` library from UT Austin's BWI lab.
The primary goal of this fork is to correct significant inaccuracies in velocity control and odometry reporting that were present in the original library.

## The Problem

Testing on both ROS 1 and ROS 2 revealed that the default driver constants for the RMP 50/100 were incorrect. This resulted in two major issues:

1.  **Velocity Mismatch:** The robot's actual velocity was consistently **30-50% lower** than the commanded velocity.
2.  **Inaccurate Odometry:** The reported distance traveled was incorrect, forcing the use of a large `linear_odom_scale` factor (e.g., `1.6` to `2.0`) in the high-level ROS driver as a workaround.
3.  **Incorrect Yaw Rate:** The `yaw_rate` was being estimated from wheel velocities instead of using the more accurate onboard gyroscope data.

## The Solution
This version of the library implements two key fixes to address these issues at the source:

### 1. Yaw Rate Calculation Correction
The packet parsing logic has been corrected to use the direct gyroscope data provided by the Segway hardware.

* **Before:** `yaw_rate` was incorrectly calculated from wheel speeds: `(v_right - v_left) / 0.5`.
* **After:** `yaw_rate` is now correctly read from the sensor data packet: `getShortInt(packet.data[4], packet.data[5]) / this->dps_to_counts_`.

This change ensures that the angular velocity reported in odometry is accurate and stable.

### 2. Linear Motion Calibration 🔧
The core conversion constants within the library have been re-calibrated based on empirical tests. These changes ensure that both the sent motor commands and the received odometry data accurately reflect real-world motion.

* **`mps_to_counts_`**: Adjusted from `401.0` to `601.5` (1.5x). This corrects the mapping from meters/second to the internal velocity counts sent to the motors.
* **`meters_to_counts_`**: Adjusted from `40181.0` to `20090.5` (0.5x). This corrects the conversion from raw encoder ticks to meters for odometry.

## Calibration Results

The new values were determined through systematic testing. The tables below show the measured performance, with sent commands and reported odometry now closely matching real-world motion.

### Linear velocity calibration (`mps_to_counts_ = 601.5`)
| Command (m/s) | Distance Travelled (m) | Time Taken (s) | Real Velocity (m/s) | % loss |
|:---:|:---:|:---:|:---:|:---:|
| 0.200 | 1.80 | 9.06 | 0.200 | ~0 |
| 0.205 | 1.80 | 8.75 | 0.206 | ~0 |
| 0.350 | 1.80 | 4.91 | 0.367 | -4.86 |
| 0.500 | 1.80 | 3.61 | 0.499 | ~0 |

### Odometry distance calibration (`meters_to_counts_ = 20090.5`)
| Command (m/s) | Program Dist. (m) | Time Taken (s) | Odometry Data (m) | % loss |
|:---:|:---:|:---:|:---:|:---:|
| 0.200 | 1.80 | 9.06 | 1.85  | ~2.70 |
| 0.205 | 1.80 | 8.75 | 1.80  | ~0 |
| 0.350 | 1.80 | 4.91 | 1.866 | −3.67 |
| 0.500 | 1.80 | 3.61 | 1.896 | 5.33 |

These results confirm that the updated constants provide accurate control and feedback with minimal error, without requiring extra scaling factors elsewhere in the stack.

### 3. Angular Motion Calibration
The angular-motion conversion constants were also re-calibrated based on empirical tests. These updates improve both the commanded turning behavior and the reported turn integration from the Segway’s onboard integrators, reducing the need for any downstream “fudge factors”.

* **`dps_to_counts_`**: Adjusted from `7.8` to `3.84` for RMP50/100 (~0.5x). This constant converts commanded angular velocity (in **deg/s**) into the internal “counts” used by the Segway command packet. Bench tests show it produces low error across typical turn rates.

* **`rev_to_counts_`**: Adjusted from `117031.0` to `61900` (~1.9x). This constant converts the raw integrated turn accumulator into **revolutions**, which the driver then converts into **degrees**. This directly impacts the accuracy of `integrated_turn_position` (and therefore odometry yaw).
  - The chosen value is a compromise that gives good accuracy at moderate turn rates (e.g., ~0% error at 0.5 rad/s), with larger deviation at lower rates likely due to real-world effects like wheel slip, static friction, and controller deadbands.

Together with the yaw-rate parsing fix (using gyroscope-derived yaw-rate from the status packet), these changes produce more stable and physically meaningful angular velocity and yaw integration in the ROS odometry output.

## Calibration Results

The new values were determined through systematic testing. The tables below show the measured performance, with sent commands and reported odometry now closely matching real-world motion.

### Angular velocity calibration (`dps_to_counts_ = 3.84`)
| Command (rad/s) | Angle Rotated (deg) | Time Taken (s) | Expected Time (s) | Real Velocity (rad/s) | % loss |
|:---:|:---:|:---:|:---:|:---:|:---:|
| 0.2  | 360 | 32.6 | 31.4 | 0.19 | 5.00 |
| 0.35 | 360 | 18.3 | 18.0 | 0.34 | 2.86 |
| 0.5  | 360 | 12.1 | 12.6 | 0.52 | -4.00 |

### Turn integration calibration (`rev_to_counts_ = 61900`)
| Command (rad/s) | Angle Rotated (deg) | % loss |
|:---:|:---:|:---:|
| 0.2  | 360 | 16.45 |
| 0.35 | 360 | -9.61 |
| 0.5  | 360 | ~0 |

These results confirm that the updated constants provide accurate control and feedback with minimal error, without requiring extra scaling factors elsewhere in the stack.

## Usage

This library is intended as a drop-in replacement for the original `libsegwayrmp` within any ROS 2 workspace that uses the `segway_rmp_ros2` driver.

1.  **Remove Old Version (Important):**
    Before cloning, ensure any existing `libsegwayrmp` or `libsegwayrmp_ros2` directories are removed from your workspace's `src` folder to avoid build conflicts.
    ```bash
    # Navigate to your workspace source directory
    cd <path_to_your_ros2_workspace>/src
    
    # Remove any old versions that may exist
    rm -rf libsegwayrmp
    rm -rf libsegwayrmp_ros2
    ```

2.  **Clone the Calibrated Library:**
    Clone this repository into your `src` directory.
    ```bash
    cd <path_to_your_ros2_workspace>/src
    git clone https://github.com/JPLDevMaster/libsegwayrmp_ros2.git
    ```
    *(Note: The repository is named `libsegwayrmp_ros2`, but the ROS 2 package it contains is `libsegwayrmp`.)*

3.  **Build the Packages:**
    Build the library and any packages that depend on it, such as `segway_rmp_ros2`.
    ```bash
    # This command rebuilds the library and ensures any dependent nodes are re-linked
    colcon build --packages-select libsegwayrmp
    ```

4.  **Update High-Level Node Parameters:**
    With the calibration now handled at the low level, the odometry scaling factor in the high-level ROS 2 driver is no longer necessary. In your launch file for `segway_rmp_ros2`, ensure the odometry scale is set back to `1.0`.
    ```xml
    <param name="linear_odom_scale" value="1.0" />
    <param name="angular_odom_scale" value="1.0" />
    ```

This ensures your entire stack operates with accurate, calibrated data directly from the source, eliminating the need for software workarounds.
