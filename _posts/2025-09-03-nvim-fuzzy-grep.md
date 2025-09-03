---
title: "Ditching the Vim fuzzy finder plugin part 2: :grep"
layout: post
category: vim
date: 2025-09-03
published: true
---

This post is a continuation of [my last post](https://jkrl.me/vim/2025/09/02/nvim-fuzzy-find.html) on ditching the Vim fuzzy finder plugin. In that post I covered file name search with :find and :buffers. In this post I will cover file content search with :grep.

With :grep we take a fundamentally different approach because we are not immediately navigating to the top result, but sending all results to the quickfix list. Everything we do in this post will be rooted in the qflist.

If you followed my last post you should have already installed [fzf](https://github.com/junegunn/fzf). The second external program you'll need to follow along here is [ripgrep](https://github.com/BurntSushi/ripgrep), which is a grep alternative that ignores .gitignored files by default. If wanted a solution with less external dependencies, you could adapt my config to use GNU grep (usually installed by default) and the `matchfuzzy()` builtin instead of fzf.

## :grep in a nutshell

Vim has two flavours of grep: :vimgrep and :grep. :vimgrep is the internal Vim option for grepping, whereas :grep is a command that relies on an external program (in our case `rg`). The command called in the shell when we call :grep is defined by 'grepprg'. Let's set ours like so:

```vimscript
set grepprg=rg\ --vimgrep\ --hidden\ -g\ '!.git/*'
```

This is actually close to the Neovim default, which uses ripgrep automatically if you have it installed. I just like ignoring gitignored files as a matter of preference.

So to use :grep it's `:grep` and any arguments provided are passed to the grepprg under the hood. You could narrow search down to a certain directory with `:grep <pattern> mydir/`. You could also pass extra flags or options to rg. Anything you want to do.

When you execute :grep, by default a pager of the results are shown, you hit ENTER and the editor jumps to the first match. To ignore the pager output, you can run it with `:sil grep`. To prevent the default jump, run grep with a bang: `:grep!`.

## :grep mappings

A fundamental aspect of :grep and ripgrep is the ability to match based on a regex pattern. However, I find myself reaching for :grep mostly to search for fixed strings, and usually don't get too complex with the regex patterns. I usually use it as a quick navigation tool to search for the simplest substring possible to get me to what I'm looking for.

There is one exception to this rule: greedy wildcards. Sometimes I can get the best match possible if I search for two different substrings that exist on the same line. Greedy wildcard match is `.*?` in ripgrep, and with this knowledge we can make some solid :grep mappings:

```vimscript
nnoremap <leader>g :grep ''<left>
nnoremap <leader>G :grep <C-R><C-W><cr>
cnoremap <C-space> .*?
"nvim only
cnoremap <A-9> \(
cnoremap <A-0> \)
```

The `<A-9>` and `<A-0>` mappings are for escaping open and close brackets in regex -- a common search pattern for me, though the mappings don't seem to work in Vim. The `<leader>G` map is for grepping for the word under the cursor -- a very useful command.

## Fuzzy filtering the quickfix list

Now we look into fuzzy matching our :greps. What the fuzzy finder plugins often do is have a second fuzzy match filter you can apply to your grep results. So you run your live grep, hit some kind of keymap, then type your fuzzy match pattern and the grep results are refiltered and sorted.

Vim does have a built in way to regex filter quickfix lists with :Cfilter, which is something you can check out. It's a plugin that comes with Vim that you source with `:packadd cfilter`. The nice part is the command also filters by file path. Read about it at `:h cfilter`.

But we're trying to do fuzzy matching here, so let's make a command similar to :Cfilter and call it :Cfuzzy. We can define the command with some mappings like so:

```vimscript
nnoremap <leader>cf :Cfilter<space> 
nnoremap <leader>cz :Cfuzzy<space> 
nnoremap <leader>co :colder<space> 
nnoremap <leader>cn :cnewer<space> 

command! -nargs=1 Cfuzzy call FuzzyFilterQf(<f-args>)

function! FuzzyFilterQf(...) abort
    call setqflist(matchfuzzy(getqflist(), join(a:000, " "), {'key': 'text'}))
endfunction
```

The `FuzzyFilterQf()` function accepts any number of arguments and joins them in a single string, allowing us to use :Cfuzzy without having to escape spaces. You'll notice `matchfuzzy()` gets used internally, which is the built-in Vim fuzzy matching function. I also added a map for :Cfilter (because why not) and maps for :colder and :cnewer, because you might want to undo or redo the filtering of your quickfix list.

## :grep and fuzzy filter in a single command

I have found it very useful to :grep for a pattern and then fuzzy match it all in one step. I call the command `Zgrep`:

```vimscript
nnoremap <leader>z :Zgrep<space>
"nvim only
cnoremap <A-space> \<space>

command! -nargs=+ -complete=file_in_path Zgrep call FuzzyFilterGrep(<f-args>)

function! FuzzyFilterGrep(query, path=".") abort
    exe "grep! '" .. a:query .. "' " .. a:path
    let sort_query = substitute(a:query, '\.\*?', '', 'g')
    let sort_query = substitute(sort_query, '\\\(.\)', '\1', 'g')
    call FuzzyFilterQf(sort_query)
    cfirst
    copen
endfunction
```

It's a little rough around the edges, but the process is:

1. :grep for some provided pattern (optionally in some subdirectory)
2. Attempt to "de-regexify" the provided pattern
3. Fuzzy filter the qflist based on the new pattern

I find this command works best for matching a few short substrings. Say in a python project I want to find a `class Event`, but there are many other classes like `class EventAction`, `class EventLog`, etc. If I do `:Zgrep cl.*?Ev`, I'll grep for all these class definitions, then match on the most concise one (`class Event`). I use this workflow regularly and it feels very smooth.  

I am sure a lot of improvements could be made here. My attempt to remove regex is very simple, just removing greedy wildcards and backslashes (generally all the regex I use for :grep). The function also takes exactly one or two arguments, so you'll have to escape any spaces in the pattern (nvim mapping provided for this). If you don't find yourself grepping within a directory much you could remove the `path` argument and accept any number with `...`, then join them like I did in `FuzzyFilterQf()`. Whatever works best for you.

Note that on nvim 0.11 the `matchfuzzy()` results are not as good as the `fzf` output. However, there have been [some recent improvements](https://github.com/vim/vim/commit/7e0df5eee9eab872261fd5eb0068cec967a2ba77) to `matchfuzzy()` which are not yet on stable Neovim (as of this writing), but will be in 0.12. I didn't find the same algorithm issues on Vim 9.1.

## Fuzzy matching your entire working directory

Let's take this a step further. Why not skip the whole :grep process and just fuzzy match on every line in your working directory? The command would be this:

```bash
rg --column --hidden -g '!.git/*' . | fzf --filter=<pattern> \
    --delimiter : --nth 4..
```

...adapted from [this](https://github.com/junegunn/fzf.vim/issues/346#issuecomment-288483704) GitHub answer from the author of fzf.

There's actually a good reason this wasn't my first recommendation: this is a *very* expensive operation. If you're searching for too general a pattern or in even a moderate sized directory, this command will be extremely slow.

However, I think this approach is still worth exploring, because it is very convenient when used properly. We'll just temporarily change our grepprg to have rg pipe into fzf, then run grep normally:

```vimscript
" WARNING: slow!
nnoremap <leader>Z :Fzfgrep<space>

command! -nargs=+ -complete=file_in_path Fzfgrep call FzfGrep(<f-args>)

function! FzfGrep(query, path=".")
    let oldgrepprg = &grepprg
    let &grepprg = "rg --column --hidden -g '!.git/*' . " 
        \.. a:path .. " \\| fzf --filter='$*' --delimiter : --nth 4.."
    exe "grep " .. a:query
    let &grepprg = oldgrepprg
endfunction
```

This command has the same issue with escaping spaces because of the one or two arguments, but you may want limit this to subdirectories enough that it's worth it. Again, whatever you want to do with it.

## Putting it all together

Here is all the code from part 1 and part 2 of my post, implementing both file name and file content fuzzy matching:

```vimscript
packadd cfilter

set wildmenu
set wildmode=noselect:longest:lastused,full
set grepprg=rg\ --vimgrep\ --hidden\ -g\ '!.git/*'
if executable('fd') && executable('fzf')
    set findfunc=FuzzyFindFunc
endif

nnoremap <leader>f :find<space>
nnoremap <leader>F :vert sf<space>
nnoremap <leader>b :b<space>
nnoremap <leader>d :Findqf<space>
nnoremap <leader>g :grep ''<left>
nnoremap <leader>G :grep <C-R><C-W><cr>
nnoremap <leader>z :Zgrep<space>
nnoremap <leader>Z :Fzfgrep<space>
nnoremap <leader>cf :Cfilter<space> 
nnoremap <leader>cz :Cfuzzy<space> 
nnoremap <leader>co :colder<space> 
nnoremap <leader>cn :cnewer<space> 
cnoremap <C-space> .*?
"nvim only
cnoremap <A-9> \(
cnoremap <A-0> \)
cnoremap <A-space> \<space>

command! -nargs=+ -complete=file_in_path Findqf call FdSetQuickfix(<f-args>)
command! -nargs=1 Cfuzzy call FuzzyFilterQf(<f-args>)
command! -nargs=+ -complete=file_in_path Zgrep call FuzzyFilterGrep(<f-args>)
command! -nargs=+ -complete=file_in_path Fzfgrep call FzfGrep(<f-args>)

function! FuzzyFilterGrep(query, path=".") abort
    exe "grep! '" .. a:query .. "' " .. a:path
    let sort_query = substitute(a:query, '\.\*?', '', 'g')
    let sort_query = substitute(sort_query, '\\\(.\)', '\1', 'g')
    call FuzzyFilterQf(sort_query)
    cfirst
    copen
endfunction

function! FuzzyFilterQf(...) abort
    call setqflist(matchfuzzy(getqflist(), join(a:000, " "), {'key': 'text'}))
endfunction

function! FzfGrep(query, path=".")
    let oldgrepprg = &grepprg
    let &grepprg = "rg --column --hidden -g '!.git/*' . " 
        \.. a:path .. " \\| fzf --filter='$*' --delimiter : --nth 4.."
    exe "grep " .. a:query
    let &grepprg = oldgrepprg
endfunction

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

About 65 lines. Since adopting this approach I have never once reached for a fuzzy finder plugin, and file navigation is totally second nature. I hope your experience is the same!
