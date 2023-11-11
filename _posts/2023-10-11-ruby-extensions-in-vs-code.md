---
layout: post
title:  "Ruby Extensions in VS Code"
date:   2023-10-11 11:59:00
tags: ruby vscode
---

Back when I switched to VS Code I followed some advice and installed the Ruby extension by Peng Lv. 
Some time more recently that extension was deprecated with a note to install the Ruby LSP extension instead, so I did that. 
In news that I did not realize was related, F12 stopped working to jump to method definitions.  That is the ONE thing I need my IDE to do.  
Puzzled, but busy with other things, I just dealt with it by searching within files.  I know the codebase well enough that I can guess where things live.  
But last week git blame in the bottom bar also stopped working, so I spent a few hours figuring out what was going on with VS Code.  
It turns out that Shopify's Ruby extension does not HAVE support for jumping to method definitions yet!  No wonder it doesn't work. I have subscribed to [issue #899][issue-899] for updates.
I uninstalled that and switched to [Solargraph][solargraph].  I do not know what else I have lost or gained, but F12 works again.

[vs-code-ruby]: https://code.visualstudio.com/docs/languages/ruby
[issue-899]: https://github.com/Shopify/ruby-lsp/issues/899
[solargraph]: https://marketplace.visualstudio.com/items?itemName=castwide.solargraph

Copyright 2023 Wendy Smoak - This post first appeared on [{{ site.url }}][site-url] and is [CC BY-NC][cc-by-nc] licensed.
