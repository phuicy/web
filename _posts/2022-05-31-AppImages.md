---
layout: post
title: AppImages and ROS2
author: Guy Burroughes
date: 31 May 2022
abstract: A brief tutorial on how to start using ROS2 applicaitons in AppImages.
---

# ROS2 in an AppImage

## Why?
{:.BodyText .Justified}
The Robot Operating System (ROS) is a set of software libraries and tools for building robot applications. 
From drivers and state-of-the-art algorithms to powerful developer tools, ROS has the open source tools you need for your next robotics project.
Its the good stuff if you are a robotics researcher or engineer.

{:.BodyText .Justified}
ROS has always been on the cutting edge, but this inevitably leads to injuries. In particular, version changes and wrestling between compatible workspaces
has been... let's say challenging.

{:.BodyText .Justified}
For a several years I have used docker to combat these issues succesfully. But the overheads of that are mind-boggling, >4GB for 4MB of application [\[1\]](https://hub.docker.com/r/ukaea/glovebox-simulator/tags), and you need docker runtime installed, and let's not even think about nvidia-docker v2.

{:.BodyText .Justified}
Beyond that, its not really suitable for distribution with users.

{:.BodyText .Justified}
AppImage to the rescue. 

{:.BodyText .Justified}
AppImage aims to be an application deployment system for Linux with the following objectives: simplicity, binary compatibility, distro agnosticism, no installation, no root permission, being portable, and keeping the underlying operating system untouched [\[2\]](https://en.wikipedia.org/wiki/AppImage).

{:.BodyText .Justified}
In other words, docker without the OS.

## Background
### AppImage
{:.BodyText .Justified}
AppImages are simple. AppImages are regular files, an AppImage contains exactly one application with **all** its dependencies.

{:.BodyText .Justified}
Each AppImage is self-contained: it includes all libraries the application depends on that are not already part of the targeted base-system. 
An AppImage consists of two parts: a runtime and a file system image. For the current type 2, the file system in use is SquashFS.

![AppImage file structure. Copyright © @TheAssassin 2019. Licensed under CC-By-SA Intl 4.0.](/public/images/architecture-overview.svg)

{:.BodyText .Justified}
What happens when an AppImage is run is that the operating system runs the AppImage as an executable. The runtime mounts the file system image.
Then the "AppDir" is available as a temporary read-only directory.[\[3\]](https://docs.appimage.org/reference/architecture.html)

{:.BodyText .Justified}
The runtime continues by calling the AppDir’s “entrypoint” AppRun using the operating system facilities. 
There are no checks performed by the runtime, the operating system is simply tasked with the execution of <AppDir mountpoint>/AppRun. 
This provides a lot of flexibility, as AppRun can be an arbitrary executable, a script with a shebang, or even a simple symlink to 
another executable within the AppDir. The only caveat is that the file must be executable.
  
{:.BodyText .Justified}
AppImage files are simpler than installing an application. No extraction tools are needed, 
nor is it necessary to modify the operating system or user environment.
Regular users on the common Linux distributions can download it, make it executable, and run it. 
  
Advantages:
  * Easier distribution
  * No Compatibility worries
  * Build once - run everywhere
  
{:.BodyText .Justified}
Drawbacks:
  * Hard for users for users to determine friend from foe, but not impossible, and who reads ROS code looking for hacks anyway. If you are running your own AppImages this problem obviously is removed.
  * Reinclusion of all system libraries, a degradtion of shared libraries; but nevermind, at least you aren't reincluding Ubuntu 20.04.
  
### LinuxDeploy
{:.BodyText .Justified}
Hand-rolling AppImages while possible would be pure massochism. Thus a variety of tools to generate AppImages have been created, the easiest to my experience: LinuxDeploy.
  
{:.BodyText .Justified} 
linuxdeploy is designed to be an AppDir maintenance tool. It provides extensive functionalities to create and bundle AppDirs for applications. It features a plugin system that allows for easy bundling of frameworks and creating output bundles such as AppImages with little effort [\[4\]](https://github.com/linuxdeploy/linuxdeploy).

### ROS2
  
{:.BodyText .Justified}
ROS2 builds on the beauty of ROS1. For our purposes the major change is the move from a bespoke communication system to a abstracted DDS system.
DDS (Data Distribution System) is a middleware standard that aims to enable dependable, high-performance, interoperable, real-time, scalable data exchanges using a publish–subscribe pattern. DDS addresses the needs of applications like aerospace and defense, air-traffic control, autonomous vehicles, medical devices, robotics, power generation, simulation and testing, smart grid management, transportation systems, and other applications that require real-time data exchange. 

{:.BodyText .Justified}
DDS is great although to be honest, its a bit too powerful. Its hard to configure. When I use I configure once with a lot of pain and then lock it in location.
ROS2 has added a layer of usability.


{:.BodyText .Justified}
Another feature of DDS, is that it is a standard, and their are multiple distributions (noting that they all have there own strengths and weakneeses).
The ROS2 DDS distro can be selected at run-time through environment variable and shared-libraries.


## How
* Step 1 - Download linux deploy and make an executable.
* Step 2 - Prepare your CMake
  This primarily means adding an install step for your intended target.
  ```cmake
  install(TARGETS bilateral unilateral bilateral_mock unilateral_mock taskspace taskspace_mock)
  ```
 * Step 3 - Add environmental variables to specify DDS distro.
  
{:.BodyText .Justified}
   Environmental variables can be set by the users, but this adds to the pain of distribution. ALternatively you can get your application to add the environmental variable if `APPDIR` is set as environmental variable. This enables your application to be run sans AppImage as well.
  
  For Cpp reference this in your main function.
  
  ```cpp
  /**
   * @brief Set the environment variable for LD_LIBRARY PATH for ROS2 in appimages
   *
   */
  void set_env_var() {
    std::stringstream s;
    auto ld_library_path_str = getenv("LD_LIBRARY_PATH");
    if (ld_library_path_str) {
      s << ld_library_path_str;
    }
    auto appdir_str = getenv("APPDIR");
    if (ld_library_path_str && appdir_str) {
      s << ":";
    }
    if (appdir_str) {
      s << appdir_str << "/usr/lib";
    }
    setenv("LD_LIBRARY_PATH", s.str().c_str(), true);
  }
  ```
  
  * Step 4 - Add .desktop
  This is a simple way to specify the application, icon and metadata.
  
  in a top level folder named desktop, then added a named dekstop file. For example, bilateral.desktop
  ```
  [Desktop Entry]
  Version=1.0
  Name=Bilateral Test
  Comment=Tests the loop cycle performance of the bilateral control system.
  Exec=bilaterala
  Path=${PWD}/build/AppDir/usr/bin/
  Icon=logo
  Terminal=false
  Type=Application
  Categories=Utility;Development;
  ```
  
  You can also add icons. By adding a PNG in this folder as well.
  
  
  
  * Step 5 - make install
  
  ```bash
  mkdir build
  cd build
  cmake .. -DCMAKE_INSTALL_PREFIX=/usr

  # build project and install files into AppDir

  make -j$(nproc)

  make install DESTDIR=AppDir
  ```

  * Step 6 - Package
  
  ```bash
  linuxdeploy-x86_64.AppImage --appdir AppDir -o appimage -d ../desktop/bilateral.desktop -i ../desktop/logo.png -l /opt/ros/foxy/lib/librmw_fastrtps_cpp.so -l /opt/ros/foxy/lib/librmw_dds_common__rosidl_typesupport_fastrtps_cpp.so -l /opt/ros/foxy/lib/librcl_interfaces__rosidl_typesupport_fastrtps_c.so -l /opt/ros/foxy/lib/librcl_interfaces__rosidl_typesupport_fastrtps_cpp.so -l /opt/ros/foxy/lib/libsensor_msgs__rosidl_typesupport_fastrtps_cpp.so 
  ```
  
  Explanation:

  -  ```--appdir AppDir``` - specify to use AppDir
  - ```  -o appimage``` - output an AppImage
  - ```-d ../desktop/bilateral.desktop ``` - specify desktop file location
  - ```-i ../desktop/logo.png ``` - specify icon file location
  - ```-l  ``` - spcify all the libraries that the linuxdeploy can't autmagically find in rpath because they are live loaded. Namely DDS distos and messages.

* Step 7 - Make executable and rename
  
  ```bash
  chmod +x unilateral-x86-64.appimage
  ```
## An Example

[https://github.com/phuicy/Kinova-Torque-Control](https://github.com/phuicy/Kinova-Torque-Control)
  
## Conclusion

{:.BodyText .Justified}
You now have 5MB of pure power.
This also works great for QT and RQT applicaitons.

{:.BodyText .Justified}
This solution is flawed and hacky, but elegance be damned. 
Perfection is the enemy of good and deploying anything.
Perfectionism is procrastination for the talented.
