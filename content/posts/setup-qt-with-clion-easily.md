+++
title = "Setup Qt with Jetbrains Clion easily"
date = "2019-08-30"
author = "Aloïs Micard"
authorTwitter = "" #do not include @
cover = ""
tags = ["C++"]
keywords = ["", ""]
description = ""
showFullContent = false
+++

If you have ever worked with C++ for GUI development, chance are that you have heard of Qt. Qt is a free and open source widget toolkit for creating GUI and cross platform applications that run on many platforms such as Linux, Windows, MacOs, Android, etc... with native capabilities and performances. Qt does not provide only GUI API but has also support for networking, audio, serial port, thread, database, etc... It's one of the biggest framework ever written for C++.

Despite of the framework, Qt also provide a code editor named Qt Creator. Qt Creator is really powerful, it integrates GUI designer for the application, has a good debugger integration, ... But let's be honest: it's far from being the best code editor. Since I work heavily with Jetbrains products I use Clion, and I am really fast and efficient with it. I have then decided to make Qt works with Clion. Fortunately It can be done easily using CMake.

Please note that the following instructions assume that you have Qt & Clion installed and configured.

# Create the project and configure CMake

For this example we are going to create a new project from scratch

![Clion setup wizard](/img/clion-project-from-scratch.png)

You can now click on the CMakeLists.txt file to view CMake configuration

![Default Cmakelists.txt](/img/clion-cmake-config.png)

The first thing to do is to edit the CMakeLists.txt file to tell CMake where to find the Qt install directory. This is done by setting the Qt5_DIR variable:

```
cmake_minimum_required(VERSION 3.14)
project(QtTest)

set(CMAKE_CXX_STANDARD 17)

# Tell cmake where Qt is located
set(Qt5_DIR "~/Qt/5.12.2/clang_64/lib/cmake/Qt5")

add_executable(QtTest main.cpp)
```

Now we are going to tell CMake to find the Core and Widgets modules of Qt. This will allow us to build a minimal GUI application.

```
cmake_minimum_required(VERSION 3.14)
project(QtTest)

set(CMAKE_CXX_STANDARD 17)

# Tell cmake where Qt is located
set(Qt5_DIR "~/Qt/5.12.2/clang_64/lib/cmake/Qt5")

# Tell cmake to find the modules Qt5Core and Qt5widgets
find_package(Qt5 COMPONENTS Core Widgets REQUIRED)

add_executable(QtTest main.cpp)
```

The REQUIRED argument tells CMake to fails if the wanted package are not found.

We are not done yet! We still need to tell CMake to link the libraries to the executable and this is done in one instruction:

```
cmake_minimum_required(VERSION 3.14)
project(QtTest)

set(CMAKE_CXX_STANDARD 17)

# Tell cmake where Qt is located
set(Qt5_DIR "~/Qt/5.12.2/clang_64/lib/cmake/Qt5")

# Tell cmake to find the modules Qt5Core and Qt5widgets
find_package(Qt5 COMPONENTS Core Widgets REQUIRED)

add_executable(QtTest main.cpp)

# Link the library to the executable
target_link_libraries(QtTest Qt5::Core Qt5::Widgets)
```

Now that CMake is fully configured we can code a tiny application.

# Sample application

Let's create a basic Hello World application and see how it runs

```cpp
#include <QApplication>
#include <QPushButton>

int main(int argc, char** argv)
{
    QApplication app(argc, argv);

    QPushButton button("Hello world !");
    button.show();

    return app.exec();
}
```

![Sample app running](/img/clion-qt-example-app.png)

# Advanced

The previous instructions should help you building a tiny application, but if you need some features of Qt such as Graphical Interface design, auto MOC, resources files, ... You'll need extra configuration. Fortunately this is not very complicated.

## Enable the Meta Object Compiler (MOC)

The meta object compiler is one of the core functionality of Qt, it reads a C++ header file and if it finds a Q_OBJECT macro, it will produces a C++ source file containing meta object code for the class. It's the mechanism that allow signal and slots to work.
To enable the auto moc'ing you'll only need to add the following instructions to the CMakeLists.txt.

```
set(CMAKE_AUTOMOC ON)
```

## GUI design support

To enable GUI design support you'll need to perform two things:

### Enable user interface compiler (UIC)

The user interface compiler is a program that read XML from the .ui file generated by Qt Interface Designer and generate C++ code from it.

To enable the auto uic'ing you'll only need to add the following instructions to the CMakeLists.txt

```
set(CMAKE_AUTOUIC ON)
```

### Allow interaction between Clion and Qt Interface Designer

In "File -> Settings -> Tools -> External Tools", add 2 external tools:

```
Program:   "$PATH_TO_QT/QT_VERSION/$ARCHITECTURE/bin/qtcreator"
Arguments: $FilePath$
```

```
Program:   "$PATH_TO_QT/QT_VERSION/$ARCHITECTURE/bin/designer"
Arguments: $FilePath$
```