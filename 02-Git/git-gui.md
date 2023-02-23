# Comandos personalizados

## Stash

I saw a recommendation here to add the Stash commands as a menu to "Git GUI" in your `$HOME\.gitconfig`.

    [guitool "Stash/show"]
    cmd = git stash show -p
    [guitool "Stash/list"]
      cmd = git stash list
    [guitool "Stash/pop"]
      cmd = git stash pop
    [guitool "Stash/drop"]
      cmd = git stash drop
      confirm = yes

I also added one more command to do the stashing (to use, you must first stage--but not commit--the files you wish stashed in order for Git to perform the required "add"):

    [guitool "Stash"]
      cmd = git stash

(Note that I chose to have "stash" not appear as a submenu, but you could do so.)

You can also add commands through Git GUI itself via Tools->Add.

Stashing can be helpful if you just want to put the files out of mind quickly without bothering to think of a branch name.