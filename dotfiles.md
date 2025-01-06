# How to use?
```shell
git clone git@github.com:Mawwlle/.dotfiles.git
cd .dotfiles
stow .
source ~/.zshrc
```

### Зачем нужны dotfiles?
Мы унифицируем настройку окружения между системами (для базовых программ, например, таких как `.zsh`, `nvim` и прочее). К тому же, можно легко бекапить это. Тут описано, скорее как сохранить это (ну, то есть когда у нас уже есть различные конфигурационные файлы, а мы хотим сделать это через репозиторий .dotfiles)

### Создание репозитория
Создадим репозиторий `.dotfiles` (в моём случае https://github.com/Mawwlle/.dotfiles). А почему бы не сделать это публичным? Я не храню там супер секретов, зато вполне удобно будет настраивать сервера.

### Создадим папку
Создаём папку именно в домашней директории, потому что так будет проще использовать инструменты по типу GNOME Stow
```shell
mkdir ~/.dotfiles
```

### Перенос 
Мы должны полностью мимикрировать структуре домашней папки внутри наших `.dotfiles`, для \*nix это довольно важно.

```shell
mv ~/.zshrc ~/.dotfiles/
mv ~/.config ~/.dotfiles/
```

### Применение .dotfiles
Мы можем создать ссылку на каждый нужный конфигурационный файл
```shell
ln -sf ~/dotfiles/.zshrc ~/.zshrc
ln -sf ~/dotfiles/.config/nvim/nvim.lua ~/.config/nvim/nvim.lua
```
Но это довольно нудно.

Можно же написать скриптес! Но это тоже нудно... Лучше вспомним про [stow](https://www.gnu.org/software/stow/)

> [!info]-
> stow - рекурсивно проходит директории и создаёт `symbolic` ссылки для всех файлов в  целевой директории!
> Чтобы применить это чудо, нужно лишь
> ```shell
> cd ~/.dotfiles/
> stow .
> ls -la ~
> ```

### Нам-ж гит нада!
Нам-ж гит нада! А он создаёт папочку `.git`, зачем же перетирать папку `~/.git`? Для этого добавим в наш `~/.dotfiles` файл `~/.dotfiles/.stow-global-ignore`
Содержимое ([c оф доки](https://www.gnu.org/software/stow/manual/html_node/Types-And-Syntax-Of-Ignore-Lists.html)):
```
RCS
.+,v

CVS
\.\#.+       # CVS conflict files / emacs lock files
\.cvsignore

\.svn
_darcs
\.hg

\.git
\.gitignore
\.gitmodules

.+~          # emacs backup files
\#.*\#       # emacs autosave files

^/README.*
^/LICENSE.*
^/COPYING
```

После этого не забываем обновить ссылочки
```shell
cd ~/.dotfiles/ 
stow .
ls -la ~
```
### Подключаем и делаем наш первый коммит

```shell
cd ~/.dotfiles
git init
git remote add origin git@github.com:Mawwlle/.dotfiles.git
git branch -M master
git add .
git commit -m "Initial commit"
git push -u origin master
```

