# Deleting the feature I used once

*2026-10-23*

---

*This post is part of a series on the technical decisions behind tinAI, a
local-first AI assistant I run on a consumer laptop. This one is about a
feature that worked correctly and still had to go.*

---

In August I shipped expense tracking into my assistant. Photograph a
receipt, send it to the bot, a local vision model pulls the total out of the
image, the bot offers a row of category buttons, I tap one, and the expense
lands in a JSON file with a monthly report generated on the first of each
month.

It took five build steps and it worked. Over the following six months I used
it once.

Last week I deleted all of it.

## The part that was easy to be proud of

The implementation had some genuinely nice pieces. Receipt totals were
extracted by a 4-billion-parameter vision model running locally, which
meant photographs of my shopping never left the machine — the whole point of
the project. Currency was configurable, because I had just moved country and
the previous hardcoded currency was suddenly wrong. Categorisation was one
tap rather than a typed label, because typing on a phone while standing in a
shop is not a thing anyone does.

Every one of those decisions was a reasonable answer to a real constraint.
None of them mattered, because they were all answers to the wrong question.

## The part I did not want to look at

The feature asked me to do something at the exact moment I have the least
capacity to do anything: immediately after paying for something. Standing at
a till, bag in one hand, phone in the other. Photograph the receipt — which
means keeping the receipt, which means remembering at the counter that I
keep receipts now.

Not hard. Just one more thing, at the worst possible time, with the reward
arriving weeks later in the form of a report I had not asked anyone for.

The telling detail is that I knew the usage number before I checked it. I
had a precise memory of the one time I used the feature, which is the kind of
memory you only have about things that happened once. I checked anyway,
because I wanted to be wrong, and the file had one entry in it.

My first instinct was to improve it. Maybe the category buttons were in the
wrong order. Maybe it should ask at the end of the day instead, so I could
batch. Maybe a reminder. I had a short list of plausible fixes before I
noticed what I was doing: looking for a UX bug in a feature whose problem
was not UX.

## The question I had not asked

The assistant was asking me for data that something else already had.

Every card payment I make is already recorded, with a timestamp, an amount,
a merchant name, and — this is the part I had not thought about — a merchant
category code. That is a short numeric code the payment networks attach to
every transaction to say what kind of business took the money. It exists so
that card schemes can route and price transactions, and it has been there,
on every transaction, the entire time I was photographing receipts.

So the replacement is not a better tracker. It is no tracker at all. The
assistant now reads statements from my bank's personal API, read-only, and
derives categories from those merchant codes. There is nothing for me to do.
The data was always there; I was just re-entering it by hand, badly,
one receipt in six months.

## Why a lookup table beat the model

The obvious move, in a project full of local language models, would have been
to let a model read each transaction description and assign a category. I
tried that. It hallucinated categories, and worse, it hallucinated
currencies — confidently labelling transactions in a currency the account
had never held.

A merchant category code maps to a category with a dictionary. It is less
clever. It gives the same answer twice. It costs nothing, cannot invent a
currency, and I can read the mapping and tell you why any single transaction
landed where it did.

This keeps happening to me and I want to say it plainly: not every problem
that sits next to an assistant is a problem for the model. The interesting
part of this feature is a lookup table. The model's job was to be absent.

## Two things the API taught me the hard way

**Amount and currency describe different things.** In a statement record,
the transaction amount and the running balance are denominated in the
*account's* currency. The currency code on that same record describes the
*operation* — what currency the thing was actually priced in. Read them as a
matched pair, which is what the field names invite you to do, and you get
numbers that look completely plausible and are wrong. I shipped that bug,
found it, and fixed it by reading the field documentation instead of
inferring meaning from names. There is no clever lesson here beyond: field
names are not a specification.

**The rate limit is part of the design, not an obstacle to it.** The API
allows roughly one statement request per minute, and caps each request to a
bounded date range. You cannot page through a year of history in a loop; you
get refused. So requests go out in windows and land in an on-disk cache, and
the caching is not an optimisation — it is the only shape in which the
feature works at all.

## What I am taking from this

A deleted feature is a result. It is not a failed one.

The thing I will actually carry forward is a changed question. I had been
asking "does this work?" — and it did, every step of it, with tests. The
better question is "is it being used?", and that one I can only answer after
the fact, by looking at a file and counting the rows.

So I am trying to ask it earlier now, and about more of what I build: not
whether the assistant *can* do something, but whether the version of me who
is tired, holding a bag, standing at a till, will ever do their half of it.
Most of the time the honest answer is no — and the feature worth building is
the one that does not need me there at all.
