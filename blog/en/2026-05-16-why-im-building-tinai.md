# The moment — why I'm building tinAI

*2026-05-16*

---

*Note for the reader: this is the first post in a series about why I'm building tinAI. The next posts will cover the technical decisions: tier routing, local memory, the read-only Obsidian contract. This one is about why the project exists at all.*

---

When I started thinking about a local AI assistant, I wasn't building anything. I was at war.

This was during the full-scale invasion. The structural problem I kept running into: critical decisions had to be made fast, on fragmented data, often without connectivity, with people tired enough that basic arithmetic became a risk.

Situational analysis. Approach planning. Map verification. Quick translation of intercepts. Pulling scattered reports into a single picture. All of these are decision-support tasks where a good AI next to a human would have given **time**. Not heroics, not "AI instead of a soldier." Time. A few minutes saved on routine analysis that could have been the difference.

Tools of that class did not exist. What did exist either didn't work without internet, or was built in the country we were fighting. Yes, we used it, because nothing else was available and people were dying faster than patriotic principles could fix that. Or it required a level of technical training most units simply didn't have.

That gap stayed with me. Critical decisions, made under stress, needed computation. Computation needed software. Software was either absent, foreign, or dependent on connectivity that the front line couldn't promise. And the people closest to the consequences had no way to fix it from where they stood.

I left the army in 2023 after five years of service. The thought didn't crystallize then. It sat there as a feeling that AI, deployed right, could have saved lives.

By 2025, AI on the front lines had stopped being a "we wish we had this." Ukrainian AI-guided drones became three to four times more effective than human-piloted ones. A million units planned for this year. AI is helping analyze reconnaissance data, guide strikes, intercept enemy drones. The trajectory I had only sensed in 2022 has been put on a production line.

And now I'm a civilian, with the time and a laptop to actually build something. Not for the front lines. There are people there doing it faster than I can. But on the principle the war taught me: the most important computation needs to run **where the user stands**, on hardware they control, without depending on someone else's server staying online and friendly.

That is what tinAI is built around. Not because local-first is technically elegant. Sometimes it isn't. But because the war made one thing obvious: infrastructure you don't control is infrastructure you can't count on when it matters.

---

*tinAI is a local-first AI assistant running on an ASUS ROG laptop with an RTX 3050. Architecture and decisions are documented at [github.com/frdvch/tinai-public](https://github.com/frdvch/tinai-public).*
