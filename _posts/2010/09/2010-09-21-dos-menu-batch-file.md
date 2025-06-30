---
layout: post
title: "Script: DOS Menu Batch File"
summary: "Reference script for a menu in dos prompt"
author: samui
thumbnail: /images/2010/09/dosprompt.png
permalink: /blog/dos-menu-batch-file/
date: "2010-09-21"
category: [dos, script]
published: true
---

Earlier today, I created a batch file to handle a small delima I was having. I have been assigned a project to build an Application Installation Image (ISO). This ISO will need testing in three different environments (DEV = A, QB = B, PROD = C). The ISO is exactly the same except for the config files. The config files are geared for each individual environment (A, B, & C). In the past, the ISO had to be rebuilt each time for each different environment because the config files had to be included within the Application build. The rebuild process took several hours. Instead of rebuilding the ISO for each individual environment to test in, my co-workers and I decided to build one Golden Image, move all of the config files into a central location, and copy only the needed config files into the application folder (overwriting the ones that were installed from the Golden Image). Once the ISO has been given the green light to be deployed to the company, we would then install the final config files within the final Application ISO build.

For example: Golden Application Image (base\_env) is installed on WIN XP. Testing with Development is needed (Env A). Copy "Env A" config files to the Application folder on the XP machine. Then when testing is needed in other groups (ie., QA - Env B, and Prod - Env C), all that would be needed is to copy "Env B" and "Env C" config files when needed.

I've been copying the files over both via command-line and using 'Explorer'. I wanted to make this simpler, so I created a DOS based menu within a batch file. Since I haven't used batch scripting in a while, I wanted to share my code. I've modified this code so that my company's info is not shared publicly. I made reference to this [website](http://http-server.carleton.ca/~dmcfet/menu.html) during the writing of the code.

```
@echo off
REM +++++++++++++++++++++++++++++++++++++
REM +++ DOS Batch Menu v 0.2 +++
REM +++ written by J. Sam Aaron +++
REM +++ 09.21.2010 +++
REM +++++++++++++++++++++++++++++++++++++

REM - MENU LAYOUT
goto menu
:menu
echo *****************************
echo * *
echo * What file(s) do you need? *
echo * *
echo *****************************
echo * *
echo * Choice *
echo * *
echo * 1. +File A *
echo * 2. +File B *
echo * 3. +File C *
echo * 4. +EXIT *
echo * *
echo *****************************

REM - MENU VARIABLES
:choice
set /P C=[1,2,3,4]?
if "%C%"=="4" goto quit
if "%C%"=="3" goto filec
if "%C%"=="2" goto fileb
if "%C%"=="1" goto filea
goto choice

REM - MENU CHOICES
:intg
xcopy "CFG\*.*" "C:\Program Files\Application\CFG\"
cls
goto menu

:syst
xcopy "CFG\*.*" "C:\Program Files\Application\CFG\"
cls
goto menu

:prod
xcopy "CFG\*.*" "C:\Program Files\Application\CFG\"
cls
goto menu

:quit
exit

:end

```

