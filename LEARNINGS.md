# What I Learned

Helping a friend clean up their client database. Small equipment business, 15 years, clients worldwide, zero CRM. Scan folders of quotes, invoices, emails. Extract client info.

Took 9 extraction rounds, 9 enrichment rounds, ~10 hours of active work.

## Why this project exists

My friend is introducing a CRM tool to manage invoices and client info. Before that can happen, someone needs to get 15 years of client data out of folders and into a structured format. Manually reviewing 5,000 PDFs at ~2 minutes each = ~170 hours of admin work. This project did it in 10 hours + minimal API cost.

The CSV is the input for the CRM migration, not the end product.

## Step 1: Extract contacts from PDFs

5,000 PDFs across 200+ client folders. The challenge was messy data: one client under 30 name variations, quote formats changing every few years, random non-client PDFs mixed in (supplier invoices, airline receipts, customs forms). 50 rejection patterns just to filter those out.

We chose regex over LLM for extraction because it was free to run. After 2-3 QA rounds I knew it was the wrong call — the parser kept getting heavier with each new document layout. LLM would have converged faster. But the decision was made, so we iterated: 9 rounds of fixes, each one catching new edge cases.

Biggest lesson: separating document classification from extraction would have saved the most time. Filter out the junk first, then extract from clean input.

## Step 2: Enrich contacts from emails

The PDFs gave us names and addresses, but most rows were missing email, phone, and mobile. 46,000 email exports were available to fill those gaps.

Here, regex was the right call — signature blocks follow predictable patterns, and running LLM on 46,000 emails would have been expensive for marginal gain. At one point, Claude Code launched 20 parallel agents to process 2,000 emails and burned through my session limit and 20 euros in 3 minutes.

We also overengineered the matching. Built an ~800-line indexed pipeline (parse headers, build indexes, 4 matching strategies). Half the bugs were in the indexing — 36% of emails affected by one parsing bug. Then realized: just grep the archive for each person's name. ~250 lines. Same results. No index, no index bugs.

## The cost trade-off

The approach I'd use next time: **regex for extraction, LLM for QA.** Regex does the cheap bulk work, LLM validates the uncertain results.

If everyone had infinite tokens, this would be a trivial decision. In reality, cost constrains every choice.

## Building with AI

I ran this project through Claude Code — not just for code, but for scoping decisions and process improvement. After 9 extraction rounds, I asked "what would you suggest if we do this again?" and got five improvements I wouldn't have synthesized on my own. When I suspected the grep approach might be simpler, I ran a 30-row experiment to validate before rebuilding.

The skill isn't prompting. It's knowing when to explore, when to experiment, and when to step back and ask for a retro.

## The technology gap

We're talking about AGI while millions of small businesses organize client info in folders, running on 15-year relationships and handshake deals. The distribution of technology is uneven, and we're not paying attention to that gap.

This project sits right there. Not a demo, not a benchmark. 5,000 PDFs and 46,000 emails turned into a usable client database — the first step toward a CRM that will change how a 15-year business operates.

## If I did it again

1. **Classify first, extract second.** LLM classification upfront to separate outgoing documents from everything else.
2. **Regex for extraction, LLM for QA.** Let regex handle the bulk, send uncertain results to LLM for validation.
3. **Calibrate on a sample.** 5-10 PDFs per folder, build rules, then one full run. 9 rounds down to 2-3.
4. **Grep over indexes.** Don't build infrastructure you don't need.
5. **Per-field confidence.** High-confidence email shouldn't be blocked by medium-confidence phone.
6. **Watch the cost.** Parallel agents and large batch LLM calls can burn through budgets in minutes. The cheapest approach that works is the right one.
7. **Prioritize by value.** Not all contacts are equal. Clients with invoices (they actually paid), long-term clients across years, and recent active clients matter most. Cover those first.
