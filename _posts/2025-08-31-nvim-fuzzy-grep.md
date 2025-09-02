---
title: "Ditching the Vim fuzzy finder plugin part 2: :grep"
layout: post
category: vim
date: 2025-08-31
published: false
---

We've got file names and now we need file contents, and our command for this is :grep. This one's a little easier because instead of using completion for matching, we feed our results to the quickfix list.

:grep has a similar customization option to findfunc available called 'grepprg' which allows us to set the command that gets run in the terminal that returns our list of results. We will set ours as follows:

```vimscript
set grepprg=rg\ --vimgrep\ --hidden\ -g\ '!.git/*'
```

This is actually close to the Neovim default, which uses ripgrep automatically if you have it installed. I just like including hidden files as a matter of preference.

If you're like me, you'll mostly grep for fixed strings and not use a ton of regex. Regex has its place, but I mostly use :grep for quick navigation, using the simplest substring I can think of that will get me close to what I'm looking for.

The one exception to this is the greedy wildcard. This I use a lot to break a query into unique substrings for more precise matching. For example, maybe I'm trying to find the definition of some class called `Event` in python. I could use `:grep Event` but that will also match every place I import or instatiate `Event`. `:grep class Event` would certainly work, but it feels like a lot of typing and we can do better.

A quick and easy way to do this is to use `:grep cl.*?Ev`. I'm searching for the first couple letters of "class", then greedy matching to the first couple letters of `Event`. That may not seem any easier than `:grep class Event` but if you add `.*?` as a command mode mapping it becomes a lot faster.

So here's our keymaps for quick and easy grepping:

```vimscript
nnoremap <leader>g :grep<space>
nnoremap <leader>G :grep <C-R><C-W><cr>
cnoremap <C-space> .*?
```

Easy enough. You can read about what the various rg options do on the rg man page. You can also customize :grep's behaviour once the results are returned. If you don't like the results being shown in the pager, you can run `:sil grep`. If you also don't want Vim to jump to the first result, you can add a bang and do `:sil grep!`.

## Fuzzy matching with :grep

So far we only have exact pattern matching. To fuzzy match our query we can pipe the every line of every file into fzf like so:

```bash
rg --column --hidden -g '!.git/*' . | fzf --filter=<pattern> \
    --delimiter : --nth 4..
```

...adapted from [this](https://github.com/junegunn/fzf.vim/issues/346#issuecomment-288483704) GitHub answer.

The only problem is we can't have two grepprg's at once, but both commands are useful. We can put the fuzzy version inside a function like so:

```vimscript
function! FuzzyGrep(query)
    let oldgrepprg = &grepprg
    set grepprg=rg\ --column\ --hidden\ -g\ '!.git/*'\ .\ \\\|\ fzf\
        \ --filter='$*'\ --delimiter\ :\ --nth\ 4..
    exe 'grep ' .. a:query
    let &grepprg = oldgrepprg
endfunction
```

So we'll temporarily change the grepprg when we're fuzzy searching. The "$*" is where the arguments provided to :grep will be expanded. FYI, the "delimiter" and on section of the command just prevents fzf from including the file names and columns in the matching process.

Now let's put it in a command and add some keymaps:

```vimscript
command! -nargs=1 Zgrep call FuzzyGrep(<f-args>)
nnoremap <leader>z :Zgrep<space>
```

`<leader>G` is a useful one as it greps for the current work under the cursor.

**A warning about FuzzyGrep:** The command is piping each line of code in the repo through fzf to be fuzzy filtered and sorted. If you are in a large repo this will be a *very* slow command. Which leads us to our next section..

## A fuzzy grep middle ground

Most fuzzy finder plugs don't do fuzzy matching via live grep because it is an expensive operation. The rg -> fzf command works well enough for me on a smaller code base, but we need a more reliable solution as well.

What the plugins often do is they'll have some kind of second filter you can apply to your grep results, which fuzzy matches them *after* the grep is complete. This is a nice compromise and not nearly as resource intensive. We'll try something like that here.

We'll do the secondary matching once our original grep has been sent to the quickfix list. So what we need is a quickfix fuzzy match and sort function. We'll make a separate command for it:

```vimscript
function! FuzzyFilterQf(pattern)
    let fuzzy_results = matchfuzzy(getqflist(), a:pattern, {'key': 'text'})
    call setqflist(fuzzy_results)
    cfirst
endfunction

command! -nargs=1 Cfuzzy call FuzzyFilterQf(<f-args>)
```

You'll notice the `matchfuzzy()` call, which is an internal fuzzy matching algo added in the last few years. We're taking the quickfix list, fuzzy filtering it, then adding a new quickfix with the results. There were even improvements pushed to the `matchfuzzy()` algo in the last few weeks as of this writing, and I noticed a significant improvement between nvim 0.11 and 0.12 versions.

Let's put our grep -> fuzzy filter in a single concise command, for times we want that:
```vimscript
function! FuzzyFilterGrep(query, path=".")
    exe "grep '" .. a:query .. "' " .. a:path
    let sort_query = substitute(a:query, '\.\*?', '', 'g')
    let sort_query = substitute(sort_query, '\\\(.\)', '\1', 'g')
    call FuzzyFilterQf(sort_query, 1)
endfunction

command! -nargs=+ -complete=file_in_path Zgrep call FuzzyFilterGrep(<f-args>)
```

Let's take my use case where I grepped for `class Event` with `cl.*?Ev`. Maybe we get a lot of matches because we have a `class EventAction`, `class EventType`, `class EventLog`, etc. So the flow here is to:

1. Do a regular :grep with my `cl.*?Ev` pattern
2. Fuzzy match the results in the qflist with `clEv` (our previous pattern with regex characters removed) and create a new qflist

This should rank the most concise match at the top (the `class Event` that we want). The two `substitute` calls in `FuzzyFilterGrep` are for removing regex greedy wildcard matches and escape backslashes. I'm sure it's not a perfect way to "unregexify" a string, but it's worked for my purposes.

<!-- all the path stuff in the grep commands that wasnt there before -->
