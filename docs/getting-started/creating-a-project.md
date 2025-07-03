# Creating a Project
## Cloning and Modifying the Hello World Project

This is the easiest and fastest way to get started with Mbed CE. You can simply clone the hello world project, rename it to your name of choice, and get coding.

For these steps, we're going to call the project MyProject -- replace that with your own project name in the commands below!

0. If you have not already done the toolchain setup guide, follow [those steps](toolchain-install.md) first.
1. Clone the hello world project. Open a terminal in the directory where you want to save the project and run
    ```shell
    $ git clone --recursive https://github.com/mbed-ce/mbed-ce-hello-world.git MyProject
    ```
2. Update the mbed-os submodule to the latest version:
    ```shell
    $ cd MyProject/mbed-os
    $ git fetch origin
    $ git reset --hard origin/master
    ```
3. Open up MyProject/CMakeLists.txt, and find the line that says
    ```cmake
    project(MbedCEHelloWorld
    ```
    Change `MbedCEHelloWorld` to `MyProject`.
4. Set up your project in an IDE ([VS Code](ide_cli_setup/vscode_setup.md), [CLion](ide_cli_setup/clion_setup.md)) or the [command line](ide_cli_setup/cli_setup.md).
5. To compile, build the `MyProject` target. As long as you've configured an upload method, you can also build the `flash-MyProject` target to upload code to your device.

## Creating a New Project from Scratch

If you would like more control over your project, you can also create it from scratch. Note that you can also watch these steps in a video version - [Youtube guide](https://youtu.be/dQHG_9O9X4c)
 
> This already assumes you have the development environment set up - [Toolchain Installation](toolchain-install.md).

1. Create a project folder, open a terminal in this directory, and generate an empty Git repo: `git init`. 
2. Optional: Add in a [basic gitignore file](https://github.com/mbed-ce/mbed-ce-hello-world/blob/master/.gitignore)
3. Add Mbed OS as a submodule: `git submodule add --depth 1 https://github.com/mbed-ce/mbed-os.git mbed-os`
4. Make sure that only the most recent commit of Mbed OS is cloned: `git config -f .gitmodules submodule.mbed-os.shallow true`. This step is optional for contributors and can be reverted, see below.
5. Create a file called `mbed_app.json5` (This file is used to configure various Mbed settings for your application.) in the project root directory with the following content:
   ```js
   {
      "target_overrides": {
         "*": {
            "platform.stdio-baud-rate": 115200,
            "platform.stdio-buffered-serial": 1,

            // Uncomment to use mbed-baremetal instead of mbed-os
            // "target.application-profile": "bare-metal"
         }
      }
   }
   ```

6. Create a file called `main.cpp` as empty program:
   ```cpp
   #include "mbed.h"

   int main()
   {
      while(true) {}

      // main() is expected to loop forever.
      // If main() actually returns the processor will halt
      return 0;
   }
   ```


7. Create a basic `CMakeLists.txt` with build rules for your program:
   ```cmake
   cmake_minimum_required(VERSION 3.19)
   cmake_policy(VERSION 3.19...3.22)

   set(MBED_APP_JSON_PATH mbed_app.json5)

   include(mbed-os/tools/cmake/mbed_toolchain_setup.cmake)
   project(MyMbedApp # here you can change your project name
       LANGUAGES C CXX ASM) 
   include(mbed_project_setup)

   add_subdirectory(mbed-os)

   add_executable(${PROJECT_NAME} main.cpp)
   target_link_libraries(${PROJECT_NAME} mbed-os) # Can also link to mbed-baremetal here
   mbed_set_post_build(${PROJECT_NAME})
   ```
   Click [here](https://github.com/mbed-ce/mbed-ce-hello-world/blob/master/CMakeLists.txt) for a more advanced example of a top level CMakeLists.txt!

9. Now, you are ready to set up the CMake project for editing. This can be done with an IDE ([VS Code](ide_cli_setup/vscode_setup.md), [CLion](ide_cli_setup/clion_setup.md)) or the [command line](ide_cli_setup/cli_setup.md).

## Note: Converting the Mbed OS Submodule
These instructions cause the Mbed OS submodule to be cloned in shallow mode, which reduces the amount of space consumed by its history.  However, this prevents you from making normal commits and contributions to Mbed OS.  If you want to make contributions to Mbed, you will need to convert this repo to a full Git repo. 

Note that these commands will wipe out any changes you have made to Mbed, so use `git stash` to save those first!

To do that, go into the mbed-os folder and run the following commands (from [here](https://stackoverflow.com/a/17937889/7083698)):
```
git fetch --unshallow
git config remote.origin.fetch "+refs/heads/*:refs/remotes/origin/*"
git fetch origin
git checkout master
git reset --hard origin/master
git config -f .gitmodules submodule.mbed-os.shallow false
```
That will get you on a master branch that matches master branch on the mbed-os repo, and you can make commits, PRs, and forks like normal.