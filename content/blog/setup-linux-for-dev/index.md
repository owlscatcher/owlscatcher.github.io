---
title: "Linux для разработчика, первые настройки"
date: 2022-12-21T10:00:20+04:00
draft: false
tags: ["linux", "dev", "vim", "zsh"]
categories: ["blog"]
---

Настроим свежую машину с Fedora36 для комфортной работы разработчика.

## Установка нужных пакетов

Кому не интересно читать всю заметку, вот список пакетов необходимых для работы. Для остальных же
-- устанавливаем все пакеты и идем разбираться ниже, что для чего и как это настраивать.

Обновляем репозитории:

```bash
sudo dnf update -y
```

Ставим нужные пакеты:

```bash
sudo dnf install -y git tmux tree zsh ranger screen neovim
```

Ставим docker и docker-compose:

```bash
sudo dnf remove docker \
                docker-client \
                docker-client-latest \
                docker-common \
                docker-latest \
                docker-latest-logrotate \
                docker-logrotate \
                docker-selinux \
                docker-engine-selinux \
                docker-engine

sudo dnf -y install dnf-plugins-core

sudo dnf config-manager \
    --add-repo \
    https://download.docker.com/linux/fedora/docker-ce.repo

sudo dnf install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

Всё, что ниже -- опционально, потому что это все можно запускать в Docker, не засоряя вашу машину,
но мне иногда больше нравится играться с CLI ruby или что-то быстренько пробовать с .net вне
докер-образа.

Ставим .NET (.net core 6)

```bash
sudo dnf install dotnet-sdk-6.0
```

Ставим Ruby:

```bash
sudo dnf install ruby

sudo dnf install rubygem-{tk{,-doc},rake,irb,test-unit}
```

## Терминал -- общаемся с машиной текстом

Часто нужно сделать что-то простое, вроде создания новой папки или файла, а GUI для того зачастую
ужасно неудобные, например, в Windows, в их Explorer, нужно ткнуть в свободно место правой кнопкой
мыши, выбрать `Create new folder`, ввести название файла и тд. Пример высосан из пальца, но
смотрите, как быстро я сделаю это в терминале:

```bash
mkdir <new_dir_name>
```

Или, если нам понадобилось добавить много вложенных папок, мы можем использовать ключ -p:

```bash
mkdir -p <dir_1>/<dir_2>/<dir_n>
```

![Терминал, создание папки](terminal_mkdir.webp)

Или, вот уже более полезный пример, мне часто нужно конвертировать .png изображения в .webp, но
я не хочу их конвертировать по одному или как-то еще, я просто хочу сказать компьютеру о своих
намерениях и всё, например, сконвертируй все файлы с расширением .png в .webp, которые ты найдешь
в этой папке, но... я забыл эту команду, там было что-то с .png... Не беда, терминал дружелюбен:

```bash
history | grep .png
```

![Терминал, история ввода](terminal_history.webp)

Нашли нужную, выполним:

```bash
for f in *.png; do cwebp -q 100 -mt $f -o ${f%.*}.webp; done
```

![Терминал, конвертация .png изображений в .webp](terminal_png_converter.webp)

Столько воды для того, что бы убедить вас, что терминал -- ваш друг. Не надо бояться и искать
красивые GUI, чтобы клацать по ним _манипулятором типа мышь_ © [Диджитализируй](https://www.youtube.com/c/%D0%94%D0%B8%D0%B4%D0%B6%D0%B8%D1%82%D0%B0%D0%BB%D0%B8%D0%B7%D0%B8%D1%80%D1%83%D0%B9)

### Oh-my-ZSH

Стандартно у вас установлен [Bash](https://www.gnu.org/software/bash/). Мы же поставим
[ZSH](https://www.zsh.org/) и твикнем его надстройкой [Oh-my-zsh](https://ohmyz.sh/).

Установим Oh-my-ZSH:

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

Oh-my-ZSH предложит выбрать по умолчанию ZSH-шелл. Выбираем. Закрываем терминал и все, все круто.

Если вдруг открывается опять Bash, назначаем руками шеллом по умолчанию для вашего юзера ZSH:

```bash
chsh -s $(which zsh)
```

Проверяем, установился ли по умолчанию у нашего юзера нужный шелл:

```bash
grep $USER /etc/passwd
```

Должны получить это:

```bash
<username>:x:1000:1000:Armstrong:/home/<username>:/usr/bin/zsh
```

#### Настройки Oh-my-ZSH

Файл настроек Oh-my-ZSH лежит в директории `~/.zshrc`, можете отредактировать его любым текстовым
редактором и перечитать (команда `source <dir>/<file_name>`), чтобы применить настройки.ш

Для примера прокинем какой-нибудь удобный alias для терминала:

```bash
echo "alias glog=\"git log --pretty=oneline\"" >> ~/.zshrc \
    source ~/.zshrc
```

## Git, расскажем о себе

[Git](https://git-scm.com/) **должен** уметь использовать любой разработчик. Но не все
озадачиваются чуть-чуть его поднастроить.

Расскажем git наше имя и нашу почту, чтобы условный gitlab или github матчил наш аккаунт с нашими
коммитами, а то бывает, что коммитят будто бы два разных человека.

```bash
git config --global user.name "<Your> <Name>"

git config --global user.email "<your_email>"
```

Чтобы посмотреть настройку, просто добавляем `--get`:

```bash
git config --global --get user.name
```

## NeoVim -- переосмысление VI и VIM

![NeoVim](neovim.webp)

_Тут еще одна шутка о том, что кто-то опять не смог выйти из VIM._

Безусловно можно исопльзовать любой редактор кода, хоть тот же
[VScode](https://code.visualstudio.com/). Все это легко устанавливается на Fedora. Но я хочу
привести несколько аргументов в пользу "неудобного" на первый взгляд любого VIM-подобного редактора:

- Однообразие рабочего пространства. Нам, как разработчикам, приходится работать как на сервере,
  так и на своей рабочей машине. VIM есть на любой linux-машине. Научившись 1 раз работать с ним, мы
  никогда не сядем в лужу, когда нам срочно понадобится твикнуть какой-то конфиг руками прям на
  сервере.
- Скорость. VIM крайне шустрый. Дурацкий пример, но все же: попробуйте загрузить JSON-файл на
  20-30 тысяч строк в VScode или какой-то другой редактор, а потом еще и поиском попробуйте
  воспользоваться.
- VIM очень гибкий. На него, как и на VScode, можно навесить очень много разных плагинов, очень
  точно настроить сам view-port редактора или вовсе переназначить хоткеи. С появлением LSP
  надобность в IDE с их [IntelliSence](https://learn.microsoft.com/en-us/aspnet/web-forms/videos/building-35-applications/intellisense) вообще отпадает.
- Существует много программ с VIM-подобным управлением, например Ranger. Так что знание VIM
  никогда не будет чем-то бесполезным.

Теперь перейдем к настройке. Мы не будем использовать ванильный Vi или VIM, а поставили форк от VIM
который называется [NeoVim](https://neovim.io/).

Создадим файл настроек NeoVim:

```bash
touch ~/.config/nvim/init.vim
```

Предварительно поставим [PlugInstall](https://github.com/junegunn/vim-plug), через него мы будим
устанавливать плагины для NeoVim:

```bash
sh -c 'curl -fLo "${XDG_DATA_HOME:-$HOME/.local/share}"/nvim/site/autoload/plug.vim --create-dirs \
       https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim'
```

И [шрифт](https://github.com/ryanoasis/nerd-fonts#font-installation), который нужен для плагина
[devicons](https://github.com/ryanoasis/vim-devicons):

```bash
mkdir -p ~/.local/share/fonts

cd ~/.local/share/fonts && curl -fLo "Droid Sans Mono for Powerline Nerd Font Complete.otf" https://github.com/ryanoasis/nerd-fonts/raw/HEAD/patched-fonts/DroidSansMono/complete/Droid%20Sans%20Mono%20Nerd%20Font%20Complete.otf
```

Далее предлагаю вставить мой конфиг (который я, кстати, тоже где-то подрезал):

Откроем наш файл конфигурации:

```
nvim ~/.config/nvim/init.vim
```

Вставляем это:

```bash
" We start by installing Vim plugin manager to manage plugins.
" A Lot of customizations doen here.
if empty(glob('~/.vim/autoload/plug.vim'))
  silent !curl -fLo ~/.vim/autoload/plug.vim --create-dirs
    \ https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim
  autocmd VimEnter * PlugInstall --sync | source $MYVIMRC
endif

call plug#begin('~/.vim/plugged')
Plug 'preservim/nerdtree' " Left file-tree pannel
Plug 'ryanoasis/vim-devicons' " Language icon for file
Plug 'vim-airline/vim-airline' " Bottom info pannel
Plug 'flazz/vim-colorschemes'
Plug 'vim-airline/vim-airline-themes'
Plug 'ervandew/supertab'
Plug 'neoclide/coc.nvim', {'branch': 'release'}
Plug 'puremourning/vimspector'
Plug 'prettier/vim-prettier', { 'do': 'npm install' , 'branch' : 'release/1.x' }
Plug 'dense-analysis/ale'
" LSP servers
Plug 'OmniSharp/omnisharp-vim' " LSP server form .net
"Fuzzy Search
Plug 'junegunn/fzf', { 'do': { -> fzf#install() } }
Plug 'junegunn/fzf.vim'
Plug 'valloric/MatchTagAlways'
Plug 'jiangmiao/auto-pairs'
call plug#end()

:map <C-n> :NERDTreeToggle<CR>
:map <C-p> :GFiles<CR>
:map <C-f> :Files<CR>
"collapse json files by themselves.
:map <C-j> :set filetype=json \| :syntax on \| :set foldmethod=syntax
"zo for open
"zc for close

" Auto indent your file.
map <F7> gg=G<C-o><C-o>
" Navigate Tabs
map <C-t><up> :tabr<cr>
map <C-t><down> :tabl<cr>
map <C-t><left> :tabp<cr>
map <C-t><right> :tabn<cr>

" Supprot for different goto definitions for different file types.
autocmd FileType cs nmap <silent> gd :OmniSharpGotoDefinition<CR>
autocmd FileType cs nnoremap <buffer> <Leader>fu :OmniSharpFindUsages<CR>
autocmd FileType cs nnoremap <buffer> <Leader>fi :OmniSharpFindImplementations<CR>
autocmd FileType cs nnoremap <Leader><Space> :OmniSharpGetCodeActions<CR>

autocmd FileType ts nmap <silent> gd :call CocActionAsync('jumpDefinition')<CR>
autocmd FileType html nmap <silent> gd :call CocActionAsync('jumpDefinition')<CR>


" The following commands are contextual, based on the cursor position.
" autocmd FileType cs nnoremap <buffer>
" autocmd FileType cs nnoremap <buffer> <Leader>fs :OmniSharpFindSymbol<CR>
let g:vimspector_enable_mappings = 'HUMAN'
let g:airline_powerline_fonts=1
let g:tmuxline_powerline_separators = 0

" colorscheme Monokai
colorscheme minimalist
let g:airline_theme='minimalist'
set colorcolumn=120
set number
" set relativenumber

" Toggle relative line number
nmap <C-L><C-L> :set norelativenumber<CR>
map  <C-R><C-L> :set relativenumber<CR>

set encoding=utf-8
scriptencoding utf-8
let g:airline#extensions#tmuxline#enabled = 0

" How tab behaves while picking auto complete.
let g:SuperTabMappingForward = '<S-Tab>'
let g:SuperTabMappingBackward = '<Tab>'

:set tabstop=4
:set shiftwidth=4
:set expandtab

set clipboard+=unnamedplus
" Move Swap Directory to something temporary.
set directory^=$HOME/tempswap//

" This directory should exist.
" Always enable preview window on the right with 60% width
let g:fzf_preview_window = 'right:60%'

" vim wiki settings.
set nocompatible
filetype plugin on
syntax on
let g:vimwiki_list = [{'path': '~/vimwiki/',
                      \ 'syntax': 'markdown', 'ext': '.md'}]


if executable('ag')
  " Use Ag over Grep
  set grepprg=ag\ --nogroup\ --nocolor

  " Use ag in CtrlP for listing files. Lightning fast and respects .gitignore
"   let g:ctrlp_user_command = 'ag %s -l --nocolor -g ""'
endif
" ctrl+p when you move to a file this highlights it.
let g:nerdtree_sync_cursorline = 1


" Use K to show documentation in preview window.
nnoremap <silent> K :call <SID>show_documentation()<CR>

function! s:show_documentation()
  if (index(['vim','help'], &filetype) >= 0)
    execute 'h '.expand('<cword>')
  else
    call CocAction('doHover')
  endif
endfunction


" Read gitignore and dont show relevant files in ctrlp.
" let g:ctrlp_user_command = ['.git/', 'git --git-dir=%s/.git ls-files -oc --exclude-standard']
let g:prettier#config#parser = 'babylon'
" hi Normal ctermbg=none
"
" For Coc Action Menu to work.
" Remap for do codeAction of selected region
function! s:cocActionsOpenFromSelected(type) abort
  execute 'CocCommand actions.open ' . a:type
endfunction
xmap <silent> <leader>a :<C-u>execute 'CocCommand actions.open ' . visualmode()<CR>
nmap <silent> <leader>a :<C-u>set operatorfunc=<SID>cocActionsOpenFromSelected<CR>g@

let g:ale_linters_ignore = {
      \   'typescript': ['tslint'],
      \}

let g:ale_linters = {
\ 'cs': ['OmniSharp']
\}

let g:OmniSharp_server_use_net6 = 1
```

Сохраняем и перечитываем файл, для этого используем команды vim. Чтобы ввести команду, нужно нажать
`:` и далее ввести шорткод. В нашем случае вводим `w` от `write`, нажимаем `enter`, затем вводим
команду `so %`, чтобы перечитать файл. На все ошибки пока не обращаем внимание, нажимаем `enter`,
пока они не пропадут.

Установим все плагины, которые мы указали в конфигуракции: `:PlugInstall`.

Установим [LSP-сервер](https://github.com/OmniSharp/Omnisharp-vim) для netcore: `:OmniSharpInstall`

Все, вам VIM готов. Хорошо бы почитать все шорткоды для установленных плагинов, например для
[NerdTree](https://github.com/preservim/nerdtree) открыть / закрыть панель будет `CTRL + n` и тд.

## Ranger -- VIM-подобный файловый менеджер

![Ranger](ranger.webp)

Ranger очень удобный файловый менеджер, который управляется примерно также, как и VIM. Он удобен
тем, что его, как и VIM, очень легко настроить под свои нужны, например мы хотим включить
предпросмотр .pdf файлов, картинок и тд.

### Включаем предпросмотр изображений и .pdf на Wayland

// TODO

## Docker -- исправление проблем

Docker постоянно то не запущен, то вы его не можете запускать... Исправляем это:

Добавляем вашего пользователя в группу Docker:

```bash
usermod -a -G docker $USER
```

Добавляем Docker в автозагрузку:

```bash
sudo systemctl enable docker
```

## Благодарности серому волшебнику

Если текст был полезен и ты не можешь усмирить желание быть благодарным, то вот мои кошелечки (USDT и TRX кошельки одинаковые, да, это не ошибка):

**Tether (TRC-20, USDT):**

```markdown
TYvFYUV3h5HwqfyTxskGQK7nDbUHTcwPn2
```

**Tron (TRX):**

```markdown
TYvFYUV3h5HwqfyTxskGQK7nDbUHTcwPn2
```

**Monero (XMR):**

```markdown
4AbxbT9vrNQTUDCQEPwVLYZq2zTEYzNr9ZzTLaq9YcwVfdxwkWjZ6FsewuXVDXPk7x2rE6FZACmLePPgJEcY4rm1GSHkwTZ
```
