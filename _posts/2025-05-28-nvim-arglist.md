---
title: "File navigation with the Vim arglist"
layout: post
category: vim
tags:
    - tech
    - programming
    - vim
    - neovim
date: 2025-05-28
---

One of the hardest things to wrap your head around when starting out in Vim is how to navigate files efficiently. A specific situation that can be frustrating is when you're navigating often between 3 or 4 files. You may navigate away from those files but regularly return to them. If you're doing something like iterating over a long quickfix list, or following a chain of several function definitions, using the jumplist to navigate back can be tedious. You could search through your buffer list, but if you have lots of open buffers in a code base with many similarly named files this might not be very efficient. Maybe you've tried using global marks, but with too many files the cognitive load of recalling which mark letter you want is inconvenient.

There's one more tool you can try! Let me introduce the Vim argument list.

## The argument list: a low level tool

The argument list is a fairly low level, rough around the edges Vim feature. Opening `:h arglist` the first thing you'll see is how it's meant to provide navigation between files when you open multiple on launch. This is one way the arglist can be used, though not the most compelling way. For demo purposes, let's generate an arglist this way when launching Vim. Run the following command:

```
vim file1.txt file2.txt file3.txt
```

Now once vim is open, if you run `:arg` you should see the following in the command line:

```
[file1.txt] file2.txt file3.txt
```

Our first argument list. The filename wrapped in square brackets is the current arglist file. The current arglist file is *unrelated* to the file you are currently editing, and the selected index will not change as you change files in the normal ways. The index only changes when you move it explicitly with some command like `:next`, `:prev`, or `:argu`, for example. This is a bit unintuitive at first but starts to make sense with more reps.

You can add files to the arglist once you're in Vim with `:arge` or `:arga`. The commands take a count and a name argument with the count representing the arglist index for the new file. You can do things like `:$arga %` to add the current buffer to the end of the list, or `:arga *.txt` to add several files at once. `:argu` used with a count is for switching to the file at the count'th index, or with no count to the current arglist file. `:next` and `:prev` will go to the next or previous file, and also take a count. Starting at Neovim 0.11 they also have built in `[a` and `]a` mappings (as well as `[A` and `]A` for `:first` and `:last`), taken from the vim-unimpaired plugin.

## Improving argument list navigation

Now for the enhancements. The arglist commands on their own are not efficient enough to justify using them over a plugin with similar functionality like [Harpoon](https://github.com/ThePrimeagen/harpoon/tree/harpoon2) or [Marks.nvim](https://github.com/chentoast/marks.nvim), so we need to add some mappings to get a smoother experience.

The first noticeable issue is the lack of feedback when using the arglist. I want to be able to view its current state easily, and when I navigate with `:next` or `:prev` I want some visual feedback that shows the state of the arglist after the change. It makes iterating over the arglist feel super snappy, taking less mental overhead than a global mark.

Getting good feedback with the arglist takes some massaging, because everything gets displayed through the command line. We'll start with a simple mapping for viewing the state of the arglist that demonstrates this below.

```
nnoremap <F2> <C-L><cmd>args<cr>
```

Simple enough -- the only problem is if there is no current arglist, `:args` won't have any effect, but we still want some visual feedback for our command. Redrawing the screen before running `:args` emulates the behaviour of `:args` printing nothing when no arglist exists.

We'll now add the `[a`, `]a`, `[A`, and `]A` vim-unimpaired mappings. Even if you're running Neovim >= 0.11, we'll enhance default versions of these like so:

```
nnoremap [a <cmd>exe v:count1 .. 'N'<bar>args<cr><esc>
nnoremap ]a <cmd>exe v:count1 .. 'n'<bar>args<cr><esc>
nnoremap [A <cmd>first<bar>args<cr><esc>
nnoremap ]A <cmd>last<bar>args<cr><esc>
```

We're going to the next or previous arglist file, but then printing the list afterwards. This way when you hit `[a` or `]a` you can see the result of your command and if you need to iterate again. I've added `<esc>` to the end of the command because if the list is long enough to span multiple lines we don't want the dreaded "Hit ENTER" behvaiour of the pager, so we'll skip it in those cases.

One annoying thing about the `[a` and `]a` mappings is that they don't wrap when at the end or beginning of the list. If we want to get really fancy we can create our own `:next` and `:prev` commands that do that with a function like so:

```vim
nnoremap [a <cmd>call NavArglist(v:count1 * -1)<bar>args<cr><esc>
nnoremap ]a <cmd>call NavArglist(v:count1)<bar>args<cr><esc>

function! NavArglist(count)
    let arglen = argc()
    if arglen == 0
        return
    endif
    let next = fmod(argidx() + a:count, arglen)
    if next < 0
        let next += arglen
    endif
    exe float2nr(next + 1) .. 'argu'
endfunction
```

In `NavArglist` We use Vim builtins like `argc()` to get the length of the arglist and `argidx()` to get the current arglist index. Then we can use modulo division with the passed count to navigate to the new index after wrapping.

Lastly, we want to be able to jump to an arglist file by index. We can do that with the following command:

```
nnoremap <expr> ga ":<C-U>" .. (v:count > 0 ? v:count : "") .. 
    \"argu\|args<cr><esc>"
```

With `[count]ga` we jump to the arglist file at the count'th index, then print the arglist afterwards. The most useful part of the command is the count is optional, and without one you'll jump to the current arglist file.

## Improving argument list modification

Now we'll get to modifying the arglist. The behaviour of typing something like `:arga otherfile.txt` to add a file I'm not currently editing isn't something I ever find myself reaching for. Most of the utility here is adding the file you're currently editing that you know you'll keep returning to. We'll do that like so:

```
nnoremap <leader>aa <cmd>$arge %<bar>argded<bar>args<cr>
```

Again, what makes this so much better than a global mark is we don't have to come up with a mark letter. It's always going to be `<leader>aa`. With global marks usually what I want is to be marking the *file*, not the cursor position, and that is what the arglist does. When navigating to an arglist file the cursor goes back to its previous position.

To break down our command a bit, `:$arge %` adds the current file to the end of the arglist and edits it. Since we're just using the current file, it doesn't much matter if we use `:arge` or `:arga`. The `argded` part is because `arga` and `arge` *do not prevent duplicates*. `argded` simply removes any duplicates from the list, so we're just ensuring that we can run `<leader>aa` as many times as we want on a buffer without adding multiple entries. Finally we'll add in `args` again like with other commands.

Next is the delete and clear commands:

```
nnoremap <leader>ad <cmd>argd %<bar>args<cr>
nnoremap <leader>ac <cmd>%argd<cr><C-L>
```

The delete command isn't much different from the add command. The clear command is similar, but `%` in this context is just a range shortcut for `1,$`, meaning delete all files (see `:h range`). Again we add `<C-L>` to show nothing after the list has been cleared.

## Other argument list enhancements

Vim's statusline has a nice integration with the arglist. I personally use the default statusline shown in `:h statusline` with one small tweak:

```
set statusline=%<%f\ %h%m%r%=%-13a%-13.(%l,%c%V%)\ %P
````

All I've added from the default is the `%-13a` which shows the argument list status in relation to the current file. If you're editing file 1 of 3 in the arglist, `(1 of 3)` will show in the statusline. If the current arglist index is at 1, but that isn't the file you're currently editing, this will show as `((1) of 3)`.

Lastly, if you're like me you might use tabs as separate workspaces in Vim. In these cases it's convenient to work with a separate arglist per tab. For this you can use local argument lists (see `:h arglocal`). These are created on a per-window basis. If you know you're always going to want tabs to have their own arglists, you can add the following autocommand (Neovim only):

```
autocmd vimrc TabNewEntered * argl|%argd
```

This way, every time you enter a new tab, you'll immediately create a fresh local arglist for the first window that's opened. Local arglists are per window, but if you split that window, the same arglist will be used in the split. In this way you can keep using the same list within the tab. `argl|%argd` makes a local copy of the global argument list, and then immediately clears it, giving you a new empty arglist in your new tab.

## Final thoughts

I think the argument list is generally under-utilized, but with the few enhancements I mentioned it becomes a powerful bookmarking system and navigation tool between a small set of files within Vim. I'd encourage anyone to try it before reaching for one of the more popular bookmarking plugins, as it may do everything you need on its own. It's surprising how often Vim has a native solution for something you'd think could only be solved with a plugin!
