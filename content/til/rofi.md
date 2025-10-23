---
title: "Rofi and its powerful scripts"
date: 2024-05-14
description: ""
tags: ["tech"]
---

{{< lead >}}
When you realize necessity is the mother of invention
{{< /lead >}}

## Help with Sxhkd

I kept on forgetting the keybindings of my [bspwm](https://github.com/baskerville/bspwm) + [sxhkd](https://github.com/baskerville/sxhkd) workflow setup, initially. The obvious solution was to setup a help menu which will help me remember the bindings whenever I needed them. That is when I went on a internet voyage to find such a thing.
To my relief I found the perfect [blog](https://my-take-on.tech/2020/07/03/some-tricks-for-sxhkd-and-bspwm/) describing this and many cool features we can achieve in the setup when combined with [rofi](https://github.com/davatorium/rofi).

`sxhkd-help`
```shell
#!/bin/bash

awk '/^[a-z]/ && last {print "<small>",$0,"\t",last,"</small>"} {last=""} /^#/{last=$0}' ~/.config/sxhkd/sxhkdrc |
    column -t -s $'\t' |
    rofi -dmenu -i -markup-rows -no-show-icons -width 1000 -lines 15 -yoffset 40
```

And then just add an appropriate key binding in a the `sxhkdrc` file

```shell
# help menu
super + slash
    ~/.config/sxhkd/sxhkd-help
```

## Clipboard

The same blog describes a method to setup a clipboard using rofi, sxhkd and [clipmenu](https://github.com/cdown/clipmenu) in the following manner.

`sxhkdrc`
```shell
# clipboard
alt + v
    CM_LAUNCHER=rofi clipmenu \
        -location 1 \
        -m -3 \
        -no-show-icons \
        -theme-str '* \{ font: 10px; \}' \
        -theme-str 'listview \{ spacing: 0; \}' \
        -theme-str 'window \{ width: 20em; \}'
```

The result is a very concise clipboard showing the recent clips.

---
