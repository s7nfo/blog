---
layout: post
title:  "TIL: Terrence Tao on Machine Assisted Proofs"
---

<iframe width="560" height="315" src="https://www.youtube.com/embed/AayZuuDDKP0?si=lNQKfWDVNUPtVLOR" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

In this TT talks about how mathematicians usually work by themselves or with
colleagues they trust a lot, because otherwise you spend a lot of time checking
everyone's work for errors.

With [Lean](https://lean-lang.org) (a theorem prover) you can automate this checking. At the moment writing Lean proofs takes ~10x longer than doing it by hand, but it gives you greates scalability through lowering the trust barrier. And presumably the cost won't be 10x forever.

It also simplifies change management. Changing one parameter no longer means having to go through the whole proof by hand.

This pattern matches what large software engineering organizations have been doing for a while. Automated testing, types systems, ...

Speaking of the 10x cost of formalizing proofs, this is something LLMs might be helpful for. Instead of formalizing everything by hand, you ask an LLM to do it for you. It's not going to get it right every time, but at least it's easy to verify whether it did or not. And even if it does not it might suggest an approach you can work out yourself.

Could software engineering learn from this? Is it finaly [TLA+](https://en.wikipedia.org/wiki/TLA%2B)'s time to shine with the help of LLMs? Could other fields, physics, chemistry, have formal languages with automated checking?

I liked the [Blueprint](https://github.com/PatrickMassot/leanblueprint) pattern a lot too: a human comes up with a “blueprint” of the proof that is linked to the Lean formalization, breaks down the work needed into smaller lemmas and definitions that can be tackled individually and provides a snapshot of progress.

![blueprint](https://terrytao.files.wordpress.com/2023/11/image.png)
