---
layout: post
title: Do LLMs spell the end for programming language innovation?
---

When I returned to the office after Christmas this year,
I opened up VS Code and got busy with some development,
but something was not quite right. Before long I realised, Copilot wasn’t working!
Despite having only started using it daily a few months earlier, it felt strange to work without it.
Of course I can code as I did before, just as I could work without an IDE, typing directly into Notepad,
or make do with an 800x600 monitor, but with any productivity boosting advancement, once you experience it, it’s tough to go back. 

Just as I've adapted to the ChatGPT/Copilot world, so have millions of others. And however many holdouts there are among my generation, you can be sure the next wave of developers will fully embrace them and storm the workforce armed with their AI assistants. I recently talked with a team who have adopted [Cursor](https://www.cursor.com) as their IDE and by all accounts it’s the real deal.

If you’re thinking these tools are only for those who can’t cope on their own, [here is Salvatore Sanfilippo](https://antirez.com/news/144), better known as antirez, the creator of Redis, extolling the benefits 

> Basically, AI didn’t replace me, AI accelerated me or improved me with feedback about my work.

We’ve come a long way since my first coding job at age 19, for a two person company building an application for users to manage an inbox of SMS messages.
We worked in Pascal, the language I had been taught in school. Since then I have programmed
professionally in Java, Scala, JavaScript, Python, Typescript and R (and of course SQL!),
and unprofessionally in Fortran, PHP, Perl, Ruby, C and C#. Some of these didn’t exist back
when my coding career got started, and some predate me by several decades. I love getting
impactful work done in code, but I also love programming languages for their own sake. The
different design choices, patterns, strengths and weaknesses. 

I firmly believe that the better programmer you are, the more these AI tools can boost your
productivity. But they are not equally able across all languages. Large Language Models rely on
a massive corpus of data to train on. ChatGPT and Claude are good at Python because they have
seen a huge portion of all the Python code ever written. Performance on existing languages has
been shown to vary significantly. Take for example [this early study on ChatGPT 3.5](https://arxiv.org/pdf/2308.04477).

> The findings provided in this study suggest that the language competency of ChatGPT is
> affected by two primary factors: 1) the level of abstraction of the language, and 2) the
> popularity of the language, which enables the model to be trained on a more extensive corpus.

The same [appears to hold true for ChatGPT 4](https://arxiv.org/abs/2501.02338).

> ChatGPT 4 demonstrated higher competence in widely used languages,
> likely due to a larger volume and higher quality of training data.

And [GitHub are transparent](https://docs.github.com/en/copilot/using-github-copilot/getting-code-suggestions-in-your-ide-with-github-copilot#about-github-copilot-and-visual-studio-code)
that Copilot is not equally capable in all languages:

> GitHub Copilot provides suggestions for numerous languages
> and a wide variety of frameworks, but works especially well for Python, JavaScript, 
> TypeScript, Ruby, Go, C# and C++.

What chance would a brand new, innovative language have in a world where almost all code is 
written with LLM assistance? Who would adopt the next Go or Rust for a project if their AI pair 
programmer suddenly becomes useless? Like many decisions, switching to a new programming
language can be looked at through the lens of cost-benefit analysis. 
Losing the contribution of your
preferred AI assistant drives the cost higher, while the benefits stay static at best, or
are perhaps eroded by the ability of the AI to smooth over some of the irritations of existing
languages.

My expectation is we will see a hegemony develop with Python as the high level language of choice, and a small number of other languages such as C/C++, Java, C#, Go and Rust where performance matters. Python has been towards the top of language popularity indexes such as
[Redmonk](https://redmonk.com/sogrady/2024/09/12/language-rankings-6-24/),
[IEEE](https://spectrum.ieee.org/top-programming-languages-2024)
and [TIOBE](https://www.tiobe.com/tiobe-index/)
for a while. IEEE and TIOBE make statements that suggest this consolidation
around Python driven by AI may have already started:

> At the top, Python continues to cement its overall dominance,
> buoyed by things like popular libraries for hot fields such as
> AI as well as its pedagogical prominence. [IEEE Spectrum]

> Python gained a whopping 9.3% in 2024. This is far ahead of its competition:
> Java +2.3%, JavaScript +1.4% and Go +1.2%. Python is everywhere nowadays,
> and it is the undisputed default language of choice in many fields.
> It might even become the language with the highest ranking ever in the TIOBE index.
> [TIOBE Index]

Some standardisation may well be beneficial for efficiency in the industry,
and arguably overdue, but for programming language enthusiasts like me,
it is a bittersweet outlook. Programming languages, like human languages, 
carry elements of culture, beauty and art alongside their usefulness. 

For now though, my Copilot is working again and, at least in Python, I’m more productive than 
ever. And when the urge to learn a new language takes me, I suppose I can always go and learn 
[Shakespeare](https://shakespearelang.com/1.0/) or [Chicken](https://esolangs.org/wiki/Chicken).
