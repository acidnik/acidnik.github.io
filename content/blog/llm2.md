+++
date = '2026-08-03T17:26:25+04:00'
draft = false
title = 'Practical Advice for Developers Integrating AI. Part 2 - Creativity'
+++

In [part one]({{< ref "llm.md" >}}), we covered the technical details of integrating LLMs into your app. Today, we will discuss the creative side of things.

> **AI Disclosure:** This text was originally written manually. AI was used only for minor style and grammar refinements.

# Stage 0. Excitement

I suppose every LLM developer goes through these stages. Stage 0 is when you finally implemented your MVP: Wow! LLMs are actually so good at creativity! Your AI girlfriend is such an interesting interlocutor! Your AI-generated video clip has such an interesting original plot!

But then, after you played with your MVP for a day or a week, you notice something. All your AI girlfriends are acting the same way. All your video clip plots are repeating the same idea. And that's exactly what you should expect: AI is just predicting the most probable words, and similar input will always produce similar output.

# Stage 1. Temperature, Top-K, Top-P

But don't we have tools for exactly that? Aren't Temperature, Top-K, and Top-P *the* options? This is the first thing you should try. Raising temperature to 1.2–1.5, setting top_k=64, and top_p=0.95 should increase the variance of the model's output. But it will not change the order of tokens that your model predicts, only increase the number from which the model makes a choice. And if the top 5 words are the most probable, it will almost always choose them. And raising the temperature higher will result in completely unexpected tokens (usually Chinese characters).

# Stage 2. "Just be more creative"

LLMs are smart, right? Just tell them what to do and they will follow your orders! You add to your prompt something like this:

> Be creative, be courageous, use different plot ideas, don't limit yourself with obvious plots

Well, this just doesn't do anything at all. Zero effect whatsoever. The LLM has no awareness of its previous outputs, so for every new request with this prompt, it will bravely output: "Here's a new and original idea - a protagonist is walking from one location to another!"

# Stage 3. Asking LLM to write prompt

At this point, you're most likely asking your favorite AI: what can we do to increase creativity? And your favorite AI will gladly offer a solution: 

> How about offering more ideas in the prompt? Your model always chooses 'Cinematic realism' for the visual style? Let's try offering different styles in the prompt:
> Choose a visual style, such as (but not limited to): Cinematic realism, Claymation, Pop-art, Aquarelle painting, Cyberpunk ... [a good dozen different options]

LLMs sure love to offer solutions based on enumerating things. Unfortunately, this doesn't really work either. In the best-case scenario, your model might now sometimes choose the second option from the list. But 95% of the time, it chooses the most obvious first option you gave it.

*Enumerating options in the prompt never increases creativity; on the contrary, it limits it to the provided options.* If you are refining your prompt with an LLM - make sure to strictly forbid putting examples in the prompt.

# Stage 4. Negative rules

In some cases, such as choosing a plot for a music video, there's an obvious strong attractor that models just cannot avoid. Apparently, most video clip plots are just "a protagonist's journey" - at least, that's what most LLMs are trained to think. 

At this stage, you are so fed up with the same plot that you decide to put a strict ban on it.

Adding such a rule can be tricky, since the LLM will try to output something very similar but not exactly the same. It can take several attempts, but the good thing is that the goal can be explained, and the whole process can be scripted and automated.

This approach actually works, especially for models with reasoning. But it still feels like now it's just always choosing the "second-best idea." And the fact that you completely excluded one of the ideas - which is actually very powerful as long as it's not overused - doesn't feel right at all.

# Stage 5. When enumerating options is a good idea

I'm glad that you read all the way to this part. Time to show what actually works.

At *Stage 3*, you may have noticed that LLMs are actually really good at coming up with multiple examples. They're just bad at choosing a different idea every time. Well, we can help with that, right?

The key idea is to ask the LLM to output multiple options to choose from:

> Generate exactly 5 distinct visual treatments. Each must be a short, dense conceptual definition focusing ONLY on the visual style. Think in terms of visual styles that are a good fit for music video clips for a given genre and lyrics.

Combine this approach with JSON output, and you've got a list of 5 options to choose from. Then just use `random.choice`, and there you have it.

This may not be obvious at first glance, but there's a deeper idea here. What is `random.choice`? It's a traditional algorithm that is not available to the LLM. 

LLMs may feel like magic at first. You tell them what to do, and they do it! But eventually, this feeling fades, and you begin to see the limitations. And sometimes the solution for those limitations is not "more LLM" but a step back. Combining an LLM with strict algorithms can help close the gap. And `random` is not the only option; depending on your task, you may come up with more complicated rules to steer the LLM away from obvious answers to areas of exploration and creativity.

**The future of LLM development isn't just better prompts - it's smarter orchestration.**
