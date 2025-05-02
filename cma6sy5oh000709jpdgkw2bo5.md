---
title: "Vampiro.Life — Build in Public: Entry Two 🧛‍♀️"
seoTitle: "Vampiro.Life: Public Build Journey 2"
seoDescription: "Vampiro.Life devlog: Skill checks, Disciplines UI, NPC updates, and quest challenges in building an AI-powered Vampire experience"
datePublished: Fri May 02 2025 13:00:53 GMT+0000 (Coordinated Universal Time)
cuid: cma6sy5oh000709jpdgkw2bo5
slug: vampirolife-build-in-public-entry-two
tags: postgresql, rpg, vite, build-in-public

---

*Rolling dice, slinging disciplines, and fixing memory one bug at a time.*

Hey everyone 👋

Time for another devlog update from the shadows. Progress on *Vampiro.Life* continues slowly but surely. Between support shifts at Supabase and late-night coding sprints, the project keeps moving toward a more complete AI-powered Vampire: The Masquerade experience.

## 🎲 Skill Checks Are Live (With Willpower!)

One of the most exciting updates: **skill checks are now functional!** You’ll see the dice rolls clearly and even have the option to **reroll using Willpower**, just like in the tabletop rules. For folks less familiar with the VTM dice system, we also display a broad result (e.g., success, messy success, bestial failure) so it’s easy to understand without knowing all the mechanics.

This was a core piece needed for gameplay to feel authentic. Now you can gamble your humanity with a click.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1746033319189/8d7806bd-cb6e-4b0d-91dd-57ed7a1e6a47.png align="center")

## 🧠 Disciplines UI

Another piece of the puzzle: **we now have a working UI for Disciplines**, with contextual info to help players understand what each ability does. The goal is to make Disciplines feel powerful and thematic, not just another menu of buttons. Still early days here, but it’s playable and already adds a lot to the experience.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1746032923721/544ec54a-c7fd-4af2-887d-ed633470278d.png align="center")

## 🧭 Refactored NPC Component (Now with POIs!)

The **Nearby NPCs** system got a full refactor and now also displays **Points of Interest** (POIs) to make the map and city feel more alive. You can get a quick snapshot of who and what is around you without needing to dig through menus or prompt the AI every time.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1746033246094/0faf3dc0-ef99-448d-8be6-3049366d941c.png align="center")

## 🔧 Fixes & Tweaks

* **Quest System**: Work has started, but it's proving tricky to get Gemini to generate quests that are actually playable. It’s not fully functional yet, but the bones are there.
    
* **Back Button on Subscription Page**: Yes, finally. Clicking into the subscription page no longer traps you in UI limbo.
    
* **Memory Management**: Fixed both the backend handling and the UI related to memory state. The AI now forgets a bit less randomly and more logically.
    
* **UI Color Update**: Text messages are no longer a weird shade of blue. They now match the darker, more gothic aesthetic we’re going for.
    

## 🔮 Next Steps

Here’s what I’m working on next:

* **Improve the Quest System** so that Gemini can reliably generate quests players actually want to pursue. This might involve more prompt engineering or extra fine-tuning.
    
* **Revamp the World-Building Experience**. The current setup works but isn’t intuitive. The plan is to make it smoother, editable, and easier to navigate without breaking the immersion.
    

---

That's it for this update. As always, if you’re into AI-powered storytelling, RPG systems, or just want to see how an indie hacker builds something weird in public, feel free to follow along or reach out.

Stay hungry (for blood),  
Rodrigo 🦇