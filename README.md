# Calibrated Segway RMP Library for Vizzy Robot

[![C++](https://img.shields.io/badge/Language-C%2B%2B-blue.svg)](https://isocpp.org/)
[![libsegwayrmp](https://img.shields.io/badge/Library-libsegwayrmp-brightgreen)](https://github.com/segwayrmp/libsegwayrmp)

This repository contains a modified and calibrated version of the `libsegwayrmp` low-level driver, specifically tuned for the **Segway RMP 50/100** mobile base used by the Vizzy robot. The primary goal of this fork is to correct significant inaccuracies in velocity control and odometry reporting that were present in the original library.

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

* **`mps_to_counts_`**: Adjusted from `401.0` to `601.5`. This corrects the mapping from meters/second to the internal velocity counts sent to the motors.
* **`meters_to_counts_`**: Adjusted from `40181.0` to `20090.5`. This corrects the conversion from raw encoder ticks to meters for odometry.

### Calibration Results
The new values were determined through systematic testing. The tables below show the performance improvements, with reported values now closely matching real-world measurements.

**Velocity Calibration (`mps_to_counts_ = 601.5`):**
| Command (m/s) | Distance (m) | Time (s) | Real Velocity (m/s) | % Error |
|:---:|:---:|:---:|:---:|:---:|
| 0.20 | 1.80 | 9.06 | 0.20 | ~0% |
| 0.21 | 1.80 | 8.75 | 0.21 | ~0% |

**Odometry Calibration (`meters_to_counts_ = 20090.5`):**
| Command (m/s) | Program Dist. (m) | Time (s) | Real Distance (m) | % Error |
|:---:|:---:|:---:|:---:|:---:|
| 0.20 | 1.80 | 9.06 | 1.85 | ~2.7% |
| 0.21 | 1.80 | 8.75 | 1.80 | ~0% |

These results confirm that the new constants provide accurate control and feedback with minimal error.

## Usage

This library is intended as a drop-in replacement for the original `libsegwayrmp` within the Vizzy ROS 2 workspace.

1.  **Build the Library:**
    Ensure this package is in your `src` folder and build your workspace.
    ```bash
    cd ~/vizzy2_ws
    colcon build --packages-select libsegwayrmp
    ```

2.  **Update High-Level Node Parameters:**
    With the calibration now handled at the low level, the odometry scaling factors in the high-level driver are no longer needed. In your launch file for `segway_rmp_ros2`, ensure the odometry scaling is set to `1.0`.
    ```xml
    <param name="linear_odom_scale" value="1.0" />
    ```

This will ensure your entire stack operates with accurate, calibrated data.
