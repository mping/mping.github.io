---
layout: post
title: "Clojure, Emacs and Cider"
date: 2015-11-17T21:10:31+00:00
comments: true
---


# Intro #

I've been coming to clojure lately, and I finally decided to give emacs a go.
Unfortunately, setting up emacs for clojure is a PITA, for lots of reasons.

Let's try to shed some light on that.

## Terminology

- **Clojure**: a programming language from the lisp family. Plenty about that on the internet
- **Leiningen**: a clojure runner, not unlike npm or maven or whatever. Actually its different because the config file is written in clojure itself. It's basically a command-line utility to interact with clojure programs, which include launching a network repl, or **nREPL**, for interacting with your clojure code
- **Emacs**: an editor written in some kind of lisp. This is important, because you customize emacs by writing lisp (which is a different from clojure)
  - major modes: determines the editing behaviour of the current buffer. Only 1 active at any time
  - minor modes: alters behaviour in well-known way. Many active at same time
  - `init.el`: where the customization of emacs starts. Should be in `~/.emacs.d/init.el`
  - **Cider**: a set of [emacs modes](https://www.gnu.org/software/emacs/manual/html_node/emacs/Major-Modes.html) for working with clojure
    Cider makes use of a lib called `org.clojure/tools.nrepl` 


<br>

## Setting up

There are already alot of tutorials on clojure and leiningen. We're gonna dive into emacs and cider.
For emacs, I followed [this guide](http://www.braveclojure.com/basic-emacs/) (actually the braveclojure site is excellent for clojure also), but you will need to tweak some things to make sure everything runs smoothly.

**Note 1:** the guide doesn't have the link to the git repo that you should clone, right now its still online at https://github.com/flyingmachine/emacs-for-clojure so you should `git clone https://github.com/flyingmachine/emacs-for-clojure ~/.emacs.d`. I'm guessing the authors are prepping for the book release so they changed the text to reflect that.

**Note 2:** When you clone that repo, you are actually getting some packages besides the configuration. If you `ls ~/.emacs.d/elpa` you will see alot of folders.

**Note 3:** ~~Don't forget to edit your  `~/.lein/profiles.clj` and include cider-nrepl:~~

    {:user {:plugins [[cider/cider-nrepl "0.10.0-SNAPSHOT"]}} ;; ignore, see Note 4

**Note 4:** CIDER used to require you to keep its dependencies in your lein profiles. But CIDER and cider-nrepl have the same version numbers so its easy to keep them in sync. You are putting a hardcoded 0.10-snapshot but CIDER is 0.14.0 on melpa stable and 0.15 on melpa.
But CIDER now injects its dependencies so stating them is no longer required, but only serves to explicitly mismatch the frontend (CIDER) from the backend (cider-nrepl)

<br>

### What it is exactly that I'm doing here?

You're:

- using **emacs** as an editor
- using **leiningen** with a plugin called `cider-nrepl`
- using an emacs package called **cider**  to send clojure code to the repl and get results, *all from inside emacs*.


I honestly don't know exactly how it works  because I'm a clojure noob. This is how I *think* it works.

- when you launch a repl (`lein repl` for instance), and you have the `cider-nrepl` plugin the plugin hooks onto the repl
- when you call `cider-jack-in` you are *launching a repl* behind the scenes
- cider is "just" for editing files (in emacs lingo, buffers)
- cider-nrepl takes care of communicating with the repl, ie, sending clojure code from emacs to the repl and getting results, exceptions, etc

<br>

> Careful.
> Emacs uses cider, which is a major-mode. Cider communicates with the nrepl, so cider and nrepl must be version-compatible.
> You have to be careful with the cider and nrepl version you are using, otherwise you will have warnings and the repl may not work.
> This means you have to pick cider from the proper repo; if you clone the emacs settings mentioned above, you are already getting a specific cider version

<br>

### Melpa, Elpa, Marmalade

You can grab packages (plugins) for emacs from different repos. There's Melpa, Elpa and Marmalade. [Here's](http://emacs.stackexchange.com/questions/268/what-are-the-practical-differences-between-the-various-emacs-package-repositorie) a link explaining the differences. Make sure you grab cider from the proper repo, and that the cider (actually cider-nrepl) version matches with the cider-nrepl you configure in your leiningen.

It took me a while to figure out that different repos have different versions (ex: stable vs snapshot).
What I actually did was remove all package folders from `~/.emacs.d/elpa` and add all repos in `~/.emacs.d/init.el`.
Emacs will install packages on the first run


    ;; Define package repositories
    (require 'package)
    (add-to-list 'package-archives
                 '("elpa" . "https://elpa.gnu.org/"))
    (add-to-list 'package-archives
                 '("marmalade" . "http://marmalade-repo.org/packages/") t)
    (add-to-list 'package-archives
                 '("tromey" . "http://tromey.com/elpa/") t)
    (add-to-list 'package-archives
                 '("melpa" . "http://melpa.milkbox.net/packages/") t)
    (add-to-list 'package-archives
                 '("melpa-stable" . "http://stable.melpa.org/packages/") t)


And below:


    ;; The packages you want installed. You can also install these
    ;; manually with M-x package-install
    ;; Add in your own as you wish:
    (defvar my-packages
     '(;; makes handling lisp expressions much, much easier
       ;; Cheatsheet: http://www.emacswiki.org/emacs/PareditCheatsheet
       paredit

       ;; key bindings and code colorization for Clojure
       ;; https://github.com/clojure-emacs/clojure-mode
       clojure-mode

       ;; extra syntax highlighting for clojure
       clojure-mode-extra-font-locking

       ;; integration with a Clojure REPL
       ;; https://github.com/clojure-emacs/cider
       cider



<br>

### Emacs, Cider, Repl

> REPL: 
> When you are using cider, you are actually sending code to the REPL for evaluation, and getting results back.
> If you change a file, you have to either reload the classpath or send the code again

Let's get this show on the road. 

Start a new project:

    $ lein new testdrive
    $ ls testdrive

You should see a bunch of files, which include `project.clj` as well as `src` and `test` folders.
There should be a `core.clj` under `testdrive/core/` that looks like this:

    (ns testdrive.core)
    (defn foo
      "I don't do a whole lot."
      [x]
      (println x "Hello, World!"))


Lets go to emacs:

    $ emacs .

Navigate to the `core.clj` file with arrows and enter key. Now we want to fire a repl through cider.
Press `Alt+X` (in emacs lingo, its `M-x`), then write `cider-jack-in` and then `ENTER`.

The repl will start. Now you try to call the `foo function` from within the repl:

    user> (in-ns 'testdrive.core)
    #object[clojure.lang.Namespace 0x79a385c4 "testdrive.core"]
    testdrive.core> (foo 1)
    CompilerException java.lang.RuntimeException: Unable to resolve symbol: foo in this context, compiling:(/tmp/form-init5405367346234268941.clj:1:1) 

WHAAAAAT? I hate you emacs/clojure/repl.

>Note: please see the shortcuts at the end on how to navigate emacs, split the frame, etc.
>Its much easier to split frame and switch between the source and the repl.

**Not so fast**. When you fire the repl, it just spins a clojure repl, that's all. You haven't loaded anything yet (I mean, eval'd).
So now you want to load the `core.clj` onto the repl. There's a bunch of cider shortcuts for this, I do `Ctrl-c Ctrl-k`  (again, in emacs lingo its `C-c C-k`) (the Ctrl key is not released). The [cider readme](https://github.com/clojure-emacs/cider) states that this means "load (eval) the current buffer".

Now if you go to the repl again and try

    testdrive.core> (foo 1)
    1 Hello, World!
    nil

YEEAAAH.

If you don't want to go to the repl, you can write this in  `core.clj`

    (foo 1)

And press `C-c C-e`. You should see the result on the same line:

    (foo 1) => nil


Now change the `foo` function to "return" something (as you know, in clojure the last result is the return value).

    (defn foo
      "I don't do a whole lot."
      [x]
      (println x "Hello, World!")
      333)

Try again with `C-c C-e`:

    (foo 1) => nil

WHAAAAAAT? FUUUUUUU!

**Remember**: the repl has the old function definition. If you change the function, you have to eval it again.
Position the cursor at the end of the new definition for `foo` and eval it through `C-c C-f` (or any other way fwiw).
Now try to call foo again through `C-c C-e` (so that it shows the result in our buffer):

    (foo 1) => 333

> Note: when using cider, the position of the cursor is important to determine
> what is going to be evaluated

Basically, what this allows you is to load and run all the code that you want without ever leaving the repl (emacs in this case).
Cider has alot of goodies for interactive development:

- `C-c ,` for running tests for a given namespace (eg: if you're in the `testdrive.core` buffer/namespace, it will run `testdrive.core-test` which correspondes to the file `testdrive/core_test.clj`)
- We already seen `C-c C-e` for eval'ing and displaying the results (kinda like lighttable)
- `C-c C-x` for reloading the classpath (eg: you changed some code, and want to run tests)
- and others

> Caveat: if you add a dependency on project.clj, you have to restart cider 
> because the classpath is defined only once at startup (its a java thing).

<br>

### Shortcuts

I recommend the following pages:

- [https://coderwall.com/p/ufflua/clojure-development-with-emacs-live-key-bindings](https://coderwall.com/p/ufflua/clojure-development-with-emacs-live-key-bindings)
- [https://github.com/clojure-emacs/cider](https://github.com/clojure-emacs/cider)

Please kindly correct me if you find any bugs. You can raise an issue: [https://github.com/mping/mping.github.io/issues](https://github.com/mping/mping.github.io/issues)
And finally, stay tuned. I'll try and post some more clojure interactive development goodies.
