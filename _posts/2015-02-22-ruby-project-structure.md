---
layout: post
title:  "Ruby Project Structure"
date:   2015-02-22 21:16:00
tags: ruby bundler rspec
---

The team I work with is [hiring](https://twitter.com/chargify/status/560954140352720896) and I thought I'd attempt the code challenge they're using for candidates.

The requirements include simulating a popular card game, and reporting statistics on the average number of hands and time it takes to complete a game.

As I pondered how to begin, I realized that I don't really know how a Ruby project is supposed to be structured.  What is the equivalent of "mvn archetype:create ..." from Java and Maven?  Where do you put your code and your tests?  What else is typically in there?

A few ideas from searching...

* <https://www.ruby-forum.com/topic/213637> Typical Ruby (non-rails) project structure
* <http://stackoverflow.com/questions/9549450/how-to-setup-a-basic-ruby-project>
* <http://learnrubythehardway.org/book/ex46.html> A Project Skeleton
* <http://stackoverflow.com/questions/614309/ideal-ruby-project-structure>

I settled on creating a gem by executing...

<pre>
$ bundle gem mygame
      create  mygame/Gemfile
      create  mygame/Rakefile
      create  mygame/LICENSE.txt
      create  mygame/README.md
      create  mygame/.gitignore
      create  mygame/mygame.gemspec
      create  mygame/lib/mygame.rb
      create  mygame/lib/mygame/version.rb
Initializing git repo in /Users/wsmoak/projects/mygame
$
</pre>

Interesting!  My next step was going to be to get this under version control, but it looks like that's already done.

<pre>
$ cd mygame/
$ git status
On branch master

Initial commit

Changes to be committed:
  (use "git rm --cached [file]..." to unstage)

	new file:   .gitignore
	new file:   Gemfile
	new file:   LICENSE.txt
	new file:   README.md
	new file:   Rakefile
	new file:   lib/mygame.rb
	new file:   lib/mygame/version.rb
	new file:   mygame.gemspec

$ 
</pre>

The git repository has been initialized and the files have been added, all that remains is to commit:

<pre>
$ git commit -m "Add initial project structure from 'bundle gem mygame'"
[master (root-commit) d9b1ec0] Add initial project structure from 'bundle gem mygame'
 8 files changed, 104 insertions(+)
 create mode 100644 .gitignore
 create mode 100644 Gemfile
 create mode 100644 LICENSE.txt
 create mode 100644 README.md
 create mode 100644 Rakefile
 create mode 100644 lib/mygame.rb
 create mode 100644 lib/mygame/version.rb
 create mode 100644 mygame.gemspec
$ 
</pre>

It just needs a little cleanup to capitalize MyGame properly and fix the version number so that we do not build a 'real' version over and over locally.  From [prerelease-gems](http://guides.rubygems.org/patterns/#prerelease-gems) it looks like .pre is the convention, so we'll go with that.  (This appears to be _roughly_ the equivalent of the -SNAPSHOT suffix that Maven uses on version numbers in Java projects.)

So that gets me a project structure and a hint about where my project code should go (in the lib directory) but there's nothing here about tests.

I'm pretty sure I want to use RSpec, and a quick search turns up <https://relishapp.com/rspec/rspec-core/v/3-2/docs/command-line> and

<pre>
$ rspec --init
  create   spec/spec_helper.rb
  create   .rspec
$
</pre>

A bit more clicking around leads me to believe I need a spec/mygame_spec.rb file, so I'll add that, with the contents...

<pre>
describe MyGame do

end
</pre>

Now how to run it?  Just running <code>rspec</code> complains

<pre>
$ rspec 
/Users/wsmoak/projects/mygame/spec/mygame_spec.rb:1:in `<top (required)>': uninitialized constant MyGame (NameError)
</pre>

That looks like it needs a <code>require</code> statement somewhere...

This appears to have the best advice: <http://stackoverflow.com/questions/4398262/setup-rspec-to-test-a-gem-not-rails>

AND it looks like I could have used <code>--test=rspec</code> with the <code>bundler gem mygame</code> command.

I generated a new project with that switch and after some fussing around trying to compare them and add the missing bits to my original project, I elected to just delete everything and start over...

<pre>
$ bundler gem --test=rspec mygame
      create  mygame/Gemfile
      create  mygame/Rakefile
      create  mygame/LICENSE.txt
      create  mygame/README.md
      create  mygame/.gitignore
      create  mygame/mygame.gemspec
      create  mygame/lib/mygame.rb
      create  mygame/lib/mygame/version.rb
      create  mygame/.rspec
      create  mygame/spec/spec_helper.rb
      create  mygame/spec/mygame_spec.rb
      create  mygame/.travis.yml
Initializing git repo in /Users/wsmoak/projects/mygame
$
</pre>

...and then do the initial commit, and repeat the fixes for the version number and capitalization.

Much better!

Now <code>bundle exec rspec</code> works.  The project skeleton comes complete with a failing test, so the output is:

<pre>
$ bundle exec rspec

MyGame
  has a version number
  does something useful (FAILED - 1)

Failures:

  1) MyGame does something useful
     Failure/Error: expect(false).to eq(true)
       
       expected: true
            got: false
       
       (compared using ==)
     # ./spec/mygame_spec.rb:9:in `block (2 levels) in <top (required)>'

Finished in 0.00155 seconds (files took 0.09872 seconds to load)
2 examples, 1 failure

Failed examples:

rspec ./spec/mygame_spec.rb:8 # MyGame does something useful
</pre>

Next up:  writing a _real_ failing test and the first bit of code for the game.

