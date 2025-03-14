---
title: "Neovim in <50 lines"
layout: post
category: programming
date: 2025-03-14
---

Neovim configs have a tendency to go off the rails. I think some amount of this has existed for a long time with Vim, but with Neovim it feels particularly exaggerated. At this point the giant bloated configs have become a part of the Neovim culture. To me it seems we've strayed pretty far from the minimalist philosophy of Vim once configs start requiring nested file structures and lazy-loaded plugins and whatever else is getting cooked up lately.

One side-effect of using more plugins is with each one you add, you get further abstracted away from using the base tool. When I started out on nvim I was pretty plugin happy, and there were things I would download plugins for that I didn't even realize you could do natively. There's something powerful about having command of as much of the base editor as possible. You can take those skills with you to pretty much any machine, and even if you don't have your perfect config set up you'll still feel like you know your way around.

I try to be as minimalist as possible without taking it so far that I'm sacrificing productivity. To me there are a few major modern IDE features that I think are huge enhancements to the development experience:

- Inline linting
- LSP completions & jump to definition
- Auto-formatting
- Fuzzy searching (both for file names and file contents)

Obviously the list is up for debate, but I feel good about these as a core set of productivity boosters. In this post, I'm going to walk through how you can get each of these with the minimal possible config. I think we can do it in less than 50 lines. I'll use Python as the language for my example setup, and macOS for my operating system, though with any *nix system it will look very similar besides the fact I'm using brew for installing system packages. Let's get to it.

## Getting a package manager

This is a prerequisite for our setup. Neovim does have a built in packages system that you could explore with `:h packages` if you wanted to go that route. What I don't like about this system is the packages you're using aren't specified anywhere in your configs. Instead, any plugins end up in your data directory only. This makes package management bit harder with version control, so we'll go with a 3rd party option.

The simplest package manager I've found is [paq-nvim](https://github.com/savq/paq-nvim). Install it like so from the command line with one command:

```
$ git clone --depth=1 https://github.com/savq/paq-nvim.git \
    "${XDG_DATA_HOME:-$HOME/.local/share}"/nvim/site/pack/paqs/start/paq-nvim
```

Then we just need to make sure we have a neovim config folder with

```
$ mkdir -p ~/.config/nvim
```

Next, create a file called `init.lua` in that directory and put the following lines at the top:

```lua
require("paq")({
    "savq/paq-nvim"
})
```

We have now a package manager. If you close and reopen Neovim and you should have a variety of `:Paq*` commands available to you. All of your future packages will go inside that initial setup call, which should stay at the top of the file. The format for adding packages is `<GitHub Org>/<GitHub Repo>`. Using `:PaqList` will show you what packages you have installed, and `:PaqSync` will both clean packages that have been removed and install new ones.

## Linting & LSP features

This one is a lot easier than you'd think with [`nvim-lspconfig`](https://github.com/neovim/nvim-lspconfig). All nvim-lspconfig does is provide a simplified way to set up the native LSP client for a large set of popular LSPs. The full list of linters and formatters it supports can be found [here](https://github.com/neovim/nvim-lspconfig/blob/master/doc/configs.md).

First let's add nvim-lspconfig to our packages:

```lua
require("paq")({
    "savq/paq-nvim",
    "neovim/nvim-lspconfig"
})
```

We're going to use pyright here for LSP features and completions, and ruff for linting. We'll set those up like so:

```lua
local lspconfig = require("lspconfig")
local lsps = { "pyright", "ruff" }
for _, lsp in pairs(lsps) do
    local setup = {}
    lspconfig[lsp].setup(setup)
end
```

All we're doing here is defining the LSPs we want to look for, and then setting them up for use in Neovim. We loop over our list of LSPs and call `setup` on each one to initialize it. Each LSP has different configuration options that can be set, found in the LSP's documentation. To modify our code to allow custom configurations for different LSPs, we'd simply add conditional logic in the for loop that adds entries to the setup table based on the current LSP name we're iterating over.

To be clear here, nvim-lspconfig *does not install the LSPs for you*. You need to do that yourself. This part is heavily dependent on what LSPs & linters you're using, but most I've used have been quite simple to install. For pyright & ruff, there are a few ways you can handle this. The simplest way would be to run:

```
$ brew install pyright ruff
```

...and the servers should be automatically be added to your $PATH and detected by Neovim. If you have Node installed on your system already I prefer to install pyright with `npm install -g pyright`. The brew install will install node as a dependency, which can override the default node version that is already set with `nvm`. For ruff, I prefer to install into the venv of the specific Python project I'm working on, but a global install works too. If you install into a venv, you just have to make sure the virtual environment is activated before you open Neovim so Neovim can find it.

## LSP completions

Surprise, this is already done! Neovim has a built in completion type called "omni completion" that the LSP will hook into by default if it's set up. You can read about it at `:h compl-omni`. When in insert mode, you can trigger it with `<C-X><C-O>`. To see the full list of completions available with `<C-X>`, check out `:h ins-completion`. The Neovim docs even give you a way you could upgrade this to autocomplete as you type, which you can see at `:h compl-autocomplete`.

## Formatting

For formatting we'll use a plugin called Conform. Formatting works very similarly to the LSP in that you just need your formatter in your `$PATH` and Conform will pick it up.

First, of course, add `stevearc/conform.nvim` to your paq setup. I won't run you through that one again.

Next we need to setup Conform. We can use the snippet below at the bottom of our `init.lua` file.

```lua
require("conform").setup({
    formatters_by_ft = {
        python = { "ruff_organize_imports", "ruff_format" },
    },
    format_after_save = {},
})
```

Take a look at all the valid formatting commands [here](https://github.com/stevearc/conform.nvim?tab=readme-ov-file#formatters). Luckily, ruff has a built-in formatter, so we don't need to install any extra packages. You'll notice two separate ruff commands -- one organizes imports and one runs the ruff formatter. We've also set up Conform to automatically format on save.

On to the next...

## Fuzzy finding

Fuzzy searching is the most efficient way to jump between files in Neovim. Instead of navigating through a traditional file tree (which there is one built in), you'll find it's usually faster to search your working directory for the file name instead. The fuzzy finder will live update as you type, and usually within a few keystrokes you'll get to the file you're looking for. The plugin we'll use for this not only allows searching file names, but also allows searches on the contents of files in your project.

We'll use a plugin called fzf-vim for this one, which is a Vim wrapper of the CLI application fzf. To start, we need to add these two packages to paq: `junegunn/fzf` and `junegunn/fzf.vim`, in that order.

Fzf-vim interfaces with fzf in the terminal via Neovim terminal mode. To make this work, we'll need a few applications installed to our system:

```
$ brew install fzf # the main fzf app
$ brew install ripgrep # for searching the contents of files
$ brew install bat # for syntax highlighting in fzf
```

Fzf is a tool you will use enough that it's worth having a couple keymaps for the main commands. Put the following right underneath your paq setup:

```lua
vim.g.mapleader = " "
vim.keymap.set("n", "<leader>f", ":Files!<cr>")
vim.keymap.set("n", "<leader>g", ":RG!<cr>")
```

In Vim, "leader" is a user defined key that is commonly used as a prefix for custom keymaps. It's there so we have a way to easily set keymaps without worrying about overwriting the many default ones Vim comes with. We've set it to the space bar here, which is a common leader key. Now `<space>f` will allow us to fuzzy search file names, and `<space>g` will search file contents!

## A few options

That's all our plugins. There's just a couple of key options worth changing and then we're off to the races. Add these right below your paq setup:

```lua
vim.opt.tabstop = 4
vim.opt.shiftwidth = 4
vim.opt.number = true
vim.opt.relativenumber = true
vim.opt.smartindent = true
```

Here we're doing a few things:

- Setting Neovim's tabs and indents to 4 spaces, overriding the somewhat antiquated default of 8 spaces
- Adding relative numbers (very useful for jumps), but still have regular line numbers turned on if relative numbers are switched off
- Allow Neovim to intelligently indent newlines based on context

## Bringing it all together

You've made it to the end. Quit and reopen Neovim, then run `PaqInstall` and your setup is complete! Let's take a look at our init.lua file after all this:

```lua
require("paq")({
    "savq/paq-nvim",
    "neovim/nvim-lspconfig",
    "stevearc/conform.nvim",
    "junegunn/fzf",
    "junegunn/fzf.vim"
})

vim.opt.tabstop = 4
vim.opt.shiftwidth = 4
vim.opt.relativenumber = true
vim.opt.number = true
vim.opt.smartindent = true

vim.g.mapleader = " "
vim.keymap.set("n", "<leader>f", ":Files!<cr>")
vim.keymap.set("n", "<leader>g", ":RG!<cr>")

local lspconfig = require("lspconfig")
local lsps = { "pyright", "ruff" }
for _, lsp in pairs(lsps) do
    local setup = {}
    lspconfig[lsp].setup(setup)
end

require("conform").setup({
    formatters_by_ft = {
        python = { "ruff_organize_imports", "ruff_format" },
    },
    format_after_save = {},
})
```

Less than 50 lines, just like we said. About half that actually! That's pretty good. Don't forget the system packages we installed! Below is every command we ran in the terminal:

```
$ git clone --depth=1 https://github.com/savq/paq-nvim.git \
    "${XDG_DATA_HOME:-$HOME/.local/share}"/nvim/site/pack/paqs/start/paq-nvim
$ brew install pyright ruff
$ brew install fzf ripgrep bat
```

## Just a few more plugins

If running only 5 plugins doesn't satiate you, there's a few more great ones I can suggest:

- **tpope/vim-surround**: A handy plugin for surrounding sections of code with brackets and quotations. This one is so widely used it should debatably be in Neovim core.
- **unblevable/quick-scope**: A small plugin that highlights unique characters in words on the current line making F/f/T/t horizontal navigation much easier.
- **tpope/vim-fugitive**: This is an incredible plugin for integrating git into your Vim editing. The main feature I use from it is the interactive blames, where for any line of code I can reblame at the parent commit and see when changes happened.
- **mbbill/undotree**: Vim has a powerful concept of the undo tree built in which lets you navgiate through the history of undos of any file. This plugin gives a nice UI to interface with that feature which is difficult to navigate on its own.
- **folke/which-key.nvim**: This plugin pops a handy floating window at the bottom of the screen that shows the currently valid keymaps. It will update as you press keys in sequence. I learned a ton about Neovim through using this initially.

## This is not the end

I am under no illusion that this is the perfect config, or that your workflow couldn't be improved by a few extra plugins. My personal config is ~100 lines where I have some additional keymaps set up, a few of the default vim options changed and a couple more plugins installed. The point here is to show how powerful Neovim can be out of the box. This is a very functional setup that you can easily build upon. It doesn't require 15 files and 40 plugins to set up a great dev environment -- the base editor is pretty damn good.

Happy vimming!

