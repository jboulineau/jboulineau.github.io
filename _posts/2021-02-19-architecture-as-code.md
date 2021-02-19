---
layout: post
title: "Architecture as Code: Managing Documentation"
date: 2021-02-19
categories: "Architecture"
excerpt: "I've been thinking about using a git repo for managing my architecture diagrams and documents and I finally figured out something worth trying."
---

For the past 18 months or so my developer/architect/leadership role mix has shifted heavily to architecture/leadership and that has lead to a significant increase in my need to manage documentation. Of course, I've been doing architecture for many years now and I've developed various ways for keeping it all straight, but it always felt clumsy. For instance, I do very much like Lucidchart for diagrams, but I do run into impedance in some areas:

- Not everyone with whom you need to collaborate has Lucidchart access.
- Lucidchart diagrams often need to be be exported to other formats to be shared or included in documents.
- Vendor lock-in. My company was acquired and they do not use Lucidchart, which means I'm going to have to convert to something else, anyway.
- It is too easy to accidentally grant someone edit permissions.
- Versioning is manual (named versions) or much too granular.
- Reviewing changes between versions isn't easy.

I've set out on a number of occasions to look for a better way, but I've never found a tool chain that made sense. Here's the tool chain that seems to be working:

- [diagrams.net](https://diagrams.net) for diagraming
- Markdown + [pandoc](https://pandoc.org) for documentation
- git (specifically, GitLab) for version control and collaboration
- Visual Studio Code
- [PlantUML](https://plantuml.org) (maybe)

## diagrams.net

[Diagrams.net](https://diagrams.net) allows persisting your diagrams as local files. I use the desktop app since it makes more sense in this model. The primary magic is that you can use a `.png` or `.svg` compatible format to store your diagrams, which means there's no need to export the diagram to other formats to share with anyone who doesn't use the same tool. And if you need to share the diagram between other tools (like lucidchart), simply convert the file to `.drawio` and it becomes portable in an editable format. I've found the `.png` format to work best, even though it suffers in visual quality somewhat. `.svg` compatibility is tenuous in Word (for instance) and non-technical users don't grok them. And (at least in GitLab) you can't compare changes between commits.

Being natively stored in an image format also allows them to be linked by reference into Markdown files so diagram changes are automagically reflected in my documents. This saves a lot of annoyance when I'm in the later stages of documentation and I'm making minor tweaks to diagrams to reconcile with my documentation.

This also allows for easy sharing. If I ever have to just drop a diagram as a one-off request to someone, I can just send the file, rather than export into something consumable. This is extra helpful for getting something to a VP who is putting together a Power Point and needs a visual.

## Markdown + pandoc

Everyone loves Markdown, right? It's everywhere, presentable by the major documentation systems, convertable to everything, and very easy to do (almost) all the formatting you want to do. But, sometimes you have to share documents in other formats. For instance, for 'official' recognition I have to submit some documents in Word format. Luckily, there is [pandoc](https://pandoc.org) and with little effort I can create the Word doc from the Markdown. You can even use a reference file to make the output conform to a required template. It's not perfect, but it works well enough with some tweaking.

Other than the simple pleasure of just editing text rather than some proprietary format with all the button clicking, linking out to the diagram files rather than embedding them into documents has been the big win.

## git

It's ubiquitous and it aligns with how the teams I support work. For collaboration and versioning it can't be beat, since almost everyone understands the flow now.  It's pretty cool to be able to compare changes to diagrams when working together and it's very handy to be able to easily keep drafts of diagrams separate from published diagrams by using branches.

I've settled on a single repo with directories for every product I work on, but do what works for you.

## Visual Studio Code

Like many others, [VS Code](https://code.visualstudio.com/) has become my primary tool for everything. As usual, there are extensions that make all of the above (nearly) seamless.

- [Draw.io integration](https://marketplace.visualstudio.com/items?itemName=hediet.vscode-drawio)
- [Mermaid for draw.io](https://marketplace.visualstudio.com/items?itemName=nopeslide.vscode-drawio-plugin-mermaid)
- [SVG Previewer](https://marketplace.visualstudio.com/items?itemName=vitaliymaz.vscode-svg-previewer)
- [vscode-pandoc](https://marketplace.visualstudio.com/items?itemName=DougFinke.vscode-pandoc)
- [Markdown All in One](https://marketplace.visualstudio.com/items?itemName=yzhang.markdown-all-in-one)
- [Markdown Lint](https://marketplace.visualstudio.com/items?itemName=DavidAnson.vscode-markdownlint)
- [Markdown Preview Enhanced](https://marketplace.visualstudio.com/items?itemName=shd101wyy.markdown-preview-enhanced)

Being able to edit diagrams and documents, as well as maintain the repo, in the same tool is very nice.

## PlantUML

I left this one last because I'm very much on the fence. I really, really like the idea of diagrams-as-code, and [PlantUML](https://plantuml.com) seems to be the most mature. At this point I use it anytime I need to do a UML diagram (but it can sort of do other diagrams, too).

I'm keeping my eye on this space as [Mermaid](https://mermaid-js.github.io/) seems to have more attention and is more flexible (and prettier). Mermaid is supported in `diagrams.net`. There's also [Diagrams](https://diagrams.mingrammer.com/), which looks really cool. I'll play with it next.

If you are keen on using PlantUML, here are the extensions I've found useful:

- [PlantUML](https://marketplace.visualstudio.com/items?itemName=jebbs.plantuml)
- [PlantUML Syntax Highlighting](https://marketplace.visualstudio.com/items?itemName=Yog.yog-plantuml-highlight)

If you do use PlantUML with these extensions, you'll need to structure your repo in a way that makes the build output create in a sane way, as output follows the structure of the directory containing your source files. This pretty tightly constrains how you set up your repo directory structure.

## What I don't like

Tools like Confluence are definitely nicer to view documentation in than Git(Lab/Hub). I'm hoping to play with the git plugin for Confluence to see how it all meshes together.

Searching documents is a problem. Sure, many people I work with can just pull the repo and `grep` for what they want, but not everyone can. However, those people tend to just ask me rather than go looking. I'm toying with the idea of creating a Jekyll site since it works nicely with Markdown files and can enable search. For now, I think keeping the repo organized in a sane way is good enough.

Lucidchart diagrams look better and work better (but not dramatically so).
