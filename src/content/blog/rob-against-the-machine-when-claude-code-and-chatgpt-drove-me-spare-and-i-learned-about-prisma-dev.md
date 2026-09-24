---
title: "Rob Against the Machine: When Claude Code and ChatGPT drove me spare and I learned about prisma dev"
description: "A story, and slight rant, about how I tried to learn from an AI and came completely unstuck"
pubDate: 2025-08-11
tags: ["ai", "llm", "prisma"]
---

I've been trying to experiment a little with agentic AI when writing code. In principle it sounds incredibly cool –* ask a robot what code you'd like written, and it does the heavy lifting for you.

At first it did some clever stuff. I got it to write a failing test, reviewed the test, and then got it to get the test to pass in the simplest way possible. Easy. The problem came when I started to ask it to do more complex things. As Claude Code churned away it started to run into problems, and started commenting out code in a very human way to try and get things working again. It then had to be prompted to re-enable the very code it had just commented out, and occasionally got into loops where it would delete code it had written seconds before.

After a while scratching my head and trying to work out what it had done, I ditched the generated code and reset the repo. I did come across one interesting line it had written in my .env file though, where it had added a very thorough-looking database connection string along with a comment referencing `prisma dev`. Not knowing the command, I asked ChatGPT:

<img src="__GHOST_URL__/content/images/2025/08/Screen-Shot-2025-08-11-at-10.09.31-p.m..png" alt="&quot;What does the prisma dev command do?&quot; / ChatGPT: &quot;prisma dev isn't actually one of Prisma's core commands – at least not in the official Prisma CLI as of 2025&quot;" />

This was on GPT 5, the "latest and greatest" of OpenAI's models, but it couldn't find anything. Not to worry, it might be out of date. Since Claude Code was the one to generate the environment variable in the first place, I headed over to Claude to work out what was happening:

<img src="__GHOST_URL__/content/images/2025/08/Screen-Shot-2025-08-11-at-10.11.28-p.m..png" alt="Claude gives a detailed description of &quot;Prisma's development mode&quot;" />

Great! We've got a description! But I can't see anything that would require a DB connection string, so what was Claude Code generating? Best to check with Claude on that detail:

<img src="__GHOST_URL__/content/images/2025/08/Screen-Shot-2025-08-11-at-10.11.47-p.m..png" alt="&quot;Does it spin up a DB?&quot; / Claude: &quot;No, prisma dev does not spin up a database server itself.&quot;" />

_Oh no._

OK, this is getting weird. Let's go old school and try the help flag:

<img src="__GHOST_URL__/content/images/2025/08/Screen-Shot-2025-08-11-at-10.20.20-p.m..png" alt="npx prisma dev --help describes the command as &quot;Spin up a local Prisma Postgres database&quot;" />

Claude seems to be outright lying now. According to the docs, spinning up a local database is the _only_ thing that `prisma dev` does. So I asked Claude for sources, and it admitted to making things up:

<img src="__GHOST_URL__/content/images/2025/08/Screen-Shot-2025-08-11-at-10.11.56-p.m..png" alt="" />

Fine, let's try Google for more info.

<img src="__GHOST_URL__/content/images/2025/08/Screen-Shot-2025-08-11-at-10.22.59-p.m..png" alt="Google's search results only describe prisma migrate dev, not prisma dev" />

_The Google, they do nothing!_

At this point I'm wondering what I've stumbled into. Google doesn't know about it, Claude lies about it but only because it doesn't know anything either, and ChatGPT says it doesn't exist at all. Meanwhile, Claude Code is merrily generating code based off its existence.

Back to ChatGPT, which at least claims to be searching the internet and updated documentation for things. I'll tell it that it works on my machine

<img src="__GHOST_URL__/content/images/2025/08/Screen-Shot-2025-08-11-at-10.10.27-p.m..png" alt="ChatGPT gives detailed options for what else it could be, none of which are the official command" />

So ChatGPT reckons that it's not an official command or I've got confused with a different command. I triple-checked my terminal: nope, I've been running the right command and there aren't any aliases.

And then came the inevitable facepalm moment. The moment when I realise my lengthy detour could have been entirely avoided. The moment when I questioned all sanity and my abilities of basic comprehension. I asked some friends.

> "I can see a dev command here?
>
> [https://www.prisma.io/docs/orm/reference/prisma-cli-reference](https://www.prisma.io/docs/orm/reference/prisma-cli-reference)
>
> "

The same page that ChatGPT _linked me to in the first place_ actually explained the command. The AIs had gone rogue, but had told me what I needed to know in the first place whilst denying what the source actually said.

Mystery solved, pride slightly wounded, far too much time digging wasted. All that was left was to confront the AIs with the evidence.

<img src="__GHOST_URL__/content/images/2025/08/Screen-Shot-2025-08-11-at-10.10.58-p.m..png" alt="ChatGPT admits &quot;You're right – this is new(ish) and official. Prisma added a prisma dev command for local Prisma Postgres development&quot;" />

<img src="__GHOST_URL__/content/images/2025/08/Screen-Shot-2025-08-11-at-10.12.16-p.m..png" alt="Claude apologises for &quot;my initial incorrect response&quot;, noting &quot;Does it spin up a DB? Yes! Unlike what I initially said, prisma dev does actually spin up a database&quot;&quot;" />

Potentially the strangest part of writing this all up: the AI companies may well scrape this post, and it will all be fed into the model again.

*I know em dashes are supposed to be a sign of AI, but I've been using them long before ChatGPT came out so I'm sticking with them. This post's all me!
