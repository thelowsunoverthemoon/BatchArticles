## Intro to Delayed Expansion


One of the first bumps in learning Batch is Delayed Expansion. If you’re coming from higher level languages, and you’re using an ```IF``` statement, this code should make sense. 

```Batch
SET "var=Hello"
IF "%var%" == "Hello" (
    SET "var=Bye"
    ECHO %var%
)
```

Output should be ```Bye```, right? Nope, it would output ```Hello```. The reason is how the CMD interpreter parses Batch script. This is the guy that reads your batch file line-by-line as it goes along. Typically, each line, it will expand all variables before stopping. However, when it reaches a parenthesis pair, it will automatically expand all the variables inside the pair. In other words, when it reaches a codeblock. So, for example

```Batch
FOR %%Q in (hello bye seeya) DO (
    REM Command
)
```

The FOR loop syntax takes in commands, and the CMD interpreter will expand all the variables inside that FOR loop first, before actually running it. So let’s say our old example

```Batch
SET "var=Hello"
IF "%var%" == "Hello" (
    SET "var=Bye"
    ECHO %var%
)
```

Once the interpreter gets to the IF, it will expand all the variables inside it, in this case ```var```. This expanded version is saved and then run. So, this version would look like

```Batch
IF “%var%" == “Hello” (
    SET “var=Bye”
    ECHO Hello
)
```

Notice how it is no longer a variable, but plain text. In effect, it’s exactly the same as a macro in a preprocessor

```C
#define PI 3.14

int
main(void) {
    printf("%f", PI);
}
```

Will become

```C
int
main(void) {
    printf("%f", 3.14);
}
```

Before compiling.

Now, all this is great, but obviously very limiting. For instance you can’t do this

```Batch
SET "num=1"
FOR %%Q in (hello bye seeya) DO (
    ECHO %num%
    SET /A "num+=1"
)
```

Which is such a simple construct. Keep in mind that only the ```ECHO``` doesn’t work. ```SET /A``` works fine, because there is no expansion involved. Now this will only output ```1```. However, we can enable Delayed Expansion. 

```Batch
SETLOCAL ENABLEDELAYEDEXPANSION
```

```SETLOCAL``` is a command that creates a "bubble", and paired with ```ENDLOCAL``` it’s like simulating variable scope. The parameters can change CMD behaviour, for example by allowing for command extensions. But to be honest, in this day and age, you won't see that option much. ```ENABLEDELAYEDEXPANSION``` is the one that allows variables to be expanded EVERY line. All you have to do is enclose it with ```!!``` instead of ```%%```.

```Batch
SET "num=1"
FOR %%Q in (1, 1, 3) DO (
    ECHO !num!
    SET /A "num+=1"
)
```

Now the script works as expected. Yay! Now you’ll notice that now we have 2 types of expansion : ```!``` and ```%```. That means we can also do this

```Batch
SET "inside=hello"
SET "ihello=Bye"

ECHO !i%inside%!
```

Will print ```Bye```, the contents of ```hello```. Like we saw before, dont think of Batch variables as objects. Think of them as text the moment they are expanded. That’s all.

```Batch
!i%inside%! -> first expansion -> !ihello! -> second expansion -> Bye
```

A key point is that ```%``` is expanded first, before ```!``` variables. This is important to keep in mind, because it can cause some unexpected errors sometimes. For example, you can’t do this

```Batch
%i!inside!%
```

Because ```%%``` are expanded first and ```i!inside!``` Is not a variable name. Now, is this the only way for double expansion? No, it’s not. You can also use CALL as a prefix. For example, to ECHO a variable

```Batch
CALL ECHO %%i%inside%%%
```

Notice how the outside```%``` are doubled. This escapes the ```%``` so it’s not expanded in the first expansion, but the inside pair is. Strangely, this method also relies on ```%``` as a second layer of expansion. So how does it work in loops? It’s because, again, ```CALL```ing a command is like ```CALL```ing a function. So it’s like

```Batch
CALL :FUNC

:FUNC
ECHO %ihello%
GOTO :EOF
```

Obviously, introducing Delayed Expansion also introduces many side effects. Read the link below on how CMD parses commands to gain a better understanding of some these effects. 

Be creative! These are basic usages but Delayed Expansion is a powerful tool that allows you to be very creative. That’s the beauty of Batch. The simplistic parsing (again, everything is text) allows a lot of interesting constructs, which you can see if you visit DOStips (a Batch forum) and look around.

### Further Links
* [More Detailed Information](https://ss64.com/nt/delayedexpansion.html)
* [How CMD parses commands](https://stackoverflow.com/questions/4094699/how-does-the-windows-command-interpreter-cmd-exe-parse-scripts)
* [DOStips](https://www.dostips.com/)
