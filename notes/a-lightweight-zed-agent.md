# A Lightweight Zed Agent

*8 June 2025*

A couple of weeks ago I participated in my company's annual hackathon with the theme 'AI Agents'. After that first brush with looking at how AI agents can be put together and what they can achieve, it was fortunate timing to encounter Thomas Ptacek's post [My AI Skeptic Friends Are All Nuts](https://fly.io/blog/youre-all-nuts/). I'm fairly wary of taking too many shortcuts with what I've been learning as a developer - I know that I learn well from repetition and practice. But I am often hitting one of two issues in my tickets at work: 

1. They relate to code which is several years old and there's no one to ask for debugging help/clarification. 
2. The task isn't difficult, but it is not adding to my knowledge anymore - it's just time consuming. 

I've been using Copilot chat for the first use case with reasonably helpful results. Most of the time I can get enough out of the responses to at least find my way forward. 

Ptacek makes a really emphatic case for bringing in AI agents in the second use case (while also touting their benefits for improving code review skills). Since I'm a bit of a curious cheapskate, I've been playing with local models rather than getting into paid subscriptions. He doesn't drop too many names in the post, but did mention that [Zed](https://zed.dev/) has a configurable agent mode, so I've been checking it out a bit. 

The question I'm most interested in is: Can a model be effective in reducing my ticket grind but lightweight enough to run locally? 

To set up, I installed Zed as well as Ollama. I ran one model at a time and removed them when I wasn't using them. 