---
title: "A Realistic Look at Renovating an Old Carbureted Dodge Van"
date: 2026-08-08
draft: false
tags: ["dodge-b350", "van-build", "engine-repair", "freeze-plugs"]
---

If you came here for a finished van with a solar array on the roof and a fridge humming in the corner, this isn't that post. This is the honest middle: a 1987 Dodge B350 that's mostly gutted, a coolant leak I chased with my own pant leg, and a freeze plug I managed to start backwards. Most of what follows is planning and research, not installs — because on an old carbureted RV, getting the plan right is most of the work, and I've already learned a couple of things the hard way.

## What I'm actually working with

It's a 1987 Dodge B350 that Xplorer Motor Home (Frank Industries, under Chrysler Canada) turned into an RV from the factory. Under the doghouse it's a 360 — carbureted, no computer, no injectors. That simplicity is half the reason I bought it and half the reason parts sourcing is going to be an adventure.

The previous owner gutted a lot of it before I got it: the sink, fridge, oven, and interior flooring are gone. What's left is the bed, the cabinets, the closet, and a bathroom shell with no toilet. The hot water heater and the fresh and black tanks are all physically in the van, but nothing is plumbed — not a single line run. There's a window AC unit shoved into a side door running off a small generator. And I tow an old pop-up trailer behind it that already has a working stove, oven, and sink, which turns out to matter a lot for how I'm scoping the van itself.

So: a rolling shell with good bones and a long list. Here's how I'm thinking about it.

## The one thing I've actually wrenched on: the leak

Before any of the build-out, the van has to be safe to drive. Top of that list is a coolant leak.

Here's how I found it, which is also a warning: I drove it with the doghouse off, and partway down the road I felt water spray my leg. I looked for a hose that could do that and couldn't find one. It was coolant — you can smell it, and you can see the color. No visible hose plus a spray coming from down low points somewhere annoying: the freeze plugs, and specifically the rear ones tucked between the block and the bellhousing. My best guess right now is the rear driver's side.

But I want to be straight about this, because it's the whole point of writing these down honestly: **I have not confirmed it yet.** Spray direction lies — the fan can fling coolant from a low weep and throw it somewhere completely different. Before I commit to anything I'm going to do a cold pressure test on the cooling system and trace the actual wet trail and rust staining back to the source. That matters here more than usual, because the rear plug is *not* a "while I'm under there" job — getting at it properly means separating the transmission from the engine. I'm not pulling a trans for a two-dollar part on a hunch.

One more thing that kept me from doing something dumb: don't diagnose this with the doghouse off and the engine running next to your leg. In this layout the engine bay opens right into the cab, so exhaust heat and fumes have nowhere to go but at you. Chock the wheels, set the brake, and do it parked.

### The freeze plug I started backwards

While I was in there I figured I'd get ahead and start an easier, accessible plug on the front passenger side. It wouldn't press in. I fought it for a bit before I realized the problem was me: I'd started it backwards. On a freeze plug the domed, convex side has to face **out**, and I had the cup facing the wrong way — it was never going to seat like that. I didn't drive it in far, so no harm done, but that's exactly the kind of gotcha that costs you an afternoon if you don't catch it.

The rules I should've been following from the start, and will be when I do these for real:

- Clean the bore back to bare metal without enlarging it.
- Measure with calipers before ordering — the side plugs are roughly 1-1/4" and the rear deep-cup plug is the bigger one, around 1-5/8".
- Domed side faces **out**. (The one I got wrong.)
- Drive on the plug's outer rim only — never with a tool that nests down inside the cup, because that squeezes the outside diameter smaller and you'll be back in here leaking again.
- Thin film of sealant, nothing more (Permatex Aviation Form-A-Gasket #3 or High Tack).
- Seat it square, flush to about 0.02–0.03" recessed.

And the reason people say to do all of them while you're in there: the pattern on these small-block Mopars is driver's-side plugs go first, then the rears, then passenger. If you have to separate the trans to reach one, you do not want to do it twice.

## Scoping a $3,000 build without lying to myself

My parts budget is $3,000, no hard deadline, buying in phases as I go. The cheapest decisions I made weren't purchases — they were subtractions:

- **No fridge.** Ice chests instead, maybe a cheap countertop ice maker later that runs off the inverter.
- **No oven.** The pop-up trailer already cooks, so the van doesn't need to.
- **Keep the existing bathroom sink** — it's fine.
- **Toilet is a porta-potti,** not a plumbed cassette.
- **The window AC stays on the generator,** not the battery bank. A 5,000–8,000 BTU unit pulls 500–900W continuously, which would flatten any battery I can afford in a couple of hours.

Every load I *don't* add is a battery I don't have to buy. That's the throughline of the whole budget.

## Heat is the real enemy (and the Reflectix mistake I nearly made)

This thing lives somewhere hot most of the year, so cooling and insulation matter more than anything fun. Two things I ran into while planning:

The onboard AC is an **R12** system. You can't just go buy R12 anymore. The realistic path is an R134a retrofit, and after this van's been sitting, that's not a recharge — it's a compressor, a new receiver/drier, a full flush of the old mineral oil, PAG oil, new O-rings, and a proper vacuum leak-down before charging. R134a doesn't shed heat as well as R12 did, so a better condenser is on the list too if I want to actually beat the heat rather than just get it blowing. First job before spending a dime on refrigerant, though: check the evaporator box for rodent nests, because a van that's sat for years is prime real estate and chewed wiring in there is a fire risk.

The bigger near-miss was insulation. My plan was to buy rolls of Reflectix and cover every interior surface. That's a well-documented mistake and I'm glad I caught it before installing anything. Reflectix only hits its advertised R-value with a maintained air gap of about 3/4" or more on the reflective side. Pressed flat against van metal it's closer to R-1. Worse, it's foil-faced, which makes it a vapor barrier — glued straight to bare steel it traps moisture against the metal with nowhere to dry, and you get hidden condensation and rust behind material you can't inspect without tearing the interior back out. On a 38-year-old steel body, that's a threat to the vehicle itself, not just comfort.

So the plan changed: **polyiso rigid board** on the walls and ceiling (glued down, edges sealed) — which conveniently is the one rigid foam that performs best in warm climates — and Reflectix only on the windows, where everyone agrees it works, ideally with a thin foam core sandwiched between two layers like a reflective windshield shade.

## The power system, on paper

Because I subtracted the big loads, the electrical plan stays modest: a daily budget of roughly 300–450 Wh for lights, the water pump, charging, and occasional ice maker use. The shopping list is 200–300W of solar, a single 100Ah 12V LiFePO4 battery, an MPPT controller, and a small pure-sine inverter — one with a real standby mode, because a cheap always-on inverter can waste 70–140 Wh a day just idling, which is enormous against a 400 Wh budget. I'm planning to upsize the main battery wiring now and fuse the positive within 7" of the terminal with a 150–200A ANL/MEGA, so adding a second battery or a DC-DC charger later doesn't mean rewiring everything.

None of this is bought yet. This is the list, not the receipt.

## Steering, the other safety item

The steering feels worn and I suspect the tie rod ends. The annoying wrinkle: the aftermarket catalogs split B350 fitment by front axle rating, and the labeling is inconsistent — I've seen what looks like the same SKU called an outer tie rod in one place and an inner in another. My front axle is rated 3,600 lb, which should put me in the standard-duty bucket, but I'm not trusting a part number off a listing for a *steering* component. I'll either VIN-match through Moog or RockAuto's own lookup or pull the old ends and match them at the counter. Worth writing down: that 3,600 lb front rating will come back for ball joints, wheel bearings, and some brake parts too.

## Why the order matters

The plan, in the order I'm going to do it: safety and drivability first — the leak and the steering — because I'll be towing a loaded pop-up with water and batteries aboard and I don't want to sort that out at highway speed. Then the infrastructure decisions, power and plumbing, because every fixture keys off what those can support. Then the fixtures themselves. Then flooring dead last, so I'm not ripping up a fresh floor to chase that same coolant leak I still haven't confirmed. The pop-up trailer reno runs in parallel the whole time, since it shares no systems with the van.

Right now the van is mostly a plan and one backwards freeze plug. But the plan is the part that keeps a cheap build from quietly turning into an expensive one. Next up is the least glamorous and most important step: actually confirming that leak before I separate a transmission for it. I'll write that one up when I do.
