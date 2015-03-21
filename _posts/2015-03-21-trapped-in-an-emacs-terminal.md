---
layout: post
title: Help! I'm Trapped in an Emacs Terminal!
date: 2015-03-21 12:00
categories: emacs
---
Exactly one week ago, I was minding my own business when suddenly I found myself trapped in an Emacs terminal with no way out. That's right, I made the mistake of trying to learn how to use this infernal piece of software, and now I can't seem to stop. It's definitely had its ups and downs so far. It looked a little something like this:

**Day One**: Well Steve, [you've done it](https://sites.google.com/site/steveyegge2/effective-emacs). You convinced me to try Emacs. And to [blog](https://sites.google.com/site/steveyegge2/you-should-write-blogs). And a lot of other things. Do you have any opinions on diet and exercise, too?

After opening Emacs, start with the tutorial: `C-h t`. Ooh, this doesn't seem too bad. I'm `C-v` and `M-v`-ing around the screen in no time. Delete that <del>wrod</del> word? `M-<backspace>`. This pretty easy.

**Day Two**: How to I get `<tab>` to insert spaces instead of tab?

**Day Three**: What the heck is with all of these `#auto-save#` and `backup~` files? Can I get rid of those please?

**Day Four**: HOW DO I GET THIS THING TO JUST INSERT FOUR GODDAMN SPACES WHEN I HIT THE `<tab>` KEY?

**Day Five**: (Googling "is learning Emacs worth it?"). (See [Matz' presentation](http://www.slideshare.net/yukihiro_matz/how-emacs-changed-my-life), decide to press on).

**Day Six**: I think I'm starting to get the hang of this.

So it's been a sometimes-frustrating but also rewarding experience. I'm feeling pretty comfortable now now with most of the basic `C-` commands, and it turns out that a lot of the shortcuts actually translate to other programs. For example, I've found that you can `C-f`, `C-b`, `C-n`, and `C-p` in most programs. Also, `M-<backspace>` and `M-d` are pretty awesome. I'm not sure how I lived without them for so long.

## Week One Survival Guide
Some of the things above took me a bit longer to figure out than they should have. I'm going to save some tips here so that future generations won't have to deal with the same frustrations that I did:

### I'm stuck in a command and I can't get out
This happened to me all the time in the beginning. I would accidentally hit `<ctrl>` instead of `<shift>` and then I would be afraid to touch anything for fear of deleting my document and not knowing how to get it back. There are three commands that you should commit to memory before doing anything else:

1. `C-g` - `keyboard-quit`: Stop whatever command that you're doing right now. Most of the time, this will quit the command completely, and the rest of the time it will just do nothing harmful.
2. `ESC ESC ESC`: This is apparently the universal "STOP DOING WHAT YOU'RE DOING" signal to Emacs. I usually use `C-g` because it's easier to type, but if you're in the middle of an `M-x` command, `ESC ESC ESC` might be a better bet.
3. `C-/` - `undo`: If you manage to insert something or run a command that you don't want to, you can just use the `undo` command. For some stupid reason the tutorial tells you to use `C-x u` for undo even though `C-/` is superior in every way. Save yourself 33% of your time and do `C-/` instead.

### What the heck is with all of #these# files~?
Emacs doesn't want you to lose any of the work that you do, so it does a lot of saving and backing up while you're working. From what I've learned, the `#files#` are autosave files that Emacs keeps around while you're actually editing a file, in case the program crashses. Like Microsoft Word's recovery files, you can get back up to speed fairly easily after a crash with `M-x recover-file`. Emacs will even tell you in the minibuffer when there's something to recover. `files~`, on the other hand, are backups, and represent the state of the file the last time you closed itout.

By default, Emacs keeps track of autosaves and backups in the same directory that the file is located in. That's pretty annoying. It turns out that, like all things in Emacs, you can change this behavior:

{% highlight lisp %}
;; Move backups to a special directory called .emacs-saves.
;; Also, specify how many versions of things to keep around.
(setq backup-directory-alist `(("." . "~/.emacs-saves")))
(setq delete-old-versions t
      kept-new-versions 6
      kept-old-versions 2
      version-control t)

;; Move auto-save files to the same directory
(setq auto-save-file-name-transforms `((".*" "~/.emacs-saves")))
{% endhighlight %}

This will prevent Emacs from polluting all of your working directories without losing the ability to recover from a crash.

### How do I insert spaces instead of tabs?
If you're like most normal people, you don't want to use `\t` to indent things. Instead, just insert some spaces. It turns out that Emacs has some pretty complicated behavior when it comes to `\t`, but more on that in a second. Here's how to insert spaces instead of tabs:

{% highlight lisp %}
;; Make the <tab> key insert spaces instead of a tab character
(setq indent-tabs-mode nil)

;; As a bonus, set the tab stops to be every four spaces instead of
;; every 8, which is the default
(setq tab-stop-list (number-sequence 4 120 4))
{% endhighlight %}

### Seriously, how do I just insert four spaces with the <tab> key?
This is the part of Emacs that, for me, was seriously absurd. It turns out that the default binding for `<tab>` is `indent-for-tab-command`, and it's changed to other, similar commands depending on the mode -- in `c-mode` it's `c-indent-line-or-region`. The behavior of this command seems to be to indent the current line to the "appropriate" level of indentation for the current mode, and then refuse to indent any further. This is normally fine when you're programming, can be a pain in the ass when you're in some kind of mixed-text document like this blog post. In that case, what you really want is for Emacs to stop telling you what the right thing to do is and just indent the damn line.

So, how do you do it? Well, the original thing that I came up with was to just bind a key to insert `"    "` that is, just insert four spaces. That looked something like this:

{% highlight lisp %}
;; I guess "interactive" makes something a command, which
;; this thing needs to be in order to let you bind a
;; key to it
(defun insert-spaces ()
  (interactive
   (insert "    ")))

(global-set-key (kbd "C-i") 'insert-spaces)
{% endhighlight %}

So it just inserts four spaces. That's cool. I eventually turned that into this:

{% highlight lisp %}
(global-set-key (kbd "C-i") (lambda ()
                                  (interactive)
                                  (insert-char ?\s tab-width)))
{% endhighlight %}

Which is better, because it uses the `tab-width` setting in order to determine how many spaces (that would be the `?\s` character) to enter. I chose `C-i` because it seemed like a good mnemonic ("**i**nsert tab" or "**i**ndent") and I thought all was well.

There's only one problem here: it turns out that, in Emacs land, `C-i` is actually an alias for the `<tab>` key. The reasons I'm sure all make perfect sense if you understand what computers were like 30 years ago, but last week it was just confusing. I like the normal behavior of `<tab>` most of the time, but sometimes I just want to indent some text. So how do I make `C-i` have the desired behavior? Well, once again, Google (and [Stack Overflow](http://stackoverflow.com/questions/1792326/how-do-i-bind-a-command-to-c-i-without-changing-tab)) to the rescue:

{% highlight lisp %}
;; Make C-i just INDENT THE DAMN TEXT *#(@$(#
(keyboard-translate ?\C-i ?\H-i)
(global-set-key (kbd "H-i") (lambda ()
                                  (interactive)
                                  (insert-char ?\s tab-width)))
{% endhighlight %}

What I ended up having to do was to turn all of my `C-i` commands into `H-i` commands (the `H` stands for "hyper" -- I have no idea what that means), and then bind my `insert-char` function to my newly-created `H-i` key. Whew! That was much more complicated than I expected, but it got the job done. I can now

{% highlight lisp %}
indent
        things
        at
                    will
{%  endhighlight %}

Neat.

### So... was it worth it?
I'm still not sure. I'm definitely writing this blog post in Emacs, and I do like it for normal text editing. It just _feels_ faster to be using keyboard shortcuts to move around the screen rather than having to jump over the the mouse every time something is more than five characters away. It's also been enlightening to find out that many of the same shortcuts translate into other programs.

For code, I'm sticking with Eclipse for now, but with the slight tweak that I'm using the [Emacs+](http://marketplace.eclipse.org/content/emacs) plugin so that I can keep practicing my keyboard shortcuts at the same time.

I guess that's not very conclusive. I'll check back in after a month or so and see where I'm at. For now, I'll just be busy abusing my `C-/` and `C-g` keys.
