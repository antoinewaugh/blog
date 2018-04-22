
# i3 window manager config
``` 
$ sudo dnf install i3 i3status dmenu i3lock xbacklight feh conky
```

## installing

## basic keyboard bindings

```
$ mod + <enter>       # new terminal horizontally tilled
$ mod + v             # change tiling to vertical
$ mod + h             # change tiling to horizontal
$ mod + 1..9          # move to workspace 1..9
$ mod + shift + 1..9  # move active tile to workspace 1..9
$ mod + shift + arrow # modify the size of a tile within its peers
$ mod + d             # application launcher
$ mod + shift + Q     # close current tile
```


## configuration

```
$ vi ~/.i3/config
```

### key bindings

Add your preferred keybindings, for example:
```

```

### auto-starting applications

```
$ exec myapp            # start myapp upon reboot
$ exec always myapp     # start myapp upon reboot AND reloads of i3 config
```
