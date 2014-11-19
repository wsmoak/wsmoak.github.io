---
layout: post
title:  "A Small Fix for Growstuff"
date:   2014-11-19 12:30:00
tags: ruby rails growstuff
---

Yesterday on [Growstuff](http://growstuff.org) I noticed that the 'New garden' page would let you create a garden with a negative number for the area.  That's not right!

I checked on [staging](http://staging/growstuff.org/gardens/new) to see if it was already fixed, and then had a look in the [Pivotal tracker](https://www.pivotaltracker.com/s/projects/646869) to see if it had been reported.  Nothing showed up, so I set about fixing the problem.

First I made a new branch on my GitHub fork of Growstuff using the web interface, then I switched to it in my local clone...

{% highlight console %}
$ git checkout negative-area
{% endhighlight %}

... and opened it in my editor.

I found what I was looking for under <code>app/models</code> in the <code>garden.rb</code> file:

{% highlight ruby %}
  validates :area,
    :numericality => { :only_integer => false },
    :allow_nil => true
{% endhighlight %}

That appears to be making sure the value (if present) is numeric.  It doesn't look like it prevents negative numbers, which is the behavior I noticed.

I found the documentation for "numericality" in [Active Record Validations](http://guides.rubyonrails.org/active_record_validations.html#numericality) and made this change:

{% highlight diff %}
  validates :area,
-   :numericality => { :only_integer => false },
+   :numericality => { :only_integer => false, :greater_than_or_equal_to => 0 },
    :allow_nil => true
{% endhighlight %}

Then I started the app locally, tried it out, and it worked!  Or rather, creating a garden with a negative number for the area DIDN'T work, which is what I wanted.

Now, though, the guilt set in.  I had changed code without a failing test!  The <code>git stash</code> command is useful here, to save a work in progress and pick it up later.

I added a feature test in <code>spec/features/gardens_spec.rb</code>

{% highlight ruby %}
  scenario "Refuse to create new garden with negative area" do
    visit new_garden_path
    fill_in "Name", :with => "Negative Garden"
    fill_in "Area", :with => -5
    click_button "Save"
    expect(page).not_to have_content "Garden was successfully created"
    expect(page).to have_content "Area must be greater than or equal to 0"
  end
{% endhighlight %}

That failed as expected, and then passed after I <code>git stash pop</code>ped my change back into place.

All done!  Or so I thought.  I submitted my [pull request](https://github.com/Growstuff/growstuff/pull/452), and immediately got a comment asking for a unit test in addition to the feature test.

That belongs in <code>spec/models/garden_spec.rb</code>:

{% highlight ruby %}
    it "doesn't allow negative area" do
      @garden = FactoryGirl.build(:garden, :area => -5)
      @garden.should_not be_valid
    end
{% endhighlight %}

I removed my code change to make sure this test also failed, then restored my patch and tried it again to see it pass.  I committed the second test and was about to ask how to update the pull request when I noticed that it happened automatically!

A committer already reviewed and approved it, but the dev branch is frozen while they prepare for a release.  It will get merged after that happens, probably some time next week.

All in all that was a lot of typing for a very small bug that wasn't really hurting anything, but now the app is a tiny bit better, and I know a few more things about Rails and Active Record.