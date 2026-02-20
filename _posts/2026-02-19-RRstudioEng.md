---
title: "R is not RStudio"
excerpt: "and that's OK"
category: rstats
tags:
  - courses
  - teaching
  - IDEs
  - Positron
header:
  image: /assets/images/featureIDE.png
  caption: "Photo from tidy dev day, borrowed from Bea Milz's blog"
---


Sometimes a brand or company becomes so popular that its name practically becomes synonymous with the product it offers. There are many examples of cases where a brand name practically replaces a product name, like frisbee, thermos, or kleenex.

In linguistics this is known as _genericization_ (thanks to [Riva Quiroga](https://rivaquiroga.cl) for teaching me the term), and this concept also applies to software. For example, referring to any spreadsheet as an *"Excel"* . This also happens a lot with **R** and **RStudio**:

- **R** is the programming language and statistical computing environment that has existed for more than a quarter of a century.
- **RStudio** is an integrated development environment (*IDE*) that makes it easier to work with R. It has been available since 2011, pretty much when I started learning to code.


In Latin America, and at least in my experience, there is little conceptual distinction between the two. The R instance running inside RStudio is perceived as just another pane in the interface, and the name of the language itself is rarely mentioned explicitly.

<figure>
    <a href="/assets/images/elhrri.png"><img src="/assets/images/elhrri.png" ></a>
</figure>

In addition to the confusion between the language and the IDE, there is a frequent mixing of visual identity: the RStudio logo is used instead of the official R logo. Interestingly, this tendency is less common in English-language educational offerings, where references to the R language predominate over the IDE.


Although it's not my area of specialization, I have not noticed a similar confusion with **Python**. References in courses, theses, or manuals usually focus on the language itself, not on a specific IDE. For example, it's rare to see courses titled *"Introduction to statistics with PyCharm"* or *"Spyder for biologists"*. I assume this is because Python doesn't have a single dominant IDE.


* * * 

I brought this up with some colleagues, we concluded that this ambiguity could originate in academic or student communities that perceive RStudio as *"another program for doing statistics"* that is free and user-friendly in contrast to commercial alternatives like SPSS or SAS. Since RStudio is the executable we interact with daily, its name ends up overshadowing that of the underlying language.

When I teach introductory courses, I always clarify that **R is not RStudio**:  
  
> - R is the programming language: it can be run from the terminal, its native GUI (on Windows), or any IDE.  
> - RStudio is a graphical interface that makes working with R easier. It's popular, convenient, and my usual choice for teaching, but **it's not mandatory nor the only option**.

In fact, when downloading RStudio from the Posit website, the instructions indicate installing R first and specify that R is *not* a Posit product. However, I noticed that a clear explanation is missing regarding R is an independent project managed by the [R Project for Statistical Computing](https://www.r-project.org/), with its own foundation (R Foundation) and development model.


At the end of 2024 I presented at Nerdear.la about **Positron**, Posit's new IDE for data science. While researching the topic, I learned that IDEs have existed for more than 30 years and that for R there have been multiple options that never achieved massive popularity like:

- Eclipse StatET™
- Nvim-R (for Neovim)
- RKWard
- R extensions for VSCode

To date, most people who program in R continue using RStudio. I have noticed more interest in Positron, especially among those who closely follow social media, events, meetups, etc. In November 2025 I incorporated Positron as an optional tool in my courses, with somewhat positive results and above all to reinforce the conceptual distinction between language and IDE.


This post is mostly to organize my observations about the confusion between R and RStudio, as reading material in my courses, to generate discussion in the community, and as background for in-depth work I want to do on the **barriers and opportunities in the adoption of new tools** including IDEs. I understand that those who use the terms flexibly are not seeking to undermine the work of the people who maintain the language, nor the R Project for Statistical Computing, nor the R Foundation/R Consortium. I also don't think Posit PBC (the company that developed RStudio and that was formerly called RStudio) seeks to monopolize the language's identity. More than anything, it seems to me to be an interesting linguistic and cultural phenomenon.


