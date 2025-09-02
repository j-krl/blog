---
title: "Ditching the Vim fuzzy finder part 1: :find"
layout: post
category: vim
date: 2025-09-02
---

if you're moderately acquainted with (n)vim you will have heard of the group of plugins called "fuzzy finders". They're very useful tools for searching for and selecting an entry from an arbitrary list of strings using a fuzzy matching algorithm. They can be used for a wide array of different useful searches, but when I used one most of my searches essentially boiled down to:

1. Searching for files by name
2. Searching through files by line contents

I will spend the next two posts giving you options for accomplishing the above two tasks using strictly native Vim features, along with three external terminal programs. In this post I will be covering file name search (1).

## Why do this?

Fuzzy finders are very nice plugins, and I won't pretend my solutions are going to be as pretty. These plugins can also do a lot more than search through file names and contents. I personally didn't utilize the other functionality much, but if you do you're likely not going to do away with your plugin.

The glaring thing left out of my native replacement is live feedback of query results on each key press, though I do a good job getting close to that. From a strict speed perspective I doubt there are improvments here either, but usually the difference will be negligible. Maybe I'm not exactly selling my case here, but what I do know is since moving off fzf.vim I never miss having a fuzzy finder.

Here are my main reasons for moving off these plugins:

- There is a certain satisfaction you get learning and using native Vim features. What you take from it feels more transferrable because you are learning the actual tool and not some ancillary plugin.
- There is less external code to rely on, and your config will feel less like a house of cards. You may have more lines of code in your .vimrc, but you're removing code that was previously abstracted away from you.
- It is more in line with the Unix philosophy of "doing one thing and doing it well". You allow Vim to stay as a text editor and use other programs to handle other functionality (like search).

If you don't care about these things then this probably isn't the post for you.

## Tools you'll need

For file name search you'll need two external programs:

1. [fd](https://github.com/sharkdp/fd): a simpler alternative to `find` for searching files
2. [fzf](https://github.com/junegunn/fzf): a command line fuzzy finder

It also wouldn't hurt to install [ripgrep](https://github.com/BurntSushi/ripgrep) now, but it's not yet necessary.

I am on nvim 0.11.3, the stable Neovim version as of this writing. I didn't do a ton of testing but I believe all of this should work on newer Vim versions that include 'findfunc'. There has even been [some recent improvements](https://github.com/vim/vim/commit/7e0df5eee9eab872261fd5eb0068cec967a2ba77) which are not yet on stable to the `matchfuzzy()` algorithm, which will be used in part 2 of this post. You could avoid using fzf altogether here if you adapted my findfunc to use `matchfuzzy()` instead of piping into fzf. I'll continue with fzf because I find I get better matches with it (on nvim 0.11.3 at least).

Let's get to it!

## Searching file names with :find

:find is the command we'll use to do file name fuzzy finding. If you're not familiar, :find is similar to :edit, but it searches for files based on the list of directories in your 'path' setting (which you can read about at `:h 'path'`).

The new option that makes :find so much more powerful is the 'findfunc' option. You can set your findfunc to a funcref, and :find will call that function instead of its default behaviour. The function is used both when invoking :find or getting completion matches for :find. The first argument of the callback is the current :find argument, and the second is a boolean indicating if the call is for getting completion matches.

If we really want to ditch our fuzzy finder, there's some minimal functionality we want out of :find:

- Search the entire cwd, ignoring .gitignored files
- Match on the full path, not just the file name
- Use fuzzy matching, not exact pattern matching

Amazingly, we can get this all with one simple command in our findfunc:

```bash
fd --hidden . | fzf --filter=<pattern>
```

We pipe all the files in our repo into fzf, then let fzf return the results fuzzy ranked based on our pattern. Now let's adapt the command to Vim and put it inside our findfunc:

```vimscript
function! FuzzyFindFunc(cmdarg, cmdcomplete)
    return systemlist("fd --hidden . \| fzf --filter='" 
        \.. a:cmdarg .. "'")
endfunction

if executable('fd') && executable('fzf')
    set findfunc=FuzzyFindFunc
endif
```

Then we'll add a couple mappings for quick finding:

```vimscript
nnoremap <leader>f :find<space>
nnoremap <leader>F :vert sf<space>
```

It's that easy! You'll notice that we never once had to mess with our 'path', which can be a finnicky option to get right. We allow `fd` to ignore sensical paths itself when searching, as it ignores .gitignored files and directories by default. If you want to add some paths to ignore for :find, you can add an `.fdignore` file to your working directory.

> **Disclaimer:** The command is going to be quite slow if you're running it from somewhere with a lot of files like your home directory, especially with the inclusion of hidden files. We're recursively piping every file and folder from the cwd down into fzf. If you're typically working out of a large directory, I'd recommend ditching fzf and sending the pattern directly to fd. You lose fuzzy finding but can use regex with fd instead, and the performance boost will be well worth it.

## Getting the best :find feedback we can

As mentioned above, We don't get live feedback with this method like we would with a plugin, but we can do *pretty* good with 'wildchar' command mode completion.

You can read `:h findfunc`, but the second findfunc arg `cmdcomplete` is for determining if the call is to show completion options, so we've already got that out of the box. What we want here is a good 'wildmode' setting.

wildmode is just the completion mode used when you hit your command mode completion key (\<tab\> by default). :find will be a lot cleaner when you set it like so in your .vimrc:

```vimscript
set wildmenu
set wildmode=noselect:longest,full
```

The wildmode options work similarly to their counterparts in 'completeopt'. "noselect" shows the completion options in a popup menu when you trigger a completion, but won't select the first option like the default "full" setting does. Only on the next trigger will we select the next completion item via the "full" after the comma. wildmode options separated by commas dictate the behaviour for each subsequent completion trigger. Go read `:h wildmode` for all the details.

We'll append "longest" to noselect because these are our options for *all* command mode completion triggers, and it sucks to have to trigger twice when not using :find. It's a nice compromise because I rarely get a match on the ":longest" bit when using :find because the fuzzy matching returns so many candidates.

If you want to explore true autocomplete options for command mode, check out [this similar post](https://cherryramatis.xyz/posts/native-fuzzy-finder-in-neovim-with-lua-and-cool-bindings/) on using :find for fuzzy finding files. The author takes it a little further by using `wildtrigger()` with an autocmd for triggering the completion menu with noselect on every key press. It's a pretty complex lua setup and I wanted to keep mine a little simpler, but you can look at that if you want to add some interactivity to the :find command.

## Send files to the quickfix list

Commonly in fuzzy finder plugins there will be a map to add the results of the search to the quickfix list. We can do that with a function, command and mapping:

```vimscript
function! FdSetQuickfix(...) abort
    let fdresults = systemlist("fd -t f --hidden " .. join(a:000, " "))
    if v:shell_error
        echoerr "Fd error: " .. fdresults[0]
        return
    endif
    call setqflist(map(fdresults, {_, val -> 
        \{'filename': val, 'lnum': 1, 'text': val}}))
    copen
endfunction

command! -nargs=+ -complete=file_in_path Findqf call FdSetQuickfix(<f-args>)

nnoremap <leader>d :Findqf<space>
```

You can run :Findqf and pass any arguments you want to `fd`, similarly to how :grep works. This will only match on exact regex matches instead of fuzzy scores.

## Bonus: most recently used files via :buffers

While not directly related to :find, I wanted to mention one more powerful navigation tool in the :buffers command. We can emulate another fuzzy finder feature: navigating through most recently used buffers. With our current wildmode setting, triggering a completion for the :buffers command shows the current list of buffers in order of buffer index -- not particularly useful. What we're really looking for is the "lastused" wildmode option. Let's add it to the wildmode we set earlier:

```vimscript
set wildmode=noselect:longest:lastused,full
```

As documented in `:h wildmode`, if we trigger a completion on a buffer name, lastused will sort buffers by last time used. We can add a simple mapping:

```vimscript
nnoremap <leader>b :b<space>
```

...and now when we tab complete :buffers our top suggestions will be our most recently used files.

## Conclusion

Putting it all together, here's all our :find and :buffers config:

```vimscript
set wildmenu
set wildmode=noselect:longest:lastused,full
if executable('fd') && executable('fzf')
    set findfunc=FuzzyFindFunc
endif

nnoremap <leader>f :find<space>
nnoremap <leader>F :vert sf<space>
nnoremap <leader>b :b<space>
nnoremap <leader>d :Findqf<space>
command! -nargs=+ -complete=file_in_path Findqf call FdSetQuickfix(<f-args>)

function! FuzzyFindFunc(cmdarg, cmdcomplete)
    return systemlist("fd --hidden . \| fzf --filter='" 
        \.. a:cmdarg .. "'")
endfunction

function! FdSetQuickfix(...) abort
    let fdresults = systemlist("fd -t f --hidden " .. join(a:000, " "))
    if v:shell_error
        echoerr "Fd error: " .. fdresults[0]
        return
    endif
    call setqflist(map(fdresults, {_, val -> 
        \{'filename': val, 'lnum': 1, 'text': val}}))
    copen
endfunction

```

Not bad for a solid replacement of some major fuzzy finder functionality!

That's all I've got for now. This post got long enough that I will break :grep into a part 2 post in the coming days. If I didn't convince you to ditch fzf.vim or telescope hopefully you learned something at the very least!

