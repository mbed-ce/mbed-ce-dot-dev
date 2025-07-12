# CLion Setup

This page will show you how to take an existing Mbed CE project and use it with the CLion IDE.

## Installing CLion

Note: CLion is a paid product for commercial use, costing about $100 yearly (there's a subscription, but if you cancel it you can keep using your current version forever).  However, it is free for students if you fill out the application [here](https://www.jetbrains.com/community/education/#students). There is also a free 30-day trial available for everyone. I (Jamie) can say that it's the best C++ IDE I've personally used, and I highly recommend giving it a shot.

1. Download the CLion installer from the [download page](https://www.jetbrains.com/clion/download/).
    a. You can also use the [JetBrains Toolbox](https://www.jetbrains.com/toolbox-app/) app to make installing and updating multiple JetBrains IDEs easier.
2. On Windows/Mac, run the installer and go through the install.  You can then run CLion from your OS menu.  On Linux, you'll get a tar file instead.  Extract the tar file to Downloads, and move the folder to /opt/.  Then, you will be able to run CLion by executing `/opt/clion-20xx.x.x/bin/clion.sh` in a terminal.
3. The first time you open CLion you will get a license screen.  You can use the "start trial" button to just use the 30 day trial, or if you have a license you should log in with your JetBrains account.
4.  You should now be at the IDE start screen!

![image](https://user-images.githubusercontent.com/2099358/187101127-f3cdba83-2932-4c56-8eba-8e96ec781c1b.png)

## Importing the Project
1. Now, in CLion, click on Open, then browse to the root folder of your project.  Then, click OK.

![image](https://user-images.githubusercontent.com/2099358/187101223-2e95b918-11b6-4275-9dc8-d4e063f83f45.png)

2. You'll likely get the below trust dialog box.  Click on Trust Project.

![image](https://user-images.githubusercontent.com/2099358/187101279-83d64ed6-0e80-4671-93bf-ac8b935d6f67.png)

3. If this is your first project, CLion will ask about setting up a toolchain.  You can accept the defaults here and click Next, because the Mbed build system overrides the compiler selection.

![image](https://user-images.githubusercontent.com/2099358/187101341-b8bb3788-61be-4182-b44e-979d81f223a4.png)

4. Now, you will get a wizard to create a build profile.  For now, change the "Name" and "Build type" settings to "Develop".  Also, in the "CMake Options" field, set MBED_TARGET to match your board name (e.g. `-DMBED_TARGET=NUCLEO_L452RE_P` for the Nucleo L452RE board). Also, optionally you can specify the [upload method](https://github.com/mbed-ce/mbed-os/wiki/Upload-Methods) to use.  Or, leave this unset to use the Mbed OS defaults for your board.
    - If you need Debug or Release configurations, you can come back here later and create them following the same process.

![Build Profiles Screen](https://user-images.githubusercontent.com/2099358/188508936-ed9958f1-ae2c-472d-b507-ef7bec556419.png)

Finally, hit Finish.

5.  If all goes well, you should see the CMake project successfully configure and load.  If not, you may need to double-check the [toolchain setup guide](https://github.com/mbed-ce/mbed-os/wiki/Toolchain-Setup-Guide) to make sure everything is set up OK.

NOTE: When working with Mbed projects, the Tools > CMake > Reset Cache and Reload Project option is your friend if things break.  If your project did not find the compilers successfully, things moved on the disk, or you're getting other errors, make it your first recourse to try this option.

![image](https://user-images.githubusercontent.com/2099358/187102104-81c52e9c-dbd8-435c-82fa-20eb68a25dc7.png)

## Flashing Code

To flash code, find the entry labeled `flash-<your target>` in the menu at the top right.  Then, click the hammer button (NOT the play button).

![image](https://user-images.githubusercontent.com/2099358/187102170-dfa3beb1-9d19-4333-8d67-a5e92c3a07d0.png)

Note: Mbed OS tends to clutter this menu up with its own targets.  To find your target quickly, you can just start typing with the menu open, and it will search the list.  Maybe this is obvious to you but it took me like 4 years to find this...

![image](https://user-images.githubusercontent.com/2099358/187102307-85664d9f-78ee-4c06-8503-9cec7c4cc232.png)

## Debugging
1. First, if you haven't already, you will need to make sure the project is configured to use an upload method that supports debugging. Read about upload methods on the [Upload Methods page](https://github.com/mbed-ce/mbed-os/wiki/Upload-Methods), and select a method to use using the -DUPLOAD_METHOD=<method> flag to CMake.  Note: you can edit the CMake flags that you initially configured by going to Settings > Build, Execution, Deployment > CMake.
2. Additionally, you will need to set the `MBED_CLION_PROFILE_NAME` CMake variable to match the exact name of the profile.  This is needed in order to generate debug configuration files correctly.
![image](https://user-images.githubusercontent.com/2099358/220048699-4a22c44a-ccac-49db-9e2e-c9a003c2405b.png)
3. In CLion, find the GDB debug configuration for your executable in the targets menu.  This will be near the bottom and start with "GDB", then your target name.  Make sure to select the one that matches your build type too -- if you are working in the Develop configuration, don't pick the Debug one!

![image](https://user-images.githubusercontent.com/2099358/220049391-a89dc7b4-188a-48fd-96fa-446c8263ad68.png)

4. With the configuration selected, click the green bug button at the top.  If source files have been modified, the code will be rebuilt and downloaded to the target.  Then, GDB will start.

Note: If you wish to have the code stop at main(), you must set a breakpoint there before starting the debug session.

Note: With CLion, the GDB command line console is available by going to the GDB tab in the Debug panel.  You can use this console in parallel with the GUI to evaluate expressions and set breakpoints.

![image](https://user-images.githubusercontent.com/2099358/187105850-0632954a-6ecb-44ae-960d-d848ef732cc8.png)

Note: To reset the MCU, press the R-with-a-bar-over-it button.

### Tips
- **Increasing GDB Timeout:** Sometimes, on Mbed, when you have deeply nested stack frames and lots of local variables, GDB can take a very long time to load after you hit a breakpoint, on the order of a minute.  This can cause CLion to time out and terminate the debugger session.  You can change the timeout by following [these instructions](https://www.jetbrains.com/help/clion/configuring-debugger-options.html#gdb-startup).  I'd recommend at least 120000 ms.