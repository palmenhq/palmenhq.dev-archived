---
title: Automating My MacOS Setup Process
date: 2021-01-19 20:12:11
tags:
    - ansible
    - automation
    - just-for-fun
---

I recently got a new MacBook, and while setting up the environment can be fun in a way (that "new-and-shiny" feeling is really nice) it's also a bit tedious and time-consuming. Therefore I decided to make a little experiment to automate the setup, using [Ansible](https://www.ansible.com/) (a tool for automating provisioning of environments) and it's [local connection feature](https://docs.ansible.com/ansible/latest/user_guide/playbooks_delegation.html#local-playbooks). Here I'll describe the basics, you can find the full setup at [GitHub](https://github.com/palmenhq/.macos-env).


## About Ansible

Ansible is a tool for provisioning environments, and server's in particular. It can be used to i.e. automate the deployment of your whole AWS account, but I'd recommend looking in to [Terraform](https://www.terraform.io/) instead for that although that's a different topic.

Ansible follows a declarative philosophy - so you declare the state you want your environment to be in with yaml files. When provisioning the environment Ansbile will figure out what changes might be needed in order for the target environment to end up in the desired state. 

On MacOS, you can use [Homebrew](https://brew.sh/) to install Ansible (`brew install ansible`).

## Setup

Normally I keep my dev stuff under `~/dev/<group?>/<repository>` but this I wanted to keep in my home directory. So to begin I created the directory `~/.macos-env`.

## Terminal Tools

### Oh My ZSH

To kick things off I started by setting up "Oh My ZSH" which I use to get a nice smooth terminal experience. I placed this in `terminal.yml` and applied by running `ansible-playbook terminal.yml`


```yaml terminal.yml
- hosts: 127.0.0.1  # for ansible to know it should execute it on this local machine
  connection: local # for ansible to know it should execute it on this local machine
  tasks:
      - name: install oh-my-zsh
        shell: sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)" # this is the actual installation script
        args:
            creates: ~/.oh-my-zsh # Ansible needs a way to know whether the above shell script was executed or not,
                                  # and checking whether this directory exists should be safe enough.

      - name: Set zsh theme
        lineinfile: # replaces the default theme with the awesome theme "sunaku" by replacing every line that starts with `ZSH_THEME=` (which is hopefully only one)
            path: ~/.zshrc
            regexp: '^ZSH_THEME='
            line: 'ZSH_THEME="sunaku"'
```

### tree

I wanted to install the [`tree` command](https://linux.die.net/man/1/tree), most preferably from Homebrew. Ansible has a really nifty [Homebrew module](https://docs.ansible.com/ansible/latest/collections/community/general/homebrew_module.html), which needs to be installed with `ansible-galaxy collection install community.general`. I put this in a special setup script and the readme, just to save me some headaches when I'll come back to this in a year or two. After modifying the file and installing the module you can apply the whole thing with `ansible-playbook terminal.yml`. Of course, you can use this for any formula available on brew.

```yaml terminal.yml
# ...
  tasks:
      # ...

      - name: Add tree
        homebrew:
            name: tree
```

### git settings

Next up I wanted to automate my settings for Git. This one is slightly trickier as it requires dynamic input (my name + email), and I want to keep my name out of source code in general. However, Ansible has this concept of [inventory](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html) that can [hold variables](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#adding-variables-to-inventory). To start, I set up an example inventory, and added the "real" inventory to `.gitignore`. Also, I like to have some things globally ignored by Git (`.DS_Store`, `.idea/*` etc), and hence created a `~/.macos-env/resources/.gitignore-global` file.

```gitignore .gitignore
inventory.yml
```

```yaml inventory.yml
local:
    hosts:
        localhost: # use this for all 127.0.0.1 hosts (connected to the `hosts` property in `terminal.yml`)
    vars:
        git_committer_email: my@email.com
        git_committer_name: Alice Bob

```

Next I built the Git setup:

```yaml terminal.yml
# ...
  tasks:
      # ...

      - name: Check for git user email
        register: git_user_email_present
        shell: git config user.email
        changed_when: git_user_email_present.rc == 1
        ignore_errors: True
      - name: Check for git user name
        register: git_user_name_present
        shell: git config user.name
        changed_when: git_user_name_present.rc == 1
        ignore_errors: True
      - name: Set git user email
        shell: git config --global user.email "{{ git_committer_email }}" # references the variable from `inventory.yml`
        when: git_user_email_present.rc == 1
      - name: Set git user name
        shell: git config --global user.name "{{ git_committer_name }}"  # references the variable from `inventory.yml`
        when: git_user_name_present.rc == 1

      - name: Check for git excludes file
        register: git_user_excludesfile_present
        shell: git config core.excludesFile
        changed_when: git_user_excludesfile_present.rc == 1
        ignore_errors: True
      - name: Set git excludesFile
        shell: git config --global core.excludesFile "~/.macos-env/resources/.gitignore-global"
        when: git_user_excludesfile_present.rc == 1
```

To let Ansible know about `inventory.yml` we need to change the command we apply it with slightly: `ansible-playbook -i inventory.yml terminal.yml`

Here's some weird stuff but let's walk through it. The first thing we want to do is set the Git user's email, but only if it's not already set. Unfortunately I didn't find any good Ansible modules for setting global Git properties, hence this little dance with shell commands. What happens is that we're testing whether the configuration key exists or not (in the step "Check for git user email"). If the shell script we run (`git config user.email`) exits with a non-zero value we know that the config key is _not_ set. We can pick up that result code and use it as a conditional for whether to run the step "Set git user email". Then we repeat that for setting the email, and something similar for the global gitignore file.

I had some ideas on whether to use the global gitconfig file and parse that somehow instead, but using Git's built-in features seemed more sane even though this got a bit clunky in the Ansible world.

### Vim

I wanted to version control my Vim configuration, and hence put more or less the full Vim configuration in the Git repository. After setting up  my `.vimrc` I linked it from `~/.vimrc` by using `source`.

```vim .vimrc
" set the runtime path to include Vundle and initialize
set rtp+=~/.vim/bundle/Vundle.vim
call vundle#begin()
" alternatively, pass a path where Vundle should install plugins
"call vundle#begin('~/some/path/here')

" let Vundle manage Vundle, required
Plugin 'VundleVim/Vundle.vim'

" The following are examples of different formats supported.
" Keep Plugin commands between vundle#begin/end.
" plugin on GitHub repo
Plugin 'tpope/vim-fugitive'
" plugin from http://vim-scripts.org/vim/scripts.html
" Plugin 'L9'
" Git plugin not hosted on GitHub
Plugin 'git://git.wincent.com/command-t.git'
" git repos on your local machine (i.e. when working on your own plugin)
" Plugin 'file:///home/gmarik/path/to/plugin'
" The sparkup vim script is in a subdirectory of this repo called vim.
" Pass the path to set the runtimepath properly.
Plugin 'rstacruz/sparkup', {'rtp': 'vim/'}
" Install L9 and avoid a Naming conflict if you've already installed a
" different version somewhere else.
" Plugin 'ascenator/L9', {'name': 'newL9'}


Plugin 'preservim/nerdtree'
Plugin 'bfrg/vim-cpp-modern'
Plugin 'arcticicestudio/nord-vim'



" All of your Plugins must be added before the following line
call vundle#end()            " required
filetype plugin indent on    " required
" To ignore plugin indent changes, instead use:
"filetype plugin on
"
" Brief help
" :PluginList       - lists configured plugins
" :PluginInstall    - installs plugins; append `!` to update or just :PluginUpdate
" :PluginSearch foo - searches for foo; append `!` to refresh local cache
" :PluginClean      - confirms removal of unused plugins; append `!` to auto-approve removal
"
" see :h vundle for more details or wiki for FAQ
" Put your non-Plugin stuff after this line

:set number
:colorscheme nord
:syntax on
:set listchars=eol:$,tab:>-,trail:~,extends:>,precedes:<
:set list
:command NT NERDTreeToggle
```

```yaml terminal.yml
# ...
  tasks:
      # ...

      - name: Add vimrc
        lineinfile:
            path: ~/.vimrc
            line: source ~/.macos-env/resources/.vimrc # wherever the file is located
```

## Graphical Apps

Instead of downloading all those nifty graphical apps ([Spotify](https://www.spotify.com/), [Spectacle](https://www.spectacleapp.com/), [VSCode](https://code.visualstudio.com/)... yadayada) I decided to automate that too with [Homebrew Cask](https://github.com/Homebrew/homebrew-cask). For that Ansible has a [Homebrew Cask Module](https://docs.ansible.com/ansible/latest/collections/community/general/homebrew_cask_module.html).

Also I decided to put this in a separate Yaml file, just to not have too big files.

```yaml apps.yml
- hosts: 127.0.0.1
  connection: local
  tasks:
      - name: Install Spectacle
        homebrew_cask:
            name: spectacle
            state: present
            greedy: yes # automatically update
      # ...
```

## Conclusion

It's really not that hard to automate the desktop environment setup. I look forward to someone expecting a day or so of setup time for me, when I can complete it in less than an hour with these simple scripts. From now on I can't really run `brew install xyz` directly, but rather I should install it via Ansible. I guess time will tell whether I will remember that habit, and whether it's actually worth the hassle of going through this. If not, at least it made a decent blog post.
