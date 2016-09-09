---
id: 67
title: '<span data-rel="title">OpenStack Cinders reference implementation</span>'
date: 2016-02-12T16:51:33+00:00
author: jgriffith
layout: post
guid: https://griffithscorner.wordpress.com/?p=67
permalink: /2016/02/12/openstack-cinders-reference-implementation/
categories:
  - Uncategorized
---
<span data-rel="content">

<p>
  As many are fully aware, over the years I&#8217;ve had pretty strong opinions about good and bad things in the OpenStack Cinder project.  Let&#8217;s be clear, most of the &#8220;bad&#8221; are criticisms against myself and how things evolved than they are about the project or the people contributing to it.
</p>

<p>
  Lately there&#8217;s been a resurgence of the &#8220;LVM as a reference implementation&#8221; discussion due to a comment I made on twitter about one off vendor features being in the Cinder API.  This created quite the buzz, ranging from &#8220;LVM sucks&#8221; to &#8220;Criticism doesn&#8217;t help the project&#8221;.  All of that aside, I wanted to point a few things out for people to think about on this topic.
</p>

<p>
  First, keep in mind the purpose of a &#8220;reference&#8221; implementation.  The WHOLE point of a reference implementation is to provide a reference for people to use in development.  LVM has been the reference because it most closely fits the model that other backend-devices use.  In particular iSCSI targets attached to LVM volumes, in my opinion provides a really flexible and dynamic transport/connect mechanism in a cloud environment.  Some argue that things like NFS are better.. and maybe in some cases that&#8217;s true, but the original design and architecture in OpenStack was to provide Block Storage, so in that context iSCSI was a pretty cheap and flexible way to go about it.  I&#8217;d also point out that iSCSI and networking in general have come a VERY long way in the last 5+ years!  This isn&#8217;t your old iSCSI where people were trying to attach storage over a 1G WAN and shockingly had &#8220;issues&#8221;.
</p>

<p>
  So, what&#8217;s wrong with LVM?  Well, if you ask me the only thing wrong with it is that the OpenStack community only has a very few people that are really interested in paying any attention to it.  There was a comment last week to the effect that &#8220;using LVM as a reference implementation makes me sad&#8221;, my response was &#8220;why&#8221;?  There seems to be a misconception that there are things LVM can&#8217;t do that other OSS products can (in a Cinder context), but I&#8217;ve yet to figure out what those things are?  If you look at the code on Github and do a check between features in LVM and features in other OSS drivers you won&#8217;t find &#8220;missing&#8221; API calls.  Of course, have fun trying to figure out the structure with the whole feature-class thing going on in the base driver to link things together and figure out what&#8217;s actually real and what&#8217;s not (I&#8217;ll save that whole topic for another day&#8230; or just keep it between me and the Cinder community).
</p>

<p>
  The point is, this statement that &#8220;LVM is bad because it can&#8217;t do things&#8221; is just plain false.  There&#8217;s no Cinder API feature that it&#8217;s not capable of, the only exception to this is some of the custom one-off API calls that have been added specifically for one or two proprietary vendors that nobody was creative enough, or spent the time or effort on to implement in the LVM driver.  We used to have a *rule* that if you wanted to add a new feature or API call to Cinder, you had to FIRST implement it in the LVM driver.  Over time however Cinder grew, and folks decided this wasn&#8217;t such a big deal, and there should be exceptions.  So some vendors were allowed to add their own custom things.  The result, API calls in Cinder that may or may not work, and nobody EVER spending the time to go back and implement them in the Reference/LVM driver.  What&#8217;s also very telling here for me is to point out that these features that I was complaining about are NOT available in the other Open Source drivers either, Ceph included.  I&#8217;d also continue to assert as I have that the ONLY thing lacking with the LVM driver is interest from developers and the perpetual off-handed comments by people who don&#8217;t actually even know anything about how LVM works or the things that can be done with it.  Rather than say &#8220;LVM sucks&#8221;, maybe you should educate yourself on LVM a bit more and write some cinder/volume/drivers/lvm.py code.  I&#8217;ll admit, it requires a bit more effort than some other backends, and you certainly have to be creative.. but that&#8217;s the fun part.
</p>

<p>
  That brings me to my next point.  There&#8217;s been a few snarky comments about Ceph as the reference implementation.  Even better, there&#8217;s been some suggestions that &#8220;Vendors are thwarting this&#8221;, I find this almost as hilarious as I find it sad.  I&#8217;ve actually proposed the topic to a number of folks over the years, including at the Summit in Paris on a public panel with Sage Weil (creator and overall Ceph GURU).  The overwhelming response was &#8220;probably not a good idea&#8221;.  The reason for most folks that had that opinion was because Ceph is VERY different than any other storage backend.  In terms of a reference, it doesn&#8217;t necessarily help other technologies (iSCSI being predominant) .  Now, let me also say it again publicly and very clearly, I think Ceph is AWESOME, as do a lot of people deploying OpenStack.  It is not however the ONLY option.  If we wanted to deliver a product instead of a framework then yes, that might be what we&#8217;d want to consider.  The fact is though, we (OpenStack) are pretty explicit that we are NOT delivering a product, but we ARE building a framework.  I&#8217;d also point out that there are a lot of use cases out there, and there is no one product/backend that meets all of them.  There are very few large deployments these days that only have a single backend for Cinder.  Yes, I know there are exceptions here, but I&#8217;m saying &#8220;majority&#8221;.
</p>

<p>
  So where are we now?  Well, given where we are now maybe we should revisit this.  Here&#8217;s why; it turns out that earlier on a number of vendors didn&#8217;t follow the reference implementation anyway.  They either didn&#8217;t understand how it worked, didn&#8217;t care or didn&#8217;t like it.  So a number of the methods in the internal API do &#8220;different&#8221; things (or nothing) depending on the driver.  The problem with this is that then, a new developer comes along and wants to implement a driver.  They happen to notice Vendor-X&#8217;s driver does things a certain way that seems to align with their product so they use it as a template.  Now we have another driver that&#8217;s diverged.  Then the next developer comes along&#8230; he/she ends up taking some pieces from one driver, combined with some pieces from another driver that somebody in IRC worked on and pointed to in order to answer some questions.  What you end up with is inconsistency across all the drivers and a very tangled mess.  Remember what I said about criticism earlier?  Well, this is something that I criticize myself for.  As the person that started Cinder and the PTL for the first few years I should have caught these disparities in reviews and made sure they were fixed up.  I also should&#8217;ve made the reference cleaner and more clear so that there wouldn&#8217;t be questions about it.  I also should&#8217;ve continued my objections via -2&#8217;s for adding API methods that were NOT in the reference implementation.  Lessons learned, and that&#8217;s where the comments I&#8217;ve made on twitter came from over the past week.
</p>

<p>
  <em><strong>I&#8217;m all for revisiting the topic of Cinder&#8217;s reference implementation, BUT do me a favor; if you&#8217;re going to participate please do some homework.  Make sure you at least have a general idea of what the capabilities, use cases and problems we&#8217;re trying to solve are.  Also, ask the question, are we &#8220;building a reference, or a default&#8221;, because they&#8217;re not necessarily the same thing.</strong></em>
</p>

<p>
  &nbsp;
</p></span>