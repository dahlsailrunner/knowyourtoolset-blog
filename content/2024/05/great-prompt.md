---
title: "A Great Prompt Experience" 
date: 2024-05-28T05:51:55-05:00 
summary: "A useful setup for Windows Terminal or other shell that enables a great and fun prompt experience." 
codeMaxLines: 30 .
codeLineNumbers: true 
thumbnail: "/images/oh-my-posh.png" 
shareImage: "/images/oh-my-posh.png" 
toc: true
tags:
  - Oh-My-Posh  
  - Nerd Fonts
  - Icons
---

Creating great prompt experience for yourself - one that works for YOU -
can help with productivity. I've spent some time setting one up for me,
and some people have asked me about it. If these things work for you,
great and feel free to use the setup as-is. But I definitely encourage
you to make sure that whatever setup you use works for ***you***.

Here's a screenshot of mine (I just a `dir` in the top pane):
![ ](/images/oh-my-posh-dir.png)

## Windows - Use Windows Terminal

If you're on Windows (I am), I recommend using Windows Terminal for all
your terminal needs (versus Command Prompt).  Lots of features and
enhancements are available there that aren't in the Command Prompt.

If you don't already have it, you can install it from the Windows Store,
or use a package installer:

```bash
winget install Microsoft.WindowsTerminal
```

Once it's installed, make sure it's your default terminal application -
you can access settings with `Ctrl+,` - and there is an option for
**Default terminal application" that you can set to Windows Terminal.

## Install [Oh-My-Posh](https://ohmyposh.dev/docs/installation/windows)

The link above is for Windows machines, but there is a [corresponding
link for MacOS](https://ohmyposh.dev/docs/installation/macos), too.

Once you've done this (you may need to relaunch your terminal window
for `PATH` changes to have been updated), install a [Nerd Font](https://www.nerdfonts.com/).
The easiest way is:

```bash
oh-my-posh font install --user
```

The best / easiest font to use here is probably the `Meslo` one -- feel free to
choose a different one but be aware that some special characters / icons may
be different for you.

Once you've installed the font, you should configure the terminal to
**use that font as your default**.  That setting can be found in the
Profiles->Defaults section under **Appearance** as shown below:
![ ](/images/font-setting.png)

You also need to **enable** Oh-My-Posh in your powershell profile.

If you type `notepad $profile` in a prompt you can add this line
to the file:

```powershell
oh-my-posh init pwsh | Invoke-Expression
```

{{% notice note "Create Powershell Profile" %}}
The following command will create a powershell profile if one doesn't already exist:

```powershell
if (!(Test-Path -Path $profile)) { New-Item -ItemType File -Path $profile -Force }
```

{{% /notice %}}

There are additional notes in the [official docs](https://ohmyposh.dev/docs/installation/prompt),
but you need to reload your profile for the new shell settings to take effect.

Either restart the terminal, or type `. $profile` in the prompt which should
also reload it.  You can also open a new tab / window to do the same.

At this point you should ***already*** have a better looking prompt / terminal
that can be further configured to your needs!

## Choose a Theme

There may be a [theme](https://ohmyposh.dev/docs/themes) available that
suits your needs -- or one that you like how it looks.  

If you see one in the above link you just need to modify the
`oh-my-posh init` line in your `$profile` to include some config:

```powershell
oh-my-posh init pwsh --config 'C:/Users/Posh/jandedobbeleer.omp.json' | Invoke-Expression
```

The only new content in the above line is (replace `jandedobbeleer` with the theme
you want to use):

```bash
--config 'C:/Users/Posh/jandedobbeleer.omp.json'
```

## Customize Your "Segments"

[**Segments**](https://ohmyposh.dev/docs) are a block of content that can be included
in your prompt -- and there are a ***lot of options.***

Using segments means adding a block of `json` to your config for Oh My Posh.  If you
have a theme that you're already using, make a copy of it and give it a new name.

Then just add the segments you want!  I've got an example of my custom theme
at the bottom of this section.

![ ](/images/segments.png)

The segments that I've got included are:

* [Angular](https://ohmyposh.dev/docs/segments/angular) - shows the Angular version when you're in a folder with an Angular project
* [Node](https://ohmyposh.dev/docs/segments/node) - shows the current Node version when you're in a folder with a `package.json` file
* [.NET](https://ohmyposh.dev/docs/segments/dotnet) - shows the .NET version in a folder with a .NET project or solution
* [Git](https://ohmyposh.dev/docs/segments/git) - Shows current branch and change count
* [Path](https://ohmyposh.dev/docs/segments/path) - Shows current directory (folder name OR full path)

The point with these segments is that they can give you some visibility
into **things that matter to you** - in my case I care about version numbers to
make sure that I don't fall too far behind on things.

One thing to note is that the segments generally show up *when they make sense.*

Other segments that may be of interest to you:

* [AWS](https://ohmyposh.dev/docs/segments/aws)
* [Azure](https://ohmyposh.dev/docs/segments/az)
* [GCP](https://ohmyposh.dev/docs/segments/gcp)
* [Java](https://ohmyposh.dev/docs/segments/java)
* [Kubectl](https://ohmyposh.dev/docs/segments/kubectl)
* [PHP](https://ohmyposh.dev/docs/segments/php)
* [Python](https://ohmyposh.dev/docs/segments/python)
* [React](https://ohmyposh.dev/docs/segments/react)
* [Ruby](https://ohmyposh.dev/docs/segments/ruby)

Here's my custom theme:

```json
{
  "$schema": "https://raw.githubusercontent.com/JanDeDobbeleer/oh-my-posh/main/themes/schema.json",
  "blocks": [
    {
      "alignment": "left",
      "segments": [
        {
          "background": "#6272a4",
          "foreground": "#ffffff",
          "leading_diamond": "\ue0b6",
          "trailing_diamond": "\ue0b0",
          "style": "diamond",
          "type": "os"
        },
  {
    "type": "angular",
    "style": "powerline",
    "powerline_symbol": "",
    "foreground": "#F8F8F2",
    "background": "#FF5555",
    "template": "  {{ .Major}}.{{ .Minor }} "
  },
  {
    "type": "dotnet",
    "style": "powerline",
    "powerline_symbol": "",
    "foreground": "#44475A",
    "background": "#F1FA8C",
    "template": "  {{ .Major }} "
  },
  {
    "type": "node",
    "style": "powerline",
    "powerline_symbol": "",
    "foreground": "#44475A",
    "background": "#8BE9FD",
    "template": "  {{ .Major }} "
  },  
        {
          "background": "#bd93f9",
          "foreground": "#ffffff",
          "powerline_symbol": "\ue0b0",
          "style": "powerline",
          "properties": {
            "style": "folder"
          },
          "template": " \uF07B {{ .Path }}",
          "type": "path"
        },
        {
          "background": "#ffb86c",
          "foreground": "#ffffff",
          "powerline_symbol": "\ue0b0",
          "properties": {
            "branch_icon": "",
            "fetch_stash_count": true,
            "fetch_status": false,
            "fetch_upstream_icon": true
          },
          "style": "powerline",
          "template": " \u279c ({{ .UpstreamIcon }}{{ .HEAD }}{{ if gt .StashCount 0 }} \uf692 {{ .StashCount }}{{ end }}) ",
          "type": "git"
        }//,
        //{
        //  "background": "#ff79c6",
        //  "foreground": "#ffffff",
        //  "powerline_symbol": "\ue0b0",
        //  "style": "powerline",
        //  "type": "kubectl",
        //  "properties": {
        //    "prefix": " \uF1D1 "
        //  },
        //  "template": " \uFD31 {{.Context}}{{if .Namespace}} :: {{.Namespace}}{{end}} "
        //}
      ],
      "type": "prompt"
    }
  ],
  "final_space": true,
  "version": 2
}
```

## Aliases / Shortcuts

One other thing that may be useful to you is to create aliases for common commands you
use.  You can set up whatever you want - just give some thought to what works best
for you.

I've got the following set up:

```powershell
New-Alias npp "C:\Program Files\Notepad++\notepad++.exe"
New-Alias which Get-Command
New-Alias docker podman
```

The above means that I can type `npp $profile` and it will open my powershell
profile using [Notepad++](https://notepad-plus-plus.org/) - which is super handy for me.

I can also use `which` to find where a program in my path lives, and
I'm using [Podman Desktop](https://podman-desktop.io/) as an alternative to
Docker Desktop and the above creates a handy alias.

## Color Schemes (optional)

If you want to change the color scheme of the terminal from the default,
you can go into the terminal settings and find the "Color Schemes" node.

You can explore and try the built in ones and add more on your own. If you
click the `+ Add New` button, you'll have to provide values for 20 different
colors.  

Alternatively, there's an `Open JSON file` in the bottom bar of the terminal
window that will show you the `json` settings file for the terminal.

In the `schemes` array, you can add another item into the array.  Each
item looks like this - with a value for the 20 different colors you need:

```json
{
  "name": "Campbell Powershell",
  "background": "#012456",
  "black": "#0C0C0C",
  "blue": "#0037DA",
  "brightBlack": "#767676",
  "brightBlue": "#3B78FF",
  "brightCyan": "#61D6D6",
  "brightGreen": "#16C60C",
  "brightPurple": "#B4009E",
  "brightRed": "#E74856",
  "brightWhite": "#F2F2F2",
  "brightYellow": "#F9F1A5",
  "cursorColor": "#FFFFFF",
  "cyan": "#3A96DD",
  "foreground": "#CCCCCC",
  "green": "#13A10E",  
  "purple": "#881798",
  "red": "#C50F1F",
  "selectionBackground": "#FFFFFF",
  "white": "#CCCCCC",
  "yellow": "#C19C00"
}
```

Note the `name` property of the scheme (which needs to be unique in the
list of schemes).  Once you have added a scheme and given it a name, you
can choose it as the default color scheme or for specific profiles (like
PowerShell versus Ubuntu or something).

I'm a huge fan of the [Dracula](https://draculatheme.com/windows-terminal)
(and its [PRO variants](https://draculatheme.com/pro)) theme.

## Terminal Icons (optional)

This is simply something that will give you nice icons when listing the contents
of a directory -- so use optionally.

Here is a repo with the details: [https://github.com/devblackops/Terminal-Icons](https://github.com/devblackops/Terminal-Icons)

To get this goodness, you need to install the terminal icons:

```powershell
Install-Module -Name Terminal-Icons -Repository PSGallery
```

Then in your `$profile` file (the same one mentioned above), add this line:

```powershell
Import-Module -Name Terminal-Icons
```
