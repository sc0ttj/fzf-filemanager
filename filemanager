#!/bin/bash

# a simple terminal based file browser, using fzf and exa.
#
# Browse through folders and open files in $EDITOR.. This file browser can be
# loaded using the `Alt-i` key binding (see ~/.bash/keybindings.bash)


# @TODO fix image previews: see https://github.com/Ckath/fuf
#
# @TODO fix broken navigation when spaces exist in dir names and file names
#
# @TODO add help panels to each mode, showing key bindings
#
# @TODO add only "relevant" cmds to history (in func create_shellprompt_history)
# - grep selection list, check extensions and mime types
# - work out "group" from mime type (audio, video, text, torrents, archives, etc)
# - get commands matching the given group from a list of cool one liners
#
# @TODO clean up end of script.. files are opened by "group", "extension" or "mime-type"
#
# @TODO fix & finish previews of non-text files, using something other than bat
#
# @TODO fix "shellprompt" mode: support commands like these: `mv {}`

test -d "$1" && builtin cd "$1"

mkdir -p /tmp/fzf
chmod 777 /tmp/fzf

function print_filemanager_header {
  header_top="  ${1:-$PWD}"
  echo "$header_top"
  [ "$PWD" = "/" ] && echo "/" || echo " .."
}

# get the "main" list of dirs, using exa,
# passed to fzf in "filemanager" mode
function list_dirs {
  exa \
    $showhidden \
    $longfilemanager \
    --icons \
    --classify \
    --git \
    --git-ignore \
    --ignore-glob '.git|.cache|buffers' \
    --group-directories-first \
    --colour=always \
    --level 1 "${1:-$PWD}" | sed "s/\*$//g"
}

# Alt-C triggers a new fzf window, uses this list, gets
# lots of dirs, 8 levels deep, using fd, or 2 levels
# deep using exa
function list_dirs_for_fzcd {
  local dir_list=""
  if [ "$(which fd)" != "" ];then
    dir_list=$(fd \
      --max-results 1000 --type d --absolute-path --max-depth 8 \
      --hidden --exclude "__pycache__*" --exclude "*.default*" \
      --exclude ".git/*" --exclude ".mednafen/b*" --exclude "*OpenWith*")
  else
    dir_list=$(exa \
      -a --git-ignore \
      --ignore-glob '*mednafen*|*.cache*|*lock*|*.git*|*OpenWith*|*__pycache__*' \
      --oneline -D -R --level=2 \
    | grep -v "^$" \
    | sed -e 's/:$//g' -e 's/$/\//g' -e 's/^\.\///g' \
    | sort -u \
    | while read line; do
        test -d "$line" && echo "$line"
      done)
  fi
  [ "$dir_list" != "" ] && echo "$dir_list"
}
export -f list_dirs_for_fzcd

# get all files in the current dir, one per line
function ls_pwd_contents_one_per_line {
  ls -1 --group-directories-first --classify "${1:-$PWD}"
}

# give fzf a list of dirs (filemanager) or commands (shellprompt)
function get_list_for_fzf {
  if [ ! -f /tmp/fzf/shellprompt ];then
    print_filemanager_header
    echo "$(list_dirs)"
  else
    # print list of commands to search, instead of a list of dirs
    echo '
xmessage                        # popup message
chmod -x --                     # make non exec
chmod +x --                     # make exec
chmod -x                        # make non exec
chmod -x                        # make non exec
$EDITOR                         # open in text editor
extract                         # extract archive
file -m -b                      # get file mime type
htop                            # monitor system processes and programs
mpv --                          # play file
mpv -fs --                      # play file fullscreen
mplayer --                      # play file
mplayer -fs --                  # play file fullscreen
pkg install                     # install the given package file(s)
pkg uninstall                   # uninstall the given package file(s)
pkg c                           # list contents of package(s)
pkg ps                          # get detailed package status
pkg PS                          # get detailed package status
pkg dir2pet                     # create a PET package from a directory
pkg dir2sfs                     # create an SFS package from a directory
rmdir                           # remove directory
rm                              # remove file
stat -f                         # get some details of file(s)
tail -f                         # show end of file(s)
w3m -dump                       # open in console browser as plain text
'
    # append commands from $HOME/bin to the list
    [ -d $HOME/bin ] && ls -1 $HOME/bin
    ls /usr/local/bin/ /usr/bin /usr/sbin/ /bin /sbin | sort -u
  fi
}

function strip_icons_from_fzf_output {
  rev \
    | cut -f1 -d' ' \
    | rev \
    | sed  \
      -e 's/\[[0-9];[0-9][0-9]m//g' \
      -e 's/\[[0-9];[0-9];[0-9][0-9]m//g' \
      -e 's/\[0m//g' \
      -e 's///g'
}

# remember user input ("query") or not:
# - dont remember it if query matches nothing in $PWD (causes
#   infinite loop)
# - enable the relevant --query option if a line in query
#   did match something in $PWD
function set_remembered_query_or_not {
  ls -1 "${PWD}" | grep -m1 -q "${query_text}" && usequery=true
  [ "$usequery" = true ] \
    && query_text="$([ -f /tmp/fzf/querytext ] && cat /tmp/fzf/querytext)" \
    && [ "$query_text" != "" ] && query="--query=${query_text// /}"
}

# if filmanager long view, search column 8 onwards,
# if filemanager normal view, search column 2 onwards
# if shellprompt, search column one onwards
function get_column_to_search {
  local nth=2
  if [ ! -f /tmp/fzf/shellprompt ] && [ -f /tmp/fzf/longfilemanager ];then
    nth='8..-1' # long detailed list view
  else
    nth='2..-1' # short list view
  fi
  [ -f /tmp/fzf/shellprompt ] && nth='..'
  echo -n $nth
}

# ------- end of functions ---------


# disable the "shellprompt" mode, always load in
# "file manager" mode first
rm /tmp/fzf/shellprompt 2>/dev/null

# <--~~ start the main program loop ~~-->
# pass contents of $PWD to fzf choose a dir to cd and reload, choose
# a file to open it, hit ! to toggle to shellprompt mode.
while :; do

  # each times filemanager reloads, we want to reset some stuff
  showhidden=''
  header_text="? for help"
#  header_text="↑,↓,←,→,⏎,],[   ! for shell   tab/shft-tab   ctrl-h/p/l/q"
  #header_text="controls:  move ↑,↓,←,→,⏎   move preview ],[   shell !   help ?  "
  header_text="${header_text} "

  prompt_text="Search: "
  query=""
  query_text="$(cat /tmp/fzf/querytext 2>/dev/null)"
  usequery=false
  fullscreen=''
  hidepreview=''
  longfilemanager=''
  disablesearch=''
  shellprompt_options=''
  history_file=''
  # Show hidden files or not
  [ -f /tmp/fzf/showhidden ] && showhidden='-a'
  [ -f /tmp/fzf/longfilemanager ] && longfilemanager='--long'
  # Enable/disable previe wpanel
  [ ! -f /tmp/fzf/showpreview ] && hidepreview=":hidden"
  # Enable/disable fullscreen
  echo "$@" | grep -qE '\-fs |\-fs$' && fullscreen='--no-height'
  set_remembered_query_or_not
  # draw preview panel
  preview_panel="VAL={2}; [ -f {8} -o -d {8} ] && VAL={8}; test -f \$VAL && \
  {
    file -biz \$PWD/\$VAL | grep ^text &>/dev/null && {
      echo \$PWD/\$VAL | xargs bat --color=always --decorations=never;
    }
    file -biz \$PWD/\$VAL | grep ^image &>/dev/null && {
      w3mimg.sh \$PWD/\$VAL
    }
    file -biz \$PWD/\$VAL | grep '/pdf' &>/dev/null && {
      zathura \$PWD/\$VAL
    }
    file -biz \$PWD/\$VAL | grep -E '^video|audio' &>/dev/null && {
      mpv \$PWD/\$VAL
    }
    file -biz \$PWD/\$VAL | grep 'x-tar' &>/dev/null && {
      tar -tvf \$PWD/\$VAL
    }
  }  \
  || \
  { \
    [ \$VAL != \$PWD ] && exa \
      $showhidden \
      --oneline \
      --git-ignore \
      --git \
      --colour=always \
      --icons \
      --group-directories-first \
      --classify \
      --level 1 \
      \$VAL 2>/dev/null
  }
  "

  # draw shellprompt header text
  shellprompt_header_text="
- Type stuff         to filter the commands
- Tab                to auto complete a command
- Up/Down            to cycle through the commands
- Shift-Up/          to cycle through the command history

Current directory: $PWD

Current selection (\"\$@\"):

$(cat /tmp/fzf/selection 2>/dev/null)

Enter a command to run on the current selection: "

  # set the key binding for enter key, used like so
  # --bind="enter:$var"
  enter_binding=accept

  # set key binding for / key
  fslash_binding='--bind /:accept'

  # Enable/disable shell prompt mode:
  # - user can enter commands on the selected items, "$@"
  # - disables the "as you type" result filtering
  # - changes prompt "icon"
  # - changes header text
  if [ -f /tmp/fzf/shellprompt ];then
    header_text="$shellprompt_header_text"
    # text shown before (left of) the user input
    prompt_text="$ "
    hidepreview=":hidden"
    longfilemanager=""
    usequery=false
    # uncomment below to prevent input being a search thing
    #disablesearch="--phony"

    enter_binding='replace-query+execute(echo {n} > /tmp/fzf/selectedline; \
      echo {q} > /tmp/fzf/querytext;)+abort+execute:\
      cat /tmp/fzf/selection | \
        while read selected; \
        do \
          $(echo {} | sed -e "s/#.*//g" -e "s/  //g") "$selected" | IFS=$'\n' fzf --phony --info=hidden --prompt="" --black; \
       done'

    # dont allow / to trigger accept(), as in file browser view,
    # let it be typed out normally
    fslash_binding=''

    # NOTE!
    # I cannot make fzf cannot parse any spaces at all in
    # $shellprompt_options! Solution: put the vars _inside_
    # the execute() calls

    # set the fzf settings we want for the "shellprompt" mode
    shellprompt_options="
      --bind left:backward-char
      --bind right:forward-char
      --bind shift-up:previous-history
      --bind shift-down:next-history
      --bind up:up
      --bind shift-tab:up
      --bind down:down
      --bind tab:replace-query
      --bind ctrl-p:ignore
      --layout=reverse-list
      --query="

    # tell fzf to use that history
    history_file="--history=/tmp/fzf/cmd_hist"
  fi

  # selected is the name of the file(s) or dir(s) we will select using fzf
  #
  # - if its a dir, cd into it, run this loop again
  # - if its a file, process it by file type and mime type
  #
  selected=$(get_list_for_fzf \
    | IFS=$'\n' fzf \
      $query \
      $disablesearch \
      $fullscreen \
      $history_file \
      --filepath-word \
      --select-1 \
      --multi \
      --tabstop=4 \
      --ansi  \
      --no-border \
      --nth=$(get_column_to_search) \
      --no-bold \
      --no-hscroll \
      --border \
      --margin=0% \
      --info=hidden \
      --header="$header_text" \
      --header-lines=0 \
      --prompt="$prompt_text" \
      --preview-window right:82:noborder"$hidepreview" \
      --preview "$preview_panel" \
      --bind '~:execute(xmessage current_line="\"{}\" toggled=\"$(cat {+f})\" \"PWD=$PWD\"" ### <-- replace)' \
      --bind '?:execute(xmessage "HELP MENU")' \
      --bind 'alt-/:execute(xmessage "PWD=$PWD")' \
      --bind 'ctrl-/:execute(xmessage "PWD=$PWD")' \
      --bind 'ctrl-a:select-all' \
      --bind 'change:top' \
      --bind 'pgup:half-page-up' \
      --bind 'pgdn:half-page-down' \
      --bind 'shift-up:half-page-up' \
      --bind 'shift-down:half-page-down' \
      --bind 'shift-left:preview-page-up' \
      --bind 'shift-right:preview-page-down' \
      --bind '[:preview-page-up' \
      --bind ']:preview-page-down' \
      --bind 'left:execute-silent(rm /tmp/fzf/querytext;)+clear-query+clear-selection+unix-line-discard+top+down+reload(builtin cd ..; echo "..")+top+down+accept' \
      --bind 'right:execute-silent(rm /tmp/fzf/querytext;)+accept' \
      --bind '!:execute(\
         cat "{+f}" | rev | cut -f1 -d" " | rev > /tmp/fzf/selection)+execute(
          [ ! -f /tmp/fzf/shellprompt ] \
            && echo {q} > /tmp/fzf/shellprompt \
            || rm /tmp/fzf/shellprompt;
          [ -f /tmp/fzf/shellprompt ] \
          && echo {q} > /tmp/fzf/querytext)+abort' \
      --bind 'alt-c:abort+execute(echo QUIT; list_dirs_for_fzcd | IFS=$"\n" fzf --select-1 | exec xargs filemanager)' \
      --bind 'ctrl-o:accept' \
      --bind 'ctrl-p:toggle-preview+execute(\
          [ ! -f /tmp/fzf/showpreview ] && touch /tmp/fzf/showpreview || rm /tmp/fzf/showpreview; \
        )' \
      --bind 'ctrl-l:clear-selection+clear-query+execute-silent(\
          [ ! -f /tmp/fzf/longfilemanager ] \
            && touch /tmp/fzf/longfilemanager \
            || rm /tmp/fzf/longfilemanager; \
        )+top+accept' \
      --bind 'ctrl-h:clear-selection+execute-silent(\
          [ ! -f /tmp/fzf/showhidden ] \
            && touch /tmp/fzf/showhidden \
            || rm /tmp/fzf/showhidden \
        )+top+accept' \
      --bind 'ctrl-q:clear-selection+execute(echo QUIT)+abort' \
      --bind 'esc:clear-selection' \
      $shellprompt_options \
      --bind "enter:$enter_binding" \
      $fslash_binding \
    | strip_icons_from_fzf_output)


  # phew .. now have a newline separated list of more or more things
  # in $selection, so lets do something with it...

  if [ -d "$selected" ];then
      builtin cd "$selected"
  elif [ "$selected" = "QUIT" ];then
    break
  else
    selected=$(echo "$selected" | tr '\n' ' ')
    # we _finally_ have our selection...
    # now... lets go through the files (or dirs) we have in
    # $selected, and choose what to do with them, based on type

    # if all selected files are of the same type, open them all at once
    file_cmd=''
    files=''
    # for each $f, set the cmd to run based on its mime type,
    # and add it to $files
    for f in $selected
    do
      case "$(file --mime-type -b "$f")" in
        text*|app*text*|app*json|app*csv|app*perl|app*php|app*pyt*|app*ruby|app*script|app*xml)
      	  [ -f "$f" ] \
      	    && file_cmd="${EDITOR:-vi}" \
      	    && files="${files} $f"
      	  ;;
        image*)
      	  [ -f "$f" ] \
      	  && file_cmd="feh -Z -x -F -B black" \
      	  && files="${files} $f"
          ;;
      esac
    done
    # if we found a match, run the cmd against the files,
    # then continue loop (back to get dirs + fzf)
    [ "$file_cmd" != "" ] \
      && [ "$files" != "" ] \
      && $file_cmd $files && continue

    # if still no match, process archive files
    for f in $selected
    do
      # get the full path to $f as $file and its dir as $folder
      function get_filepath {
        file="$f"
        folder=$(dirname "$f" | xargs realpath)
        [ "$folder" = "$PWD" ] \
          && folder="$folder/${file%.*}" \
          && folder="${folder//.tar/}"
        file=$(realpath "$f")
      }
      # mv foo/foo/* to foo/*, if needed
      function archive_cleanup {
        spare_dir=$(echo "$folder/$(basename "$folder")" | sed 's|//|/|g')
        if [ -d "$spare_dir" ] \
        && [ "$spare_dir" != "" ] \
        && [ "$spare_dir" != '/' ]
        then
          mv "$spare_dir/"* "$folder/"
          rmdir "$spare_dir"
        fi
      }

      # try to unpack $f, and continue back to fzf if successful
      (
        # get full paths of archive, empty for now, but we'll use
        # get_filepath to set values from $f
        file=''
        folder=''
        case "$f" in
          *.tar.bz2) get_filepath; tar xvjf "$file" ;;
          *.tar.gz) get_filepath; tar xvzf "$file" ;;
          *.tar.xz) get_filepath; tar xvJf "$file" ;;
          *.lzma) get_filepath; unlzma "$file" ;;
          *.bz2) get_filepath; bunzip2 "$file" ;;
          *.rar) get_filepath; unrar x -ad "$file" ;;
          *.gz) get_filepath; gunzip "$file" ;;
          *.tar) get_filepath; tar xvf "$file" ;;
          *.tbz2) get_filepath; tar xvjf "$file" ;;
          *.tgz) get_filepath; tar xvzf "$file" ;;
          *.zip) get_filepath; unzip "$file" ;;
          *.Z) get_filepath; uncompress "$file" ;;
          *.7z) get_filepath; 7z x "$file" ;;
          *.rar) get_filepath; unrar x "$file" ;;
          *.xz) get_filepath; unxz "$file" ;;
          *.exe) get_filepath; cabextract "$file" ;;
          *.deb) get_filepath; pkg unpack "$file" ;;
          *.rpm) get_filepath; pkg unpack "$file" ;;
          *.pet) get_filepath; pkg unpack "$file" ;;
          *.sfs) get_filepath; unsquashfs "$file" ;;
          *) false ;;
        esac
        # now move name/name/<files> to name/<files>
        archive_cleanup

      ) && continue
    done

    # else, check what we got by mime-type
    for f in $selected
    do
      case "$(file --mime-type -b "$f")" in
        image*)
          feh -Z -x -F -B black "$f"
          ;;
        audio*)
          ffplay -i "$f" -hide_banner -vn -nodisp -fast -autoexit -exitonkeydown \
            || aplay -i "$f" 2>/dev/null
          ;;
        video*)
          mpv -fs -zoom "$f" 2>/dev/null \
            || mplayer -fs -zoom "$f" 2>/dev/null \
            || ffplay -i "$f" -fs -hide_banner -fast -autoexit 2>/dev/null
        ;;
      esac
    done
  fi

done

# if user exited properly, dont keep the search query
rm /tmp/fzf/querytext 2>/dev/null

