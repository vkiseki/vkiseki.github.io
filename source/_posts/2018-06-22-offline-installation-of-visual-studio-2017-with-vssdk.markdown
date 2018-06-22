---
layout: post
title: "Offline Installation of Visual Studio 2017 with VSSDK"
date: 2018-06-22 15:17:37 +0800
comments: false
categories: visual studio, vssdk, vsix
---

Download the installation files of Visual Studio 2017 on a computer connected to the Internet, then you can move those files somewhere and perform a installation.

<!--more-->

<br>

#### Steps
1. Download `Visual Studio bootstrapper` from [Microsoft](https://docs.microsoft.com/en-us/visualstudio/install/create-a-network-installation-of-visual-studio).

2. For `Visual Studio Enterprise`, run the command below. If you need English version, just replace `zh-CN` with `en-US`.<br>
   {% codeblock %}
D:\vs_Enterprise.exe --layout D:\VisualStudio2017 --add Microsoft.VisualStudio.Workload.ManagedDesktop -add Microsoft.VisualStudio.Workload.VisualStudioExtension --lang zh-CN
   {% endcodeblock %}

3. After downloading, all files are placed at `D:\VisualStudio2017`, just copy the folder to another computer.

4. Run `vs_setup.exe` in the installation folder.

5. How to verify:<br>
Open Visual Studio 2017, on the **File** Menu, click **New** then click **New Project**.<br>
Choose **Visual C#/Extensibility** in the left pane, and then create a **VSIX Project**.