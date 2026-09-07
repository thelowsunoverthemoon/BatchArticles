# Emulating Iterators in Batch Script

In C#, iterators are objects that traverse a block of code. Everytime you call the iterator method, it runs the function until it hits ```yield```. For example, stealing from the Microsoft documentation

```C#
public IEnumerable<int> GetSingleDigitNumbersLoop()
{
    int index = 0;
    while (index < 10)
        yield return index++;
}
```

would return ```0, 1, 2, 3, ...``` everytime you call the function. An important component of iterators is that it **can** use lazy evaluation. That pretty much means that execution is paused until the next time ```yield``` is called. Thus, in Batch, that means you have to "pause" the inside of a ```FOR``` loop. The fastest, but hardest way to do is is to parse Batch to convert all loops to non looping commands. However, this would be a pain due to some of the archaic rules of Batch. An alternative is start a new process and use some sort of waiting mechanism. We will also ned to communicate to the main batch file : one way to communicate is through redirection, and catching the signal through ```SET /P```.

That means we will have to precompile all the iterator functions to seperate batch files with a wait function. Something like this

```Batch
@ECHO OFF
(
    WAIT
    FUNCTION (replace YIELD with WAIT)
)<"wait%1.txt"
EXIT
```

Thus, we have

```Batch
@ECHO OFF
SETLOCAL ENABLEDELAYEDEXPANSION

CALL :ITER_INIT 3

ECHO Press a key to begin
PAUSE>NUL

(
    CALL :MAKE_ITER OTHER test2
    CALL :MAKE_ITER RANDOM test3
    CALL :MAKE_ITER RANGE test 200 300
    CALL :ITER test #a
    CALL :ITER test2 #b
    CALL :ITER test #c
    CALL :ITER test2 #d
    CALL :ITER test #e
    CALL :ITER test3 #f
    CALL :ITER test #g
    CALL :ITER test3 #h
    CALL :ITER test #i
    CALL :ITER test #j
    CALL :ITER test2 #k
)<"ret.txt"

SET #

PAUSE
EXIT /B

[iter] :RANGE
FOR /L %%Q in (%1, 1, %2) DO (
    YIELD RANGE %%Q
)
[iter] GOTO :EOF

[iter] :OTHER
FOR /L %%Q in (100, 50, 250) DO (
    YIELD OTHER %%Q
)
[iter] GOTO :EOF

[iter] :RANDOM
YIELD RANDOM BIRD
YIELD RANDOM GOOSE
YIELD RANDOM FLY
[iter] GOTO :EOF

:ITER <var> <ret>
SET "input="
ECHO a>>wait!%1!.txt

:WAIT_I
SET /P "input=" & IF defined input SET "%2=!input!" & GOTO :EOF
GOTO :WAIT_I

:MAKE_ITER <type> <ret> <args>
SET "input="
SET /A "%2=iter[num]", "iter[num]+=1"
START /B "" %1.bat !%2! %3 %4 %5 %6 %7 %8 %9

:WAIT_M
SET /P "input=" & IF defined input GOTO :EOF
GOTO :WAIT_M

:ITER_INIT <max>
DEL /F /Q wait*.txt ret.txt 2>NUL
(
    COPY NUL ret.txt
    FOR /L %%Q in (1, 1, %1) DO (
        COPY NUL wait%%Q.txt
    )
)>NUL

SET /A "iter[num]=1", "mode=1"
FOR /F "tokens=1,*" %%A in (%~nx0) DO (
    IF /I "%%A" == "[iter]" (
        IF !mode! EQU 1 (
            SET "name=%%B"
            SET "name=!name::=!"
            (
                ECHO @ECHO OFF
                ECHO GOTO :FUNC
                ECHO :WAIT
                ECHO SET /P "input=" ^& IF defined input SET "input=" ^& GOTO :EOF
                ECHO GOTO :WAIT
                ECHO :FUNC
                ECHO SET "file=%%1"
                ECHO SHIFT /1
                ECHO (
                ECHO     ECHO a ^>^>ret.txt ^& CALL :WAIT
            )>!name!.bat
        ) else (
            (
                ECHO ^)^<"wait%%file%%.txt"
                ECHO EXIT
            )>>!name!.bat
        )
        SET /A "mode*=-1"
    ) else IF !mode! EQU -1 (
        (IF /I "%%A" == "YIELD" (
            ECHO ECHO %%B ^>^>ret.txt ^& CALL :WAIT
        ) else (
            ECHO %%A %%B
        ))>>!name!.bat
    )
)
GOTO :EOF
```

We can redirect a "wait" file into each of the iterator functions, so they will only run once they receive a message from the main batch. The main batch will then wait until it receives the value from ```YIELD```. Since there is only one return at a time, we can redirect all of it to one ret.txt, where it is received by the main batch. Thus, we get

```
#a=RANGE 200
#b=OTHER 100
#c=RANGE 201
#d=OTHER 150
#e=RANGE 202
#f=RANDOM BIRD
#g=RANGE 203
#h=RANDOM GOOSE
#i=RANGE 204
#j=RANGE 205
#k=OTHER 200
```

Unfortunately there are some drawbacks to this method in particular. All the iterators have to be in one ret.txt or we will have to clear it if we break the redirection. As well, commands like ```PAUSE``` won't work since we are inside the redirection. However, these limitations can be worked around. There is also an alternative method (though very slightly slower) using a lock file instead of waiting for a signal. It is the exact same thing, but much more robust because we depend on the file itself, not its contents.

```Batch
:ITER <var> <ret>
ECHO a>>wait!%1!.txt

:WAIT_I
IF exist ret.0 SET /P %2=<ret.0 & MOVE ret.0 ret.1 >NUL & GOTO :EOF
GOTO :WAIT_I

:MAKE_ITER <type> <ret> <args>
SET /A "%2=iter[num]", "iter[num]+=1"
START /B "" %1.bat !%2! %3 %4 %5 %6 %7 %8 %9

:WAIT_M
IF exist ret.0 RENAME ret.0 ret.1 & GOTO :EOF
GOTO :WAIT_M

:ITER_INIT <max>
DEL /F /Q wait*.txt ret.* 2>NUL

ECHO.>ret.1
FOR /L %%Q in (1, 1, %1) DO (
    ECHO.>wait%%Q.txt
)

SET /A "iter[num]=1", "mode=1"
FOR /F "tokens=1,*" %%A in (%~nx0) DO (
    IF /I "%%A" == "[iter]" (
        IF !mode! EQU 1 (
            SET "name=%%B"
            SET "name=!name::=!"
            (
                ECHO @ECHO OFF
                ECHO GOTO :FUNC
                ECHO :WAIT
                ECHO SET /P "input=" ^& IF defined input SET "input=" ^& GOTO :EOF
                ECHO GOTO :WAIT
                ECHO :FUNC
                ECHO SET "file=%%1"
                ECHO SHIFT /1
                ECHO (
                ECHO     RENAME ret.1 ret.0 ^& CALL :WAIT
            )>!name!.bat
        ) else (
            (
                ECHO ^)^<"wait%%file%%.txt"
                ECHO EXIT
            )>>!name!.bat
        )
        SET /A "mode*=-1"
    ) else IF !mode! EQU -1 (
        (IF /I "%%A" == "YIELD" (
            ECHO ECHO %%B ^>ret.0 ^& CALL :WAIT
        ) else (
            ECHO %%A %%B
        ))>>!name!.bat
    )
)
GOTO :EOF
```

Because of this, we can actually ```PAUSE``` inside, and won't need to enclose it in a redirection. Overall though, there is still much to do such as nested iterators (which might not be possible using this setup), and killing the iterator when it reaches the last loop.
