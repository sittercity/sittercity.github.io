---
layout: post
title:  "Creating A Design System"
author: "Byonca Honaker"
date:   2016-07-26 15:00:00
categories: technology
---

I had the opportunity to go to [Smashing Conference](http://smashingconf.com/) in San Fransisco. It was a great experience and would highly recommend anyone interested in front-end technology to attend. At Sittercity, we are asked to write or present what we learned at a conference. I found Alla Kholmatova’s talk on <i>Atoms, Modules, and Other Fancy Particles</i> to be the most interesting because it could be beneficial to our new product, [Chime](http://www.hellochime.com).

Kholmatova works for a company called [FutureLearn](https://www.futurelearn.com/). It is a massive, open online course learning platform. They derive their design thinking from Brad Frost’s Atomic Design. [Atomic Design](http://atomicdesign.bradfrost.com/) is a methodology for creating effective design systems by utilizing reusable components. These components can be broken down to <i>Atoms, Molecules, Organisms, Templates, and Pages</i>.

<!-- More -->

Kholmatova presented examples on how her team drew inspiration from Atomic Design for FutureLearn. The foundation of a design system can be summed up by this quote from Donella Meadows' book <i>Thinking in Systems</i>:

>“A system is an interconnected set of elements coherently organized in a way that achieves something.”

Kholmatova stressed the importance of how a system is more than a collection of components. In order for them to work together, they must share a purpose and show their connections to one another. Similar to teams in game, the team (elements) must cooperate with one another (connection) to win the objective of the game (purpose).

>“A design system is a set of coherent patterns that facilitate and encourage certain types of behaviour.”

She provided a few principles for creating an effective design:

+ Name things based on their global function — the function of the module in the context of the whole system, rather than a page. This is will help establish their rightful place in the system as a whole.

<br>
![FutureLearn's Boombox](/assets/img/design_systems/boombox.png)

<center>This is FutureLearn’s <i>boombox</i>. It is used for: A louder alternative to a <i>whisperbox</i> that is used to nudge users towards secondary actions on the page, such as purchasing a certificate, but in a way that's more conspicuous and less likely to be overlooked.</center>
<br>

+ Name things collaboratively and refer to them by their names. Developers and designers should both work and understand the system in order to utilize it to the fullest. Names with personality are easier to remember. They stick around and inspire other names.

>“What matters is not the structure itself but that it’s shared and understood by the people who use it.”

+ Sometimes it’s better to make the module generic, and sometimes more specific. Names should reflect that.

<br>
![FutureLearn's Generic Whisperbox](/assets/img/design_systems/whisperbox_generic.png)
<pre>= render layout: 'shared/whisperbox', locals: { whisperbox_heading: "A generic whisperbox" } do
  %p
    Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod
    tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam,
    quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo
    consequat.
</pre>
<br>

<br>
<center>A social related <i>whisperbox</i>.</center>
![FutureLearn's Social Related Whisperbox](/assets/img/design_systems/whisperbox_social.png)
<br>
<pre>
= render layout: 'shared/whisperbox', locals: { whisperbox_heading: "Join the conversation on social media", whisperbox_modifier: 'm-whisperbox--social'} do
  %p
    Use the hashtag
    %a{:href => "#"} #FLwebsci
</pre>

<br>
<center>An alternative section <i>whisperbox</i>.</center>
![FutureLearn's Alternative Section Whisperbox](/assets/img/design_systems/whisperbox_alt_section.png)
<br>
<pre>
.a-section.a-section--alt
  = render layout: 'shared/whisperbox', locals: { whisperbox_heading: "I am on alternative section", whisperbox_modifier: 'm-whisperbox--on-alt m-whisperbox--with-statement-icon'} do
    %p
      Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod
      tempor incididunt ut labore et dolore magna aliqua.
</pre>
<br>

+ If you can’t come up with a name for a thing, something is not right with it. This was inspired by Abby Convert from her book <i>How to Make Sense of Any Mess</i>. 

>“When this [coming up with a name] cannot be done, the module is too complex and must be broken down even more.”

What I appreciated about this talk was how it focused on both designers and developers in FutureLearn collaborated to create this design system. It wasn't left for one team to do in isolation and then other teams adapt and follow. It was a breath of fresh air in the designer vs. developer debate. Cross-team collaboration is necessary in order to create a product that is both designed well and functional to the user. I think creating a design system would help Chime have a more solid and consistent feel. We are currently re-design a new look for Chime and this would be the perfect opportunity for fresh start for Chime.