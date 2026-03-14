# Open Brain Extensions: Giving Your AI Memory System Hands and Feet

> **TL;DR:** Thousands of people built Open Brain — a personal database with vector search that gives every AI tool persistent memory. Now the question is what to do with it. The answer: build extensions that create a "shared surface" where both you and your agent operate on the same data through different interfaces. Six use cases — from household knowledge to job hunting — demonstrate four design principles that make agent-powered systems genuinely useful: time-bridging, cross-category reasoning, proactive surfacing, and a clear judgment line between what the agent surfaces and what the human decides.

**Source:** "Your Open Brain Can Think — Now It Needs Hands" (YouTube + Substack companion article)
**Speaker:** Nate B. Jones, creator of Open Brain

---

## The Most Common Question After Building Open Brain

"I built it. Now what?"

That has been the most common message since the Open Brain guide shipped. People are not writing in with bug reports or setup questions. They are staring at a working personal AI memory system and having no idea what to do next.

The infrastructure works. The database is live. The agent reads from it, writes to it, remembers what you said last Tuesday. And now you are sitting there realizing that the hard part was never the build. The hard part is knowing what to put in it.

The usual advice does not help much. "Capture your thoughts." "Ask it questions." That is like handing someone who just built a workshop full of beautiful tools a piece of sandpaper and telling them to get sanding.

What changed for Jones was building extensions on Open Brain where both he and his agent could see and act on the same data. A maintenance log where the agent catches that a warranty is about to expire based on something a technician mentioned eighteen months ago — and he pulls up the same log on his phone when the repair tech asks what was done last time. A family schedule where the agent cross-references both parents' calendars and every kid's activity simultaneously — and Sunday night the family is looking at the same view on a tablet planning the week. A job search dashboard where the agent spots that a warm introduction is going cold while you are drowning in eleven other workstreams — and you see the whole pipeline at a glance over coffee.

The pattern underneath all of these is the same: a shared surface with two doors. Your agent enters through one. You enter through the other. Both sides read the same data, both sides write to it, and each one does what it is best at.

## The Two-Door Principle: Why a Chat Window Is a Keyhole

Right now, when you use Open Brain through a chatbot — Claude, ChatGPT, OpenClaw — you are chatting through a keyhole. It is text-based. You can ask one question and get one answer. You cannot see the landscape.

Ask your agent "what's on the family schedule this week?" and you get a text list. Fine. But you cannot scan it at a glance while packing lunches, hand your phone to your partner, or spot the Thursday conflict by looking at two columns side by side. A calendar app gives you the visual layer — but it cannot cross-reference your partner's work schedule against the school calendar, or flag that you promised snacks for Saturday's game but have not bought them yet.

You need both. The agent reasoning across the data, and you seeing the data with your own eyes. The agent writing to the data when it catches something in conversation, and you writing to it when you are standing in the kitchen and need to add something fast. Neither interface alone is enough.

## The Table as Shared Surface

The architecture is deliberately simple. Start from the ground up with a structured table in your Open Brain database — the same Supabase instance you may have already built. Then add a light visual interface over the top.

The table in the database is the shared surface. Your agent can read it through MCP the same way it already reaches Open Brain. You read it the way a human would want to — through a simple view, a web page, a mobile bookmark, maybe eventually an app that reads from and writes to the same table.

No new platform. No additional service. No middleware sitting between you and your data. The table remains the single source of truth, and both you and your agent have direct access to it through different doorways.

When your agent writes to the table during a conversation, the entry is immediately visible the next time you pull up the view on your phone. When you update a field on your phone, the change is immediately there the next time your agent queries the table. Both sides are reading and writing to the same rows. The consistency is architectural — it is built in.

Because it is just a table with a view, you can build it for anything. A household knowledge base in one table, one view. A maintenance tracker in a table and a view. A job search dashboard in a few tables and a few views. The pattern is the same and the pattern scales.

## Building the Visual Layer

The process for creating the "human door" is straightforward:

1. **Talk to your AI.** Describe what you want: "I want a mobile-friendly view of my maintenance table. It should show every appliance in the house, the warranty date, and the last service date. Highlight anything expiring in the next 30 days." Your AI generates a small web application, and because it knows your Supabase data schema, it builds with high fidelity.

2. **Iterate in conversation.** Adjust the layout, change what is highlighted, make it exactly the way you want. Go back and forth until it looks right.

3. **Deploy for free.** Upload the application to Vercel — a free hosting service — and get a live URL. A real web address you can open on your phone, your partner's phone, a tablet at the kitchen table. Bookmark it on your home screen and it behaves like an app. No app store, no subscription — just a page on the internet that talks to your database.

Now both doors are open. The agent reaches your table through MCP. The visual reaches the same table through a database connection. One source of truth, two interfaces, each built for what its user does best.

For those who want a shortcut, connecting Supabase to a service like Lovable is another option. But the open-source path requires no middleman and no recurring cost.

## Use Case 1: Household Knowledge Base

Every household runs on information that lives absolutely nowhere. Paint colors. Kids' shoe sizes. The plumber you used two years ago. The WiFi password for the guest network. The answer always exists somewhere — in a text thread, in a receipt in the drawer, in someone's memory — and you can never find it the minute you need it.

Start feeding that information into a table in your Open Brain as you encounter it. You are talking to Claude about something else entirely and mention the paint color: "Hey, save this — the living room paint is Benjamin Moore Hale Navy. We got it at Sherwin-Williams back in October." Claude writes the entry. Two seconds. Done.

You could have that same conversation with ChatGPT or OpenClaw. It does not matter. The data lands in the same table. The agent-writable side captures facts in structured form from ordinary conversation. Over a few weeks, the table fills with the institutional knowledge of your household — where the keys live, what color the paint is, when you last got the tires changed.

The visual layer for this one is simple: a search bar at the top of the page where you can search for anything, plus categories of knowledge created by the agent as it categorizes entries. You start to see groupings — everything about the living room, everything about the car, everything about the kids and school — and you can dive in and understand where you have recorded your household knowledge and where the gaps are.

This is the simplest use case. But it already illustrates the design principle: all four modes — agent reads, agent writes, human reads, human writes — operating on the same data.

## Use Case 2: Home Maintenance — and the Principle of Time-Bridging

When did you last service the HVAC? Is the roof still under warranty? The dishwasher is making a noise — what did the repair tech say a year and a half ago?

Structured maintenance data: asset, purchase date, warranty expiration, service history, vendor contacts, notes. Every time a tech comes out, you log what they found and what they recommended. Takes two minutes.

Here is where the first real principle emerges. You open Claude on a Sunday morning and ask, "Anything I should deal with around the house?" Claude sees the whole timeline across every asset and catches things invisible to daily life: "The dishwasher warranty expires March 15th. The last tech noted pump wear in September. You might want to schedule service before the warranty lapses."

Stop and notice what just happened. The agent bridged an eighteen-month gap between two events — a technician's offhand comment about a pump and an approaching warranty deadline — and synthesized them into a recommendation at the exact right moment. You did not ask about the dishwasher. You did not remember the pump comment. You certainly did not have the warranty date in your head. The agent held the entire timeline and surfaced the connection because the dates made it urgent now.

No human does this. Human memory simply does not work this way. You do not wake up on a random Tuesday and think "what did that repair tech say eighteen months ago and how does it relate to my warranty expiration?" Your brain flushed those details months ago to make room for things that felt more pressing. The agent flushed nothing.

The autonomous version goes further. OpenClaw scans your maintenance data on a schedule, and Monday morning you wake up to that same insight as a notification — without asking. Same data, same table, same insight. The difference is whether you had to think to ask.

**Principle one: your agent bridges time.** Its memory does not decay. Something logged six months ago is exactly as accessible as something logged this morning. Anywhere the value comes from connecting events spread across months or years — home maintenance, medical history, financial decisions, career moves — that is your agent's territory.

The human side earns its keep too. The HVAC tech is in your living room asking what was done last time. You pull up the maintenance view on your phone and hand it to him. Service dates, parts replaced, notes. He reads it himself. A visual interface solving a problem no chatbot can, because the tech needs to scan information quickly, not have a conversation.

## Use Case 3: Kid Logistics — Seeing Across Categories

If you have children, this is the use case that will make everything else click.

Pickup is at 3:15 except Wednesdays when it is 2:00. Soccer practice moved to field B starting next week. Permission slip due Friday. Picture day is the 14th but your kid needs a haircut before then. Teacher conference is Thursday, but your partner has a client dinner that night and someone needs to swap. The older kid has a birthday party Saturday but you have not bought a gift yet, and the younger kid has a playdate at the same time in the opposite direction.

Every parent holds this graph in their head and drops things. Regularly. It is not about being disorganized — it is that the complexity genuinely exceeds what a single human brain can track in real time while also holding a job, feeding people, and remembering to pick up the dry cleaning.

A calendar does not fix this. A calendar shows events. It does not show conflicts and dependencies. It will happily display the Thursday teacher conference and your partner's Thursday client dinner on separate calendars without ever telling you those two things are a problem. Picture day is on the 14th, but the calendar has no idea your kid needs a haircut first. The birthday party is Saturday, but nothing in the interface knows you have not bought a gift.

Your agent sees both parents' schedules and all kids' events simultaneously. You ask ChatGPT on Sunday night, "Walk me through next week — any collisions?" and it comes back: "You signed up for the Thursday conference, but your partner has the client dinner. Wednesday at 4pm is open for both of you — want me to check if the school has that slot?" Cross-referencing two people's constraints, finding a conflict neither noticed, scanning for alternatives that work for both, and proposing a resolution — all because you asked one broad question and the data was structured for reasoning.

**Principle two: your agent sees across categories.** When multiple types of structured data exist in your Open Brain — schedules, tasks, contacts, logistics — the agent cross-references them in ways no human would do manually. The power is not in any single table. It is in the connections between tables.

The visual layer matters just as much. Sunday night, both parents at the kitchen table. The family view is on a tablet — both schedules, all activities, conflicts highlighted, undone tasks flagged. You can see the week as a single picture and plan together while pointing at the screen. That is fundamentally different from asking an agent to read you a list, because planning is visual — it requires scanning and comparing and making quick decisions while both of you are looking at the same picture.

## Use Case 4: Meal Planning — The Collision of Four Datasets

Not recipes. The problem that actually defeats families week after week is the collision of what everyone eats, what is in the house, what the weekly schedule demands, and what needs to go on the grocery list.

Your agent holds all four simultaneously. You are talking to Claude about the week ahead — not about food, just about what is coming — and it volunteers: "You haven't done fish in three weeks. Thursday is soccer night so you need something fast. Last busy Thursday you did sheet-pan chicken and everyone ate it. You're out of chicken thighs. Adding it to the list, putting fish on Saturday when you have time."

You did not say "plan my meals." Claude noticed a gap because it could see the meal history, the schedule, the pantry, and the shopping list in the same pass. An autonomous agent like OpenClaw goes further — it generates the meal plan and shopping list on Sunday night without you asking, ready for you to review Monday morning.

**Principle three: the most valuable answers are to questions you did not ask.** A database waits for queries. An agent notices things proactively — gaps, patterns, deadlines, conflicts. A conversational client does this when you give it room in a broad question. An autonomous agent does it on a schedule. Design your tables for what you want your agent to notice, not just what you want to look up.

The human-readable layer is what makes this practical: standing in the grocery store on Thursday afternoon, the shopping list is on your phone, populated by the agent, organized by aisle. You grab the chicken thighs, check it off. You see the avocados look good and add guacamole ingredients on the spot. The agent sees those additions next time it looks at what is in the house.

## Use Case 5: Professional Relationships — and the Judgment Line

You have dozens of professional relationships that matter and limited ability to maintain them all. The context is scattered across emails, messages, and a memory biased toward the most recent interaction.

You ask Claude, "Anyone I've been neglecting?" and it scans the full picture: "You haven't reached out to James in four months. Last time you talked, he was worried about his team's reorg. That kind of thing typically resolves in a quarter — might be a good time to check in." Time-bridging and cross-category reasoning working together, triggered by one open-ended question.

Or OpenClaw catches the same gap autonomously and flags it in a Monday morning digest — you did not ask, but the window was closing and the data made it obvious.

But here is the fourth principle, and the most important: **agent surfaces, human decides, agent executes.** Your agent should absolutely tell you James is overdue for a reach-out and give you every piece of relevant context. It should not draft and send the email. Because the right message depends on things no database captures: the politics at his company, the tone your relationship calls for, whether this week is actually the right moment given what else is happening in your life. The agent handles memory and pattern recognition. You handle judgment. The division is clean and it is what makes the system trustworthy.

This applies to every use case. The agent notices the scheduling conflict, flags the warranty deadline, spots the pattern in your interview data. You decide what to do about each one. The system works because the division of labor is clean: the agent does what agents do well — holding vast amounts of data, seeing connections across time and categories, never forgetting — and you do what humans do well — making judgment calls, reading social situations, weighing values and priorities that cannot be quantified.

Get this line wrong and you build something you do not trust. Get it right and you build something that makes you better at everything it touches.

## Use Case 6: The Job Hunt — Where Every Principle Fires at Once

A job hunt is a dozen parallel workstreams pretending to be one activity — companies, roles, contacts, applications, interviews, follow-ups, resume versions, compensation data, and your own shifting sense of which opportunity you actually want. This is where the full architecture earns its keep.

**Cross-category reasoning:** You paste a posting into ChatGPT and say, "What do I have on this company?" It does not just read the posting. It scans your contacts, conference notes, relationship data. "You met someone from their data team at the conference in October. You noted a good conversation about distributed systems. He said they were hiring." A warm introduction instead of a cold application, surfaced by connecting data across tables you would never cross-reference manually.

**Time-bridging:** Maria offered to introduce you to a VP nine days ago. You meant to follow up. Life intervened. OpenClaw catches the gap overnight and flags it Monday morning: "That was nine days ago. The window on a warm intro is roughly two weeks." You did not ask. You did not remember. An autonomous agent running on a schedule caught what a session-based client never would — because you would have had to think to ask, and the whole problem is that you forgot. Job hunts do not die because people are unqualified. They die because warm introductions go cold while you are drowning in the other eleven workstreams.

**Proactive pattern recognition:** After several interviews, you ask Claude, "What patterns do you see across my interviews?" It reads every note simultaneously: "You keep getting energy from platform team conversations and losing it in direct-to-consumer ones. Your four strongest interviews were at companies under 200 people. You might be optimizing for the wrong company size." You would figure that out eventually — maybe after two months of scattered results. Claude sees it in two weeks because the data was structured for reasoning.

**The emotional corrective:** Job hunting is demoralizing in a specific way that most productivity tools ignore. You get ghosted after a great conversation. You bomb a final round. Your brain — wired to construct narratives from recent experience — starts telling you a story that says you are not good enough. The story is wrong, but it feels true because the most recent data point is negative.

Your agent has the actual record. You say, "I'm losing confidence — am I actually getting anywhere?" and it comes back: "You've advanced past five of seven first-round interviews, a 71% conversion rate. The two rejections were both in industries where you have no direct background — biotech and fintech. In your core domain, you're five for five. The rejections are telling you something about fit, not ability."

Data correcting a distortion your brain produces automatically under stress. Only possible because the full history is structured, queryable, and interpreted by something that is not emotionally invested in the outcome.

**The human-readable layer:** You pull up your search dashboard with coffee. The whole pipeline is visible at once — which companies are active, where you are in each process, what is due today, what is stalled. You can see the shape of your search in a way no chat window provides, because a pipeline is spatial. You need to see what is at the top of the funnel, what is in the middle, what is close to an offer.

And the judgment line holds: your agent gives you the normalized compensation comparison, the pattern analysis, the conversion data, the full pipeline view. It does not tell you which offer to take. That decision depends on what kind of work you want to wake up to every morning, which team felt right when you were in the room, what your family needs from this transition.

## The Architectural Advantage: Why This Only Works Because of Where You Built It

Everything described here depends on one architectural decision already made. Your data is agent-readable. Your AI reaches it directly — queries it, reasons about it, writes to it, connects it to other data — without scraping a UI, without an export step, without asking permission from a platform that controls the access layer.

Because the data layer is yours, any client can reach it. Claude for a deep reasoning session. ChatGPT for a quick question. OpenClaw for autonomous monitoring while you sleep. They all hit the same tables, and none of them needs the others' permission.

That is the real paradigm: conversational clients handle the **pull** — you ask a broad question and the agent reasons across your data in the moment. Autonomous agents handle the **push** — they scan your data on a schedule and surface what is urgent before you think to ask. Same database, different interfaces, each doing what it is best at. The data does not care which model is reading it.

When your schedule lives in someone else's calendar app, a human can see it. An agent cannot — not without a limited API that the platform controls and can restrict whenever it chooses. When it lives in your own database, your agent sees it the same way you do — first-class access, no intermediary, no one else's permission required.

The long-term bet is simple: every time the models get smarter, every extension you have built gets more valuable automatically. The agent reading your family schedule today is good. Next year's model will be better at spotting conflicts, anticipating needs, doing things you cannot even imagine prompting for right now. You are building on a foundation that appreciates with every model improvement, because the data is structured for reasoning, not just storage.

## The Four Principles, Named

1. **Your agent bridges time.** Its memory does not decay. Anywhere the value comes from linking events spread across months or years, that is your agent's territory.

2. **Your agent sees across categories.** The power is not in any single table. It is in the connections between tables that no human would cross-reference manually.

3. **The most valuable answers are to questions you did not ask.** A database waits for queries. An agent notices things proactively — gaps, patterns, deadlines, conflicts. Design for what you want your agent to notice.

4. **Agent surfaces, human decides, agent executes.** The agent handles memory and pattern recognition. You handle judgment. The division is clean and it is what makes the system trustworthy.

Any problem in your life that involves scattered information, events spread over time, or multiple categories that need cross-referencing — point your Open Brain at it. Create the table. Feed it data. Let the agent do what it was built to do.

## The Build Kit: Six Extensions You Build in Order

The principles are worth nothing without implementation. The [open-brain-cli repository on GitHub](https://github.com/az9713/open-brain-cli) contains six extensions, one for each use case, designed to be built in order. Each one teaches new concepts through something you will actually use — and they compound, because your CRM knows about thoughts you have captured, your meal planner checks who is home this week, and your job hunt contacts automatically become professional network entries.

| # | Extension | What You Build | Difficulty |
|---|-----------|----------------|------------|
| 1 | Household Knowledge Base | Home facts your agent can recall instantly | Beginner |
| 2 | Home Maintenance Tracker | Scheduling and history for home upkeep | Beginner |
| 3 | Family Calendar | Multi-person schedule coordination | Intermediate |
| 4 | Meal Planning | Recipes, meal plans, shared grocery lists | Intermediate |
| 5 | Professional CRM | Contact tracking wired into your thoughts | Intermediate |
| 6 | Job Hunt Pipeline | Application tracking and interview pipeline | Advanced |

The repo includes the SQL, the edge functions, the table schemas, and step-by-step instructions written for the same audience that built the original Open Brain in 45 minutes.

This is not a read-only repo. OB1 is built for community contributions — recipes, schemas, dashboard templates, new integrations — with a real contribution process behind it. Every PR runs through an automated review agent that checks eleven rules before a human admin ever sees it, and nothing ships without passing both gates.

## Companion Prompts for Getting Started

The repo also includes companion prompts designed to bridge the gap between "I built it" and "I use it daily":

- **Extension Matchmaker** — Interviews you about your actual life situation and recommends which extensions to build first, with direct links to the build guides.

- **GitHub Navigator** — Teaches you how to read, navigate, and use the OB1 repo, even if you have never touched GitHub before.

- **Extension Launcher** — Walks you through populating your first extension table with real data from your life. Adapts to whichever extension you chose — appliances and warranties for home maintenance, contacts and relationships for CRM, and so on.

- **Two-Door Audit** — Evaluates your current setup against the two-door principle and identifies where you are only using half the system.

- **Design Your Own Extension** — Uses the four principles to help you architect custom extensions for problems the six built-in ones do not cover.

- **Contribution Builder** — Turns something you built on your Open Brain into a properly structured community contribution that passes the repo's automated review.

## Key Takeaways

- A chat window is a keyhole into your own data. You need both an agent door (MCP) and a human door (visual interface) operating on the same table for the system to reach its full potential.

- The table is the shared surface and single source of truth. When the agent writes during a conversation, you see it on your phone. When you update on your phone, the agent sees it next time it queries. No sync layer, no export step, no middleware.

- Time-bridging is the agent's superpower that humans simply cannot replicate — connecting a technician's offhand comment from eighteen months ago with a warranty deadline approaching next week.

- Cross-category reasoning multiplies the value of every table you add. The meal planner checking the family calendar, the job hunt pipeline scanning your professional contacts — connections between tables that no human would cross-reference manually.

- The most valuable insights come from questions you did not think to ask. Design your tables for what you want your agent to notice proactively, not just what you want to look up.

- The judgment line is what makes the system trustworthy. The agent surfaces patterns, deadlines, and gaps. You make the decisions that require social awareness, values, and priorities that cannot be quantified. Blur this line and you will stop using it.

- Building the visual layer is accessible: describe what you want to your AI, iterate until it looks right, deploy for free on Vercel. No app store, no subscription.

- Every model improvement automatically makes your extensions more valuable, because your data is structured for reasoning, not just storage. You are building on a foundation that appreciates over time.

- You do not need an autonomous agent to get enormous value. Claude and ChatGPT, used conversationally with structured data, already deliver time-bridging and cross-category reasoning. Autonomous agents add the push — proactive monitoring — when you are ready for it.

- The six OB1 extensions are designed to be built in order, each teaching new concepts through practical use cases. Start with the one that hit you hardest, build it, and let the compounding begin.

---

*Sources:*
- *"Your Open Brain Can Think — Now It Needs Hands" — YouTube video by Nate B. Jones*
- *"You Built an AI Memory System. Now Your Agent Needs Hands." — Substack article by Nate B. Jones (March 13, 2026)*
- *"Open Brain Extensions: Companion Prompts" — Prompt Kit by Nate B. Jones*
