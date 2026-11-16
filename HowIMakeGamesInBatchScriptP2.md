# How I make games in Batch Script, Part 2

## Introduction
This is the second part of my guide to making games in Batch Script ([part 1](https://github.com/thelowsunoverthemoon/BatchArticles/blob/main/HowIMakeGamesInBatchScriptP1.md)), and covers a variety of miscellaneous, smaller topics. The main focus is on the visual aspect: how to get colourful graphics in your games. However, note that those methods only apply to Windows 10+ (specifically Windows 10 version 1511+. Note that 1511 didn't support true 24 bit RGB colours but still accepted the sequences, just rounded them to the original 16 colour palette; RGB colours were added in 1703), as they rely on VT100 escape sequence support, but they would still work if using a terminal emulator that supports those sequences.

## Font

Changing the font can be useful when you have a custom font with sprites, for example. Font changing can be done by using the [REG](https://ss64.com/nt/reg.html) family of commands. These commands edit the registry, which contains configurations for the operating system, applications, and user preferences. Tangentially, these commands are quite useful for many of the annoying things in Windows. For example, disabling Cortana:

```Batch
@ECHO OFF
REG ADD "HKLM\SOFTWARE\Policies\Microsoft\Windows\Windows Search" /V "AllowCortana" /T REG_DWORD /D "0" /F
PAUSE
EXIT /B 
```

There are requirements for the font used. For example, in Conhost, there are a list of requirements such as being monospace (full list [here]( https://renenyffenegger.ch/notes/Windows/dirs/Windows/System32/conhost_exe)). The overall technique is to add a ```Console``` registry key with the console title (in the example, it's GameEngine), start the process again with a parameter (so it gets the new font and skips creating the reg key) then delete the reg key so it doesn't affect other scripts if they have the same title. The specification for the key can be found [here](https://renenyffenegger.ch/notes/Windows/registry/tree/HKEY_CURRENT_USER/console/index). 

```Batch
@ECHO OFF
IF "%1" == "" (
    >NUL REG ADD HKCU\Console\GameEngine /v FaceName /t reg_sz /d "Lucida Console" /f
    START "GameEngine" "%~0" 1
    EXIT
) else ( >NUL REG DELETE HKCU\Console\GameEngine /f )

ECHO Hello, I'm in a different font^!
PAUSE
EXIT
```

In the code above, I only change the FaceName (the font family) to Lucida Console, a preinstalled console font, but you can also change aspects such as the font size. For example, IcarusLives made it into a function with more parameters ([source](https://github.com/IcarusLivesHF/Batch-Script-Projects/blob/cb7e5baea3f7deecb9c397939d5aec1343f0de1f/atom/atom.bat#L4)):

```Batch
:Set_Font FontName FontSize max/nomax dummy
if "%4"=="" (
  for /f "tokens=1,2 delims=x" %%a in ("%~2") do if "%%b"=="" (set /a "FontSize=%~2*65536") else set /a "FontSize=%%a+%%b*65536"
  reg add "HKCU\Console\%~nx0" /v FontSize /t reg_dword /d !FontSize! /f
  reg add "HKCU\Console\%~nx0" /v FaceName /t reg_sz /d "%~1" /f
  set "m=" & if /I "%~3"=="max" set "m=/max"
  start "%~nx0" !m! "%ComSpec%" /c "%~f0" _
  exit /b 1
) else ( >nul reg delete "HKCU\Console\%~nx0" /f )
goto:eof
```

You can also combine this with the multithreading techniques I showed in part 1. For example, in [Pacman](https://github.com/thelowsunoverthemoon/Arcade.bat/blob/master/Scripts/Pacman.bat), I use a specialized version of this. Instead of returning ```1``` I return ```Font```, so the other processes (```GAME``` and ```CONTROLLER```) won't go through the font setting.

```Batch
IF "%1" == "" (
    >NUL REG ADD HKCU\Console\PacMan /v FaceName /t REG_SZ /d "Lucida Console" /f
    START "PacMan" "%~0" Font
    EXIT
) else IF "%1" == "Font" (
    >NUL REG DELETE HKCU\Console\PacMan /f
) else (
    GOTO :%1
)
```

## VT100 Escape Sequences
## Sixels
## Code Pages
## Music
## Framerate
## Hitboxes
