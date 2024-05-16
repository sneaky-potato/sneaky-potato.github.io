---
title: "Down the linux rabbit hole"
date: 2024-05-16
description: "do it yourself arch"
tags: ["tech", "linux", "os"]
---

{{< lead >}}
Assembling real linux OS experience
{{< /lead >}}

## One way trip

I had manjaro installed in my main SSD and for the most part is was awesome, until it *wasn't*. I backed up every important config folder and my work directories in my hard drive.

Next thing, I installed Arch Linux in my hard drive (I was sceptical and hesitant) and setup `bspwm` `polybar` `sxhkd` and `rofi` to test things out. There was nothing, no frontend, it felt like a server without pages, can't control brightness, had to setup audio, everything felt like a punch in the face, a blissful punch.

## Big decision

I installed Arch but this time in my SSD, but I took the foolish path of a script kiddie, I used the horrendous arch installer script. And the horror doesn't end here, for some reason I installed Gnome DE.

But then I really wanted to try and setup bspwm and polybar. To try a tiling window manager as my main workflow, it was a dream, I wanted to become a wizard. I posted on reddit to help me with this predicament, particularly removing gnome and installing a window manager. I recieved a comment which simply stated to not use arch installer script, somebody asked why and the godly reply after that read

```text
It can lead to cases where someone doesn't know how to install packages on their own
For example when switching from gnome to bspwm
```

And it hit hard.

## Why Arch

I reinstalled arch but this time manually. I setup all the configs for my bspwm setup and it is amazing. The joy that I got by writing a bash script for invoking a `dunst` notification on pressing volume keys on my keyboard was **phenomenal**.

You can have a look at my dotfiles from [here](https://github.com/sneaky-potato/dotfiles)

![corner](/snap.png)

The reason why arch exists is to customize the linux experience and to present a real do-it-yourself os. Don't use it like windows, kids, be a real man and face the black screen of suffering.

---
