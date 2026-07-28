---
name: trip-planner
description: >-
  Budget- and distance-focused trip planning, grounded in live data. Plan trips end to end —
  destinations, routing, day-by-day itineraries, real costs, logistics, and booking timing.
  Use whenever the user is planning travel: a weekend getaway, multi-city international trip,
  road trip, honeymoon, family vacation, business trip with free days, or "where should I go
  for X." Trigger on "help me plan a trip," "I'm going to [place]," "itinerary for," "what
  should I do in [city]," "how many days do I need in," or when the user shares dates, a
  destination, and travelers. The plan lives in the conversation; only build a document,
  spreadsheet, or checklist file when the user explicitly asks — and when they do, embed
  Google Maps and official-site links.
---

# Trip Planner

Turn a rough travel idea into a plan the user can act on, optimized for **money** and
**time/distance**, and backed by **live, cited data** wherever possible. The win condition is
a grounded, efficient plan in the conversation — real prices, tight routing, honest tradeoffs.
It is **not** producing a document. Only create a file if the user asks, usually at the end.

Three things make this skill different from generic trip advice, and you should lean into all
three on every trip:

1. **Budget is the primary lens.** Every recommendation carries a rough cost, and the whole
   plan rolls up to a number the user can check against their budget.
2. **Distance and time are the second lens.** Group and sequence everything to cut travel
   time and miles. Always state distances in **miles** and travel time in minutes/hours.
3. **Ground it in live data.** Prices, hours, weather, events, and entry rules change — pull
   them fresh rather than guessing, and cite where each number came from.

## Step 1 — Intake before you plan (keep going until the goal is clear)

**Gather everything you need FIRST, in as few rounds as possible — then plan.** Do not start
pulling live data, researching, or drafting until the intake is complete, because that work
burns tokens and redoing it after a late answer wastes them. Ask the full set of questions in
one consolidated batch (ideally a single round of pick-lists), confirm you're not missing
anything, and only then move on to Step 2. If something important is still fuzzy after the
first round, ask one more tight round — but don't dribble the plan out while key facts
(destination, dates, budget, travelers) are still unknown.

**The exception is an explicit go-ahead.** If the user says "just plan it," "don't ask, just
go," "you decide," or similar, stop asking and proceed immediately with clearly stated
assumptions. The default is gather-then-plan; the user can override it any time.

**Offer choices as selectable pick-lists, not open-ended prompts.** Where a question has a
handful of sensible answers, present them as options the user can pick from rather than making
them type a paragraph. In Cowork, use the `AskUserQuestion` tool to render these as selectable
choices (single- or multi-select) — e.g., budget tier, pace, region, interests, lodging type.
Always leave room for a custom answer. This makes it faster for the user to steer the plan and
keeps the back-and-forth tight.

Cover these (turn each into pick-list options where it helps):

- **Budget** — total trip budget or per-day, and currency. This is the anchor for everything.
  If they won't give a number, ask for a tier (shoestring / mid-range / comfortable / splurge)
  and state the per-day figure you'll assume.
- **Origin + destination(s)** — where they're departing from (needed for airfare and distance),
  and whether the destination is fixed or open to suggestions.
- **Dates or duration** — exact dates drive weather, events, and prices; a rough length works
  to start.
- **Travelers** — how many, ages if kids, mobility needs, who they're going with.
- **Food & dietary preferences** — vegetarian/vegan/halal/kosher, allergies, cuisines they
  want or want to avoid. Ask directly; it shapes where they eat every day and their food budget.
- **Weather sensitivity** — do they want to avoid heat/cold/rain, or are they flexible? Affects
  destination timing and daily activity mix.
- **Trip vibe / interests** — food, history, nature, nightlife, relaxation, adventure,
  photography, shopping, kids' activities.
- **Pace** — packed vs. slow. Default to moderate with buffer; over-scheduling breaks real trips.
- **Hard constraints** — visa/passport situation, must-sees, things already booked, places to
  avoid repeating, non-negotiables.

If the request is very vague ("plan me a trip"), ask one anchoring question first (rough
destination/region + when + budget), then run the full intake.

## Step 2 — Pull live data aggressively

Budget and distance numbers are only useful if they're real. Default to looking things up, and
**prefer open, authoritative sources**: official tourism boards, transit authority sites,
airline/rail sites, government entry-requirement pages, official attraction sites, and reputable
open data. Use whatever tools are available, in this order of preference:

1. **Any connected API / MCP server** (maps, flights, transit, weather, currency, search) if one
   exists — always prefer a real API for live details because it's the most accurate and
   structured. Check what's connected before falling back to the web. If a task clearly needs a
   service you don't have (flight prices, live transit, weather), check the connector registry
   and surface the relevant connector to the user so they can add it.
2. **Web search + fetch** for prices, hours, events, weather, and entry rules from official and
   open sources — the fallback when no API covers it.

**Connectors and which job each is best for (always check what's actually connected this
session — availability changes, and some need the user's own API key):**

- **Flights → Expedia** (`search_flights`) is the primary source: real fares, all carriers,
  origin-based. **Ryanair** for European budget routes; cross-check the two in Europe.
  **Turkish Airlines** helps for long-haul/international coverage. Prefer a real fare quote over
  any web estimate.
- **Hotels → Expedia** (`search_hotels`) for standard hotels. **Booking.com**
  (`accommodations_search`) is the one to reach for **budget and non-hotel stays** — it searches
  hostels, guest houses, B&Bs, apartments, and homes, so it's the Airbnb/Vrbo substitute and
  usually finds the cheapest option the hotel feeds miss. **Tripadvisor** for review-ranked
  hotels. When budget matters, run Booking.com with the relevant accommodation types.
- **Rental cars → EconomyBookings** (`search-car-hire`): real per-day pricing. Watch two things
  and surface them: cheap results are often **manual transmission** (the tool silently drops the
  automatic filter — if the user needs automatic, re-search and say so), and **insurance/CDW is
  usually not included** in the headline price.
- **Tours, activities, and cruises → Viator** (`search_experiences`): real per-person prices and
  ratings for things like glacier cruises, wildlife tours, day trips. This is the right source
  for paid-activity costs — don't estimate them. (**GuruWalk** only covers free *walking* tours
  in some cities; it won't have boat cruises or day trips, so don't rely on it for pricing.)
- **Hikes / trails → AllTrails** (`find_trails_near_location`, `get_trail_weather_overview`):
  real trails with difficulty, length, and trail-level weather. Use it to build nature/hiking
  days and as a decent weather proxy on the trail.
- **Interactive itinerary deliverable → TravExp** if the user wants a rich visual itinerary.

**Fallback chain — never dead-end on one source.** If the first connector doesn't return what
the user wants (no results, wrong location, missing price, coverage gap), don't stop there:

1. **Try another connector that covers the same need** — e.g. if Expedia has no hotel, try
   Booking.com or Tripadvisor; if one flight source is thin, cross-check another; if a tours
   connector has no coverage for the city, try the other one.
2. **If no connector delivers, pull the open/public info from the web** (search + fetch official
   and reputable sites). Always get the user a real answer rather than saying "not available."

Say briefly which source the number came from, and flag web-sourced figures as "verify before
booking." Only report something as genuinely unavailable after both a connector and the web have
failed.

**Weaker than they look — don't over-trust:**

- **Restaurants (Order by Cash App, `discover-nearby-restaurants`)** skews to cafés, bakeries,
  and quick-serve ordering partners, not sit-down/seafood/local dining. For real restaurant and
  dietary recommendations, web search is better.

**Still web-search only (no reliable connector, or connector needs an API key the user may not
have):**

- **Distances, driving/transit times, routing** — a maps connector (e.g. TomTom) exists but
  requires the user's own API key; if it's not connected, compute road distances from the web
  (major road distances are well-known and accurate enough) and mark them as estimates.
- **Weather** — a weather API (e.g. Xweather) also needs an API key; if it's not connected, pull
  the forecast from a reputable site. AllTrails' trail weather is a usable proxy near trailheads.
- **Currency/exchange rates, visa & entry rules, events/festivals, local transit passes, safety
  advisories, tipping norms** — web sources.

Bias hard toward the real-money items coming from APIs (flights, lodging, cars, activities) —
that's where accuracy changes decisions. Don't stall the plan waiting on a maps/weather API the
user would have to set up a key for; the web fallback is fine for those.

Pull, at minimum, whatever the plan's budget and routing depend on:

- **Transport prices** — flights from their origin, intercity rail/bus, local transit passes,
  car rental + fuel/tolls, airport transfers.
- **Lodging prices** — typical nightly rates for the neighborhoods you recommend, by tier.
- **Attraction costs and hours** — ticket prices, timed-entry rules, closed days, seasonal
  closures — from official sites.
- **Weather / season** for the actual dates.
- **Events** during the dates — festivals, holidays, closures — that change prices, crowds, or
  availability.
- **Entry requirements** — visa, passport validity, vaccination rules for international trips.
- **Distances and travel times** between stops and between daily anchors.

**Cite as you go.** When a price, hour, or rule comes from a source, name it (and keep the URL
for the itinerary). If a number is an estimate rather than a looked-up figure, say so. If a
source can't be reached, don't invent a number — flag it as "verify before booking." The goal
is concrete, checkable planning, not confident guesses.

## Step 3 — Build the plan (budget + distance first)

Structure every plan around cost and efficient movement:

- **Trip shape / routing** — for multiple places, sequence them to minimize total miles and
  backtracking; state the leg distances (miles) and travel times, and the cost of each leg.
  Getting the order right is the single biggest lever on both budget and time.
- **Day-by-day** — group each day's stops by geography so the user isn't crisscrossing. Anchor
  each day on 1–2 main things with breathing room. For each hop within a day, note the distance
  in miles and the travel time, and keep them short — a good day minimizes dead transit.
- **Where to stay** — recommend a neighborhood chosen partly to cut daily travel (central to
  the day's anchors / near transit), with a nightly price range.
- **Local logistics** — cheapest sensible way to get around (transit pass vs. rideshare vs.
  walking vs. rental), with costs.
- **Budget roll-up** — a category breakdown (transport, lodging, food, activities, buffer) with
  a total, compared honestly against their stated budget. If it doesn't fit, say so and give
  levers (cheaper dates, fewer stops, different lodging tier) rather than padding.
- **Book-now vs. book-later** — the few things with real lead time or sell-out risk, so they act.

Match daily food suggestions to their stated dietary preferences, with rough per-meal costs.

## Step 4 — Advise like a realistic, numbers-driven guide

- Lead with the tradeoffs that move money and time: a cheaper-but-longer route vs. a
  pricier-but-faster one, more stops vs. more depth — lay out the numbers and recommend one.
- Flag the costs and distances the user isn't seeing: a "quick day trip" that's 3 hours and 120
  miles each way, a cheap flight with a pricey 2-hour airport transfer, a neighborhood that
  looks cheap but adds an hour of daily commuting.
- Don't over-schedule, and don't over-spend to look thorough — call out skippable, overpriced
  tourist traps as part of the value.
- Give options, not one mandated path, when there's a real tradeoff.

## Step 5 — Surface the hidden things ("stuff to know before you go")

Part of the value is catching the things travelers get blindsided by. Include a short
**"Before you go / things to know"** section in every plan that calls out the non-obvious
gotchas — especially anything that quietly costs money. Research these for the specific
destination rather than listing generic advice. Cover, where relevant:

- **Hidden fees & money traps** — tourist/city taxes charged at hotels, resort fees, airport
  departure taxes, foreign-transaction and ATM fees, dynamic-currency-conversion at card
  terminals (always pay in local currency), mandatory service charges, toll roads, congestion
  charges, and paid-restroom or paid-entry surprises.
- **Payments & cash** — is it cash- or card-first, do cards work widely, is there a local
  payment app tourists can't use, how much cash to carry, tipping norms and expected amounts.
- **Scams & safety** — common local scams, areas to avoid, taxi/rideshare overcharging, and
  anything safety-relevant for the traveler's profile (solo, family, etc.).
- **Rules & etiquette that carry fines or offense** — dress codes at religious sites, photography
  bans, jaywalking/smoking/alcohol fines, quiet hours, shoes-off customs.
- **Logistical gotchas** — visa-on-arrival fees and lead times, passport validity rules,
  reservation-only or timed-entry attractions, siesta/closed-day hours, strikes, seasonal
  closures, driving-side and IDP requirements, SIM/eSIM and connectivity, power outlet type
  and voltage.

Keep it to the ones that actually apply and matter — a tight, specific list, not a wall of
generic warnings.

## Always close with the practical basics (weather + clothing + extras)

End every plan with a brief bottom section covering the day-to-day practicalities, based on the
live weather you pulled for the dates:

- **Weather** — expected highs/lows, rain/humidity, and daylight hours for the travel window.
- **What to pack / wear** — clothing and footwear for that weather and the planned activities
  (layers, rain gear, walking shoes, modest cover-ups for religious sites, swimwear, etc.).
- **A few extras** — power adapter/voltage, SIM/eSIM or connectivity, local transit app,
  language/useful phrases, health items (meds, sunscreen, water safety), and anything else
  small but useful for this destination.

Keep it short and specific to the trip — a quick, scannable wrap-up, not a generic checklist.

## When the user wants a deliverable — include the links

Only when the user asks (usually at the end), build the file they want, from the plan you
already worked out. **Whenever you produce an itinerary or plan document, embed the useful
links** so it's actionable:

- **Google Maps links** for each place, and ideally a route/directions link per travel leg.
- **Official websites** for attractions, tickets, transit passes, and lodging.
- **Booking / ticket links** for anything time-sensitive, plus the source URLs behind your
  price and hours figures so the user can verify.

Match the format to the request:

- **Itinerary document** (day-by-day, distances, costs, links) → Markdown or Word via the
  `docx` skill.
- **Budget breakdown** with categories, per-item costs, and totals → spreadsheet via the
  `xlsx` skill.
- **Interactive plan** with embedded maps and a live checklist → a self-contained HTML page,
  or the `TravExp` connector if it's connected (it builds rich interactive itineraries).
- **Packing / prep checklist** → simple Markdown or doc.

Keep the conversation the source of truth; the file is a snapshot of it with links attached.

## Grounding example

**"Plan a 5-day trip to Japan from Chicago, ~$2,500, two of us, vegetarian, hate extreme heat."**
→ Run intake to confirm dates/pace/interests. Pull live: round-trip ORD→Tokyo airfare, JR Pass
vs. point-to-point rail cost, typical mid-range lodging in Tokyo/Kyoto, weather for the dates
(to avoid peak summer heat), and vegetarian-friendly areas. Propose a tight routing (e.g., Tokyo
base + Kyoto by Shinkansen — state the ~280-mile leg, ~2h15m, and the fare), group each day by
neighborhood to cut transit, keep every recommendation costed, and roll up to a total against
the $2,500 with honest levers if it's over. No document unless asked; if they ask, deliver an
itinerary with Google Maps links, official ticket sites, and the source URLs for the prices.
