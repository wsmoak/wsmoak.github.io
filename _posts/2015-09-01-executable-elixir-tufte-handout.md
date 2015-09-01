---
layout: post
title:  "Executable Elixir in Tufte Handout PDFs"
date:   2015-09-01 12:52:00
tags: elixir tufte rmarkdown pdf
---

Recently I asked the Internet to tell me what the state of the art is these days for plain text to PDF.  Pandoc and ASCIIDoctor and Kramdown they said.

Meanwhile, I found [Tufte Handout][tufte-handout].  I've been a fan of Edward Tufte's work for years and once had the opportunity to attend one of his seminars.

Here is a link to a sample document produced with the Tufte Handout template: <http://rmarkdown.rstudio.com/examples/tufte-handout.pdf>

This is what I want! I set about trying to get it to run. I installed R and RStudio with Homebrew.  It wasn't happy.  I discovered I needed "MacTex" which weighs in at a whopping 2.5GB for the distribution.  Eventually, the download finished and I installed that.  Still no luck with the sample document, so I removed the figures and tables from it. I don't need those anyway. Now it works!

Somewhere in my wanderings I ran across this [article on running Go language chunks in Rmd files][go-rmd].  Interesting!  Could it work with Elixir code?

I don't know R, but it looks fairly straightforward from the code and description.  It's creating a temporary file, executing it, and capturing the output.  I took a shot at modifying it to execute elixir instead of go, and it worked!

Here is the source for a standalone RMarkdown file that executes embedded Elixir code and includes the output of the code in the PDF:

[view source](https://gist.github.com/wsmoak/f5fd090df809e87a13fb)

[download source](/images/2015/09/Example.Rmd)

And here is a section of the resulting PDF:

![Executable Elixir With Output](/images/2015/09/elixir-and-output-in-tufte-handout.png)

[download PDF](/images/2015/09/Example.pdf)

I'm sure someone can make this into a proper plugin or engine or whatever the correct term is for RStudio.  (Or they already have and someone will point this out to me in the comments.)

Now I suppose I have to get started on the *content* of the handout that I wanted this for...

### References

* [Tufte Handout Format][tufte-handout]
* [Running Go Language Chunks in Rmd Files][go-rmd]
* <http://crab.rutgers.edu/~karel/latex/class4/class4.html>
* <https://www.rstudio.com/wp-content/uploads/2015/02/rmarkdown-cheatsheet.pdf>

[go-rmd]: http://www.r-bloggers.com/running-go-language-chunks-in-r-markdown-rmd-files/

[tufte-handout]: http://rmarkdown.rstudio.com/tufte_handout_format.html
