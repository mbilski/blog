---
title: Getting started with emacs for scala programming
date: 2016-05-26 11:34:35
---

This month I switched to emacs for scala programming. I was tired of the IntelliJ IDEA slowness and the fact that it consumes lots of CPU and memory. It works great for java, but with scala it's unpleasant. Then I discovered spacemacs and ensime. I felt a relief. I find it much more ergonomic and faster IDE for scala. However, the beginnings are hard. I wrote this post to help you with the firsts steps in emacs.

<!--more-->

## Spacemacs to the rescue

I have never used *emacs* before. After the first launch, I did not even know how to turn it off. I felt the same way when I used *vim* for the first time. It is a powerful tool but requires some time investment.

Emacs has an enormous amount of plugins and it allows you to configure everything the way you want. However, I am using [spacemacs] which is a package of pre-configured plugins.

Spacemacs by default is using *evil* mode which is *extensible vi layer for emacs*. For me, it was easier to switch to emacs with this mode turned on as I could use the commands that I already know.

For scala, I am using [ensime] which provides most of the features that heavy IDEs like IntelliJ IDEA or Eclipse have.

![emacs for scala programming](/img/emacs.png)

## Installation

To install emacs on OS X run

```
$ brew tap d12frosted/emacs-plus
$ brew install emacs-plus --with-cocoa --with-gnutls --with-librsvg \
  --with-imagemagick --with-spacemacs-icon
$ brew linkapps
```

OSX has old version of emacs already installed in `/usr/bin/emacs`. Add this line to your `~/.zshrc`

```
alias emacs="/usr/local/bin/emacs"
```

To fetch the spacemacs configuration execute
```
git clone https://github.com/syl20bnr/spacemacs ~/.emacs.d
```

Open new terminal window and run `emacs`. Spacemacs will ask if you want to use *vim* or *emacs* editor mode. Then, it will download and install all the packages. Press `control+x` and then `control+c` to exit.

To enable scala plugin edit the `~/.spacemacs` file and add `scala` to `dotspacemacs-configuration-layers`.

Now you need to add ensime plugin to sbt. Add this line to `~/.sbt/0.13/plugins/plugins.sbt`

```
addSbtPlugin("org.ensime" % "sbt-ensime" % "0.5.1")
```

For every scala project you need to generate `.ensime` file. Run these in a root project directory

```
sbt ensimeConfig
```

And that's all. Now you can start programming in scala using emacs!

## Hello world of emacs

Let's do some simple `hello-world` tutorial now. Go to the root project directory and run `emacs`. Press `SPC p f`. After each key you will see all possible commands and description at the bottom.

![](/img/emacs-help.png)

> In emacs commands are described like this `C-x C-f`. It means that first you need to press `ctrl+x` keys and then `ctrl+f`. `M-x` stands for `alt+x`. `SPC p p` means that you need to press `space`, then `p` and then `p` again. If you do a mistake, you can cancel operation using `C-g`.

Now you should be able to choose a file to open. Choose something and press `↩`. In the file editor, you can use commands like in vim. Press `i` to enter insert mode, type quickly `fd` to exit. Save the file using `C-x C-s`.

Press `M-x`, type `ensime`, and press `↩` to run ensime server and connect to it. Press `C-c C-c e` to show all errors and warnings. It will create a window. You can switch between windows using `M-number`. Press `M-1` to come back to opened file. To close current window press `C-x 0`.

When emacs opens a new file, shows type errors or launches sbt it creates a new buffer for it. You can switch to another buffer in a window using `C-x b`. Closing a window does not delete a buffer.

Press `C-c C-b t` to start sbt and run tests. Sbt is launched in a new buffer. You can switch to it and use it as a regular sbt shell.

This should be enough for an introduction. You opened and modified a file, type checked the project and run tests. Those are just a few basic commands, but I hope that it encourages you to try emacs.

## How to learn emacs

Emacs provides tutorials. Press `SPC h T` to launch the emacs *evil* tutor or press `C-h t` to launch the standard emacs tutorial. Note that for the latter one you will need to disable the *evil* mode.

You might also want to take a look at [ensime] (scala), [projectile] (project navigation and management) and [magit] (plugin for git).

I also recommend to print some cheat sheets:

- [how to learn emacs](http://sachachua.com/blog/wp-content/uploads/2013/05/How-to-Learn-Emacs-v2-Large.png)
- [emacs reference card](https://www.gnu.org/software/emacs/refcards/pdf/refcard.pdf)
- [vim cheat sheet](http://vim.rtorr.com/)
- [ensime cheat sheet](http://ensime.github.io/editors/emacs/cheat_sheet/)
- [projectile interactive commands](http://batsov.com/projectile/)
- [magit cheat sheet](http://daemianmack.com/magit-cheatsheet.html)

## Key bindings

Emacs allows you to customize key bindings. For that, edit `~/.emacs.d/init.el` file.

```
(global-set-key (kbd "<home>") 'back-to-indentation)
(global-set-key (kbd "<end>") 'move-end-of-line)
(global-set-key (kbd "<S-up>") 'move-text-up)
(global-set-key (kbd "<S-down>") 'move-text-down)
```

These commands overwrite the default behaviuor of home, end, page up and page down keys. With these definitions it works like *regular* editor.

```
(global-set-key [next]
  (lambda () (interactive)
    (condition-case nil (scroll-up)
      (end-of-buffer (goto-char (point-max))))))

(global-set-key [prior]
  (lambda () (interactive)
    (condition-case nil (scroll-down)
      (beginning-of-buffer (goto-char (point-min))))))

(defun pbcopy ()
  (interactive)
  (call-process-region (point) (mark) "pbcopy")
  (setq deactivate-mark t))

(defun pbpaste ()
  (interactive)
  (call-process-region (point) (if mark-active (mark) (point)) "pbpaste" t t))

(defun pbcut ()
  (interactive)
  (pbcopy)
  (delete-region (region-beginning) (region-end)))

(global-set-key (kbd "C-c c") 'pbcopy)
(global-set-key (kbd "C-c v") 'pbpaste)
(global-set-key (kbd "C-c x") 'pbcut)
```

This snippet integrates osx `pbcopy`, `pbpaste` and `pbcut` with emacs. To select a text use `C SPC`. Then, press `C-c c` to copy.

## Summary

I hope that this post might encourage you to try emacs and help you to do the first steps in it. I am sure that the time invested in learning emacs is worth it.

[spacemacs]: http://spacemacs.org/
[ensime]: http://ensime.github.io/
[projectile]: http://batsov.com/projectile/
[magit]: https://magit.vc/
