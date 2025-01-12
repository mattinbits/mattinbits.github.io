---
layout: post
title: Learning about excellence from Roger Rabbit
---

![Roger Rabbit](/img/rogerrabbit.png){:style="display:block; margin-left:auto; margin-right:auto"}

If you have kids of a certain age, you’ve probably seen Moana a hundred times. This is not a new phenomenon, I used to watch Beauty and Beast on repeat with my siblings, on VHS. We can still sing most of the songs by heart. Rewinding the tape just gave the perfect opportunity to dash for snacks. A tale as old as time, you might say.

Still, in that pre-streaming era, there was something special about chancing upon a film you'd never seen before, while aimlessly flicking through channels. A now-or-never chance to be thrown into a brand new story with no warning, and no idea of what to expect.

For me, one of those films was _Who Framed Roger Rabbit_. I probably only saw it once as a kid, maybe not even all the way through, but I remember being engrossed by the fusion of the urban, real world setting and the larger than life toons.

Robert Zemeckis’ film is renowned for the brilliance with which it blends live action and animation. There’s one scene in particular that has become a microcosm for the extreme skill and dedication that went into making it. Just watch this:

<div class="video-container">
<iframe width="560" height="315" src="https://www.youtube.com/embed/_EUPwsD64GI?si=WflyHykWiF_BOfUT" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
</div>
<br/>

Several times in the scene, Eddie bumps his head on the ceiling lamp. This is not just for slapstick comic effect, although there's plenty of this to go around. It required the animators to individually shade Roger into each frame such that the shadows cast on him, and by him, match up with the swinging light.

The creators of the film didn’t need to have Eddie bump the lamp. It would have been simpler, and “good enough” to leave it undisturbed. But by including the bumps, and taking the painstaking effort to create the animation to match, the audience experiences an unparalleled level of immersion.

“Bumping the lamp” has since [become a symbol of going the extra mile to create a near perfect result](https://www.mrroughton.com/blog/bumping-the-lamp). That “good enough” is not, in fact good enough. As Lumiere might put it, "a dinner here is never second best".

I am not an animator. No lamps were bumped in the production of this parasaurolophus riding a skateboard. 

![Pencil drawing of a dinosaur](/img/para.png){:style="display:block; width:300px; margin-left:auto; margin-right:auto"}

But I do write software. When building software systems, especially in a corporate environment, the work is often considered done when “good enough” is reached. What is the equivalent in our field of bumping the lamp? Everyone will have their own list (let me know yours!), here are some of mine.

- Produce comprehensive, automated tests that cover as many scenarios and edge cases as possible. Including the ones you _really_, _really_ didn't feel like writing.
- Write great documentation. Both for end users, and for other developers. Some good examples are [Pydantic](https://docs.pydantic.dev/latest/) and [SqlAlchemy](https://docs.sqlalchemy.org/en/20/). Bonus points for documentation rendered and deployed automatically from the source repository, and always kept up to date.
- Specific, human readable, actionable error messages. For example [Rust compiler errors](https://rustc-dev-guide.rust-lang.org/diagnostics.html).
- Eliminate all manual deployment tasks. It’s often the case that 90% of a deployment is easily automated, and the last 10% presents a disproportionately large effort. Persist with the task and get to 100% automation.
- Polishing the user experience beyond what a user would consider satisfactory, for example achieving best-in-class responsiveness in the user interface.

There plenty of other examples we could come up with. For any given one, we could go the extra mile and tell ourselves we are doing the software engineering equivalent of the film makers opting to bump the lamp. But although this one scene has become the metaphor, the lesson we should take from the film is a little different.

Roger Rabbit is not excellent because of a single example of the team going above and beyond, but because director Robert Zemeckis and lead animator Richard Williams insisted on striving for brilliance scene by scene, frame by frame, person by person. Because of the culture fostered on set such that everyone felt the freedom and expectation to do the same. This shines through in various behind the scenes material.

<div class="video-container">
<iframe width="560" height="315" src="https://www.youtube.com/embed/60hCE3ld9yA?si=w7Ik_YzmfKNDmHkc" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

<iframe width="560" height="315" src="https://www.youtube.com/embed/siAby_GLQIg?si=s6xNTG43h-HG81pK" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

</div>
<br/>

If we want to lead teams to build systems that are better than “good enough”, we must channel our inner Zemeckis to foster a culture, through our communication and through the example we set, that excellence is the default. The end users of those systems, like the audience of the film, will feel the cumulative effect of the effort, even without being able to pinpoint the specific techniques involved.

If you’re just finishing Moana on Disney+ for the hundred-and-first time, do yourself a favour and flick over to [Who Framed Roger Rabbit](https://www.disneyplus.com/en-dk/movies/who-framed-roger-rabbit/20GDm8DYpIsC). You’re Welcome! 




