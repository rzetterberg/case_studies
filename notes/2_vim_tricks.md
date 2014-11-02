#Vim tricks

**Linux, Vim**

**Disclaimer:** These are some tricks I have collected over the years, in no way
are they the only way to do things. This is just the way I do things. If you 
have a better solution I would love to hear that, I am always looking to learn
something new.

##1. Productive shell scripts

I like to write scripts to automate common development tasks, such as running 
tests and compiling. I also want to be able to run these scripts without having
to switch to my shell and write the commands. To be able to do that I set up 
hotkeys for my most common tasks. For example I would have F5 to be bound to 
run my tests.

Now there are 3 different ways I run shell scripts:

- Hidden
- Visible
- Send to shell in a tmux pane

###Hidden 

Some tasks are very common, almost never fail and produce output I do not need
to see. Those tasks I run hidden and use a notification daemon to notify me 
when they are complete.

To run a shell command hidden you use the vim function 
`call system("yourcommand")`.

For example I would have a script that cleans the working directory of build
files called `clean_src.sh`, and I want to be able to run this command by 
pressing F5.

I first set up a command called `CleanSrc`:

```
:command CleanSrc :call system("./clean_src.sh")
```

I then bind F5 to run this `CleanSrc` command:

```
map <F5> :CleanSrc <CR>
```

When you run commands like this you will not know if the command failed. What
I like to do is to have the script use `notify-send` to send notifications to
me to tell me whether the command failed or succeeded. 

This is especially 
useful when the command would run for a couple of minutes. I can do something 
else while the command is running and then get a notification when it 
completes.

###Visible

Some tasks are very common, will often fail and produce output I do need
to see. Those tasks I run visible and use a notification daemon to notify me
when they are complete. 

To run a shell command visible you use the vim function `! yourcommand`.

For example I would have `make` compile my project and I want to be able to run
this command by pressing F4.

Like with hidden I set up a command, this time it is called `Compile`:

```
:command Compile :! make; false
```

The reason why I execute the command `false` after my command is because 
otherwise vim will execute the command and exit. By using `false` vim will
execute the command and then wait for me to press a button to bring me back
to editing. 

I then bind F4 to run this `Compile` command:

```
map <F4> :Compile <CR>
```

For commands like this I also like to use `notify-send` in my scripts to notify
me when the long running command is complete. This way I can do something else
will the command is running and also see the output after it is complete.

###Send to shell in a tmux pane

Some tasks are very common, will often fail, produce output I need to see and 
produce a state I need to be able to manipulate. 

For example I would have a REPL I want to send commands to that is opened in 
a tmux pane.

One way of doing this is using the plugin `vim-slime`. Using `vim-slime` you can
select code in visual mode and send it to a tmux pane. This very nice when you
have a piece of code you want to try out. 

One scenario I have encountered often is when you want to send the same piece 
of code many times. By having to select the code in visual everytime can be 
tedious. 

I looked at the source code of the `vim-slime` plugin to try to figure out if
you could send the contents of a file instead of the visual selection. It turns
out `vim-slime` do that each time by creating a temporary file. 

Tmux have two commands called `load-buffer` and `paste-buffer` to provide this
functionality. `load-buffer` takes a filepath, and `paste-buffer` sends that 
buffer to the pane specified by the `-t` flag. Here is what a script looks like 
that reads the content of the file `tmp/repl_buffer` and sends it to the second
pane in the first window of the session `vim_tricks`:

```
#!/bin/bash
tmux -L default load-buffer tmp/repl_buffer
tmux -L default paste-buffer -d -t vim_tricks:0.1
```

For example I am developing a python project. I have a tmux window split
vertically. On the left side I have vim and on the right side have python open.

I am currently working on a module called `parsing` that contains a function
called `extract_person` that I want to be able to run quickly with test data to 
see how it behaves. Here is what my `tmp/repl_buffer` contains:

```
import parsing

reload(parsing)

parsing.extract_person("Hello, {{person}}! How are you?")
```

I want to run this code in the REPL by pressing F1, so I add this to my rc:

```
:command SendRepl :call system("./send_repl.sh")
map <F1> :SendRepl <CR>
```

Since I am not interested in the output of `send_repl.sh` I run it hidden.
Now I can just press F1 to to run my function in the REPL. This becomes
very efficient when you want to test a very small part of your codebase
often. You could develop your unit test in your `tmp/repl_buffer` while
testing everything before putting the unit test in the appropriate module.

##2. Same shell and environment variables  

Often I found myself wanting to use the same shell and environment variables
when running shell commands in vim. A way I found to do that is to use a 
local bashrc file that I tell bash to use.

For example I am working on a python project which uses `virtualenv` to 
locally install all modules. I want my scripts that I execute in vim to 
also use `virtualenv`. 

I then create a file called `.bashrc.local` which contains:

```
#!/bin/bash

source env/bin/activate
```

By using the vim variable `shell` I can tell vim to use bash and source my
`.bashrc.local`:

```
set shell=/bin/bash\ --rcfile\ .bashrc.local\ -i
```

Now everytime I am using `call system("./mycommand")` or `! ./mycommand; false`
bash is using my `virtualenv`.

You can put other environment variables in there too, by using 
`export VAR_NAME="value"`.

##3. Project specific vimrc

Binding hotkeys and running scripts often is very specific to a project. 
Therefor I like to set up a project specific vimrc.

In vim you can read rc-files with the command `source`. You can put that
command in your global .vimrc:

```
source .vimrc.local
```

The problem is that when you open vim in a directory without a `.vimrc.local`
file it will give you an error. To avoid that only try to source it when the
file actually exists, like so:

```
if filereadable(glob(".vimrc.local")) 
    source .vimrc.local
endif
```

Now you can put a `.vimrc.local` with project specific hotkeys and whatnot!
