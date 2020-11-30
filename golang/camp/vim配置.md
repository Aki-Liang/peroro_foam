# golang vim 配置

## vim-go
https://github.com/fatih/vim-go

## coc

https://github.com/neoclide/coc.nvim

```
" plug
call plug#begin()
Plug 'joshdick/onedark.vim'
Plug 'itchyny/lightline.vim'
Plug 'majutsushi/Tagbar'
Plug 'preservim/nerdtree'
Plug 'fatih/vim-go', { 'do': ':GoUpdateBinaries' }
call plug#end()

set fileencodings=utf-8,ucs-bom,gb18030,gbk,gb2312,cp936
set termencoding=utf-8
set encoding=utf-8

" 自动提示
set completeopt=menu

" 语法高亮
syntax on
" 侦测文件类型
filetype on
" 载入文件类型插件
filetype plugin on
" 为特定文件类型载入相关缩进文件
filetype indent on


" show print margin
set colorcolumn=79
" 设置背景
" set background=dark
" 配色方案
"let g:molokai_original = 1
"let g:rehash256 = 1
"Tomorrow-Night-Blue
"Tomorrow-Night-Bright
"Tomorrow-Night-Eighties
"Tomorrow-Night
"Tomorrow
"molokai
"solarized

colorscheme onedark
let g:lightline = {
  \ 'colorscheme': 'onedark',
  \ }

" 去掉输入错误的提示声音
set noeb
" 启动的时候不显示那个援助索马里儿童的提示
set shortmess=atI
" 关闭菜单的ALT
set winaltkeys=no
" 设定默认解码
set fenc=utf-8
set fencs=utf-8,usc-bom,euc-jp,gb18030,gbk,gb2312,cp936
" 不要使用vi的键盘模式，而是vim自己的
set nocompatible
" history文件中需要记录的行数
set history=100
" 与windows共享剪贴板
set clipboard+=unnamed
" 状态行颜色
"highlight StatusLine guifg=SlateBlue guibg=Yellow
"highlight StatusLineNC guifg=Gray guibg=White
" 取消自动注释
set formatoptions=ql
set paste
" 不要备份文件（根据自己需要取舍）
set backup
set backupdir=~/.vimbak
" 不要生成swap文件，当buffer被丢弃的时候隐藏它
setlocal noswapfile
set bufhidden=hide
" 字符间插入的像素行数目
set linespace=0
" 增强模式中的命令行自动完成操作
set wildmenu
" 在状态行上显示光标所在位置的行号和列号
set ruler
set rulerformat=%20(%2*%<%f%=\ %m%r\ %3l\ %c\ %p%%%)
" 命令行（在状态行下）的高度，默认为1，这里是2
set cmdheight=2
" 使回格键（backspace）正常处理indent, eol, start等
set backspace=2
" 允许backspace和光标键跨越行边界
set whichwrap+=<,>,h,l
" 可以在buffer的任何地方使用鼠标（类似office中在工作区双击鼠标定位）
set mouse=a
set selection=exclusive
set selectmode=mouse,key
" 通过使用: commands命令，告诉我们文件的哪一行被改变过
set report=0

" 不让vim发出讨厌的滴滴声
set noerrorbells

" 在被分割的窗口间显示空白，便于阅读
set fillchars=vert:\ ,stl:\ ,stlnc:\

"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" Search
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" 高亮显示匹配的括号
set showmatch
" 匹配括号高亮的时间（单位是十分之一秒）
set matchtime=5
" 在搜索的时候忽略大小写
set ignorecase
" highligth the search word
set hlsearch
" 在搜索时，输入的词句的逐字符高亮（类似firefox的搜索）
set incsearch
" 输入:set list命令是应该显示些啥？
set listchars=tab:\|\ ,trail:.,extends:>,precedes:<,eol:$
" 光标移动到buffer的顶部和底部时保持3行距离
set scrolloff=3
" 不要闪烁
set novisualbell
" 我的状态行显示的内容（包括文件类型和解码）
set statusline=%F%m%r%h%w\ [FORMAT=%{&ff}]\ [TYPE=%Y]\ [POS=%l,%v][%p%%]\ %{strftime(\"%d/%m/%y\ -\ %H:%M\")}
" 总是显示状态行
set laststatus=2

"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" Text
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" 自动格式化
set formatoptions=tcrqn
" 继承前一行的缩进方式，特别适用于多行注释
set noautoindent
" 为C程序提供自动缩进
set nosmartindent
" 使用C样式的缩进
set cindent
" 制表符为4
set tabstop=4
" 统一缩进为4
set softtabstop=4
set shiftwidth=4
" 不要用空格代替制表符
set expandtab
" 换行
set wrap
" 在行和段开始处使用制表符
set smarttab
" 用空格键来开关折叠
set foldenable
"set foldmethod=marker
" show line number
set number

" quickfix
" set switchbuf=newtab

"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" Session
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
set sessionoptions=blank,buffers,sesdir,folds,help,options,tabpages,winsize

"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" NERDTree
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
noremap <F7> :NERDTreeToggle<CR>
" autocmd BufReadPost *.go NERDTree

" ============================================================================
" Go
" ============================================================================

" quickfix
let g:go_list_type = "locationlist"

" highlight
let g:go_highlight_types = 1
let g:go_highlight_functions = 1
let g:go_highlight_methods = 1
let g:go_highlight_structs = 1

" let g:go_highlight_fields = 1
let g:go_highlight_operators = 1
let g:go_highlight_extra_types = 1 
let g:go_highlight_build_constraints = 1
let g:go_highlight_generate_tags = 1


"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" gotags
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
let g:tagbar_width = 45
let g:tagbar_autoclose = 0
let g:tagbar_autofocus = 0
let g:tagbar_show_linenumbers = 0
let g:tagbar_show_visibility = 1
let g:tagbar_autoshowtag = 1

let g:tagbar_type_go = {
            \ 'ctagstype' : 'go',
            \ 'kinds'     : [
            \ 'p:package',
            \ 'i:imports:1',
            \ 'c:constants',
            \ 'v:variables',
            \ 't:types',
            \ 'n:interfaces',
            \ 'w:fields',
            \ 'e:embedded',
            \ 'm:methods',
            \ 'r:constructor',
            \ 'f:functions'
            \ ],
            \ 'sro' : '.',
            \ 'kind2scope' : {
            \ 't' : 'ctype',
            \ 'n' : 'ntype'
            \ },
            \ 'scope2kind' : {
            \ 'ctype' : 't',
            \ 'ntype' : 'n'
            \ },
            \ 'ctagsbin'  : '/Users/terrysmao/go/bin/gotags',
            \ 'ctagsargs' : '-sort -silent'
            \ }

noremap <F8> :TagbarToggle<CR>

""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" godef
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
 let g:go_def_mode = 'godef'
 
 " open the definition in a new tab
 let g:godef_split=0
 
 " want jumps to the same file to move your current cursor instead of splitting
 let g:godef_same_file_in_same_window=1
 
 
 """""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
 " lint
 """""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
 let g:go_metalinter_enabled = ['vet', 'golint', 'errcheck']
 let g:go_metalinter_autosave = 1
 let g:go_metalinter_autosave_enabled = ['vet', 'golint']
 let g:go_metalinter_deadline = "5s"
 
 
 """""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
 " goimports
 """""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
 let g:go_fmt_command = "goimports"
 let g:go_fmt_autosave = 1
 let g:go_fmt_fail_silently = 1
 
 """""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
 " identifier
 """""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
 " let g:go_auto_type_info = 1
 " let g:go_auto_sameids = 1
 " set updatetime=100
 " autocmd FileType go nmap <Leader>i <Plug>(go-info)
 ```