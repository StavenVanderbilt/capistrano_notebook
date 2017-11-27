# Install git and Vundle
  Install Vundle to manage Vim plugins，go to github search the vundle vim plugin. it will tell you how to install it clearly.

```
git clone https://github.com/gmarik/Vundle.vim.git ~/.vim/bundle/Vundle.vim
```




=================================================================================
# Edit your .vimrc file

```

set nocompatible              " be iMproved, required
filetype off                  " required
" set the runtime path to include Vundle and initialize
set rtp+=~/.vim/bundle/Vundle.vim
call vundle#begin()
" alternatively, pass a path where Vundle should install plugins
"call vundle#begin('~/some/path/here')
" let Vundle manage Vundle, required
Plugin 'gmarik/Vundle.vim'
" Code/project navigation
Plugin 'scrooloose/nerdtree'              " Project and file navigation
Plugin 'majutsushi/tagbar'                " Class/module browser
" others
Plugin 'vim-airline/vim-airline-themes'   " Lean & mean status/tabline for vim
Plugin 'fisadev/FixedTaskList.vim'        " Pending tasks list
Plugin 'rosenfeld/conque-term'            " Consoles as buffers
Plugin 'tpope/vim-surround'               " Parentheses, brackets, quotes, XML tags, and more
Plugin 'ctags.vim'
" language support
Plugin 'elixir-lang/vim-elixir'
" The following are examples of different formats supported.
" Keep Plugin commands between vundle#begin/end.
" plugin on GitHub repo
" Plugin 'tpope/vim-fugitive'
" plugin from http://vim-scripts.org/vim/scripts.html
" Plugin 'L9'
" Git plugin not hosted on GitHub
" Plugin 'git://git.wincent.com/command-t.git'
" git repos on your local machine (i.e. when working on your own plugin)
" Plugin 'file:///home/gmarik/path/to/plugin'
" The sparkup vim script is in a subdirectory of this repo called vim.
" Pass the path to set the runtimepath properly.
" Plugin 'rstacruz/sparkup', {'rtp': 'vim/'}
" Avoid a name conflict with L9
" Plugin 'user/L9', {'name': 'newL9'}
" All of your Plugins must be added before the following line
call vundle#end()            " required
filetype plugin indent on    " required
" To ignore plugin indent changes, instead use:
"filetype plugin on
"
" Brief help
" :PluginList          - list configured plugins
" :PluginInstall(!)    - install (update) plugins
" :PluginSearch(!) foo - search (or refresh cache first) for foo
" :PluginClean(!)      - confirm (or auto-approve) removal of unused plugins
"
" see :h vundle for more details or wiki for FAQ
" Put your non-Plugin stuff after this line
"
syntax on
set ruler
" autocmd vimenter * TagbarToggle
" autocmd vimenter * NERDTree
" autocmd vimenter * if !argc() | NERDTree | endif

colorscheme desert
set nu
set nobackup
set smarttab
set tabstop=2
set laststatus=2
let g:airline_theme='badwolf'
let g:airline#extensions#tabline#enabled = 1
let g:airline#extensions#tabline#formatter = 'unique_tail'
map <F4> :TagbarToggle<CR>
let g:tagbar_autofocus = 0
map <F3> :NERDTreeToggle<CR>
map <F2> :TaskList<CR>

" alchemist for Elixir
Plugin 'slashmili/alchemist.vim'

```


** Notes that this file is mainly for Elixir. **

