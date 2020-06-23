---
title: LiPo Battery Basics
tags: electronics robotics arduino
header:
  image: "/assets/2020/06-22/header1280.jpg"
  teaser: "/assets/2020/06-22/header1280.jpg"

# standard header size is 1280 x 400
# image paths:
#   publish:                        (/assets/2020/mm-dd/pic.jpg)
#   edit from _post or _templates:  (../assets/2020/mm-dd/pic.jpg)
---

Usage and charging of hobbyist LiPo batteries

<!--more-->

This is another of those "notes to myself" articles, although these basic facts and details will be useful to anyone who is new to RC cars, planes, drones, or robotics. Or, if you're like me, you might have been into those things at one time and need to refresh your memory.

Years ago, my wife and I built and flew our own drones. We started with a large DJI F450 camera drone, moved on to the "250" sized racing drones, and eventually were designing our own frames. That led me further into Arduino-style electronics. Unfortunately this can be a sprawling, messy hobby, and that wasn't compatible with relators walking prospective buyers through our house, so everything got packed away.

This blog has been entirely programming-oriented so far, but electronics and robotics fascinates be because it's where software begins to mix with the real, physical world. For quite a few years I've been meaning to get back into these hobbies, and since these things run on batteries, this article is Step One.

**Warning:** Lithium batteries can be dangerous if damaged, or improperly stored, or over-charged. A lithium battery fire is a chemical reaction that _will not_ be extinguished by a regular fire extinguisher. Never leave lithium batteries on a charger while unattended and always store them in a fireproof container. If you burn your house down (this has happened), don't come crying to me.
{: .notice--warning}

## Lithium Battery Technology

There are actually several lithium-based battery technologies which all work approximtely the same way. We only use lithium-polymer or LiPo batteries, but there are also LiFe (lithium-iron) and Li-Ion (lithium-ion) batteries.

If you've never worked with lithium batteries, it's important to understand that they can be extremely dangerous. A shorted or damaged battery can catch fire or even explode. The infamous Samsung S7 and various iPhone battery fires over the past few years are all examples of LiPo failures. You should always store your batteries in a metal case. I use a metal toolbox which is stored on the concrete floor of the garage away from anything flammable:

![toolbox](/assets/2020/06-22/toolbox.jpg)

In the hobbyist world there are certain ratings and terminology used for lithium batteries that you will need to understand. The battery in the header picture of this article is the largest I own and powered the large DJI camera drone for flights of about 10 or 12 minutes. We'll discuss the various technical aspects of that battery below. 

### Cells

Although my Venom batteries don't show cell-count, the battery in the header picture is called a "4S" battery. That means it contains four cells wired in series. A single cell is actually a battery in and of itself -- almost all batteries (of any type) are actually multi-cell batteries on the inside. 

If you have a lithium battery which doesn't show the cell count, you can count the colored wires connected to the "balance plug". That plug allows a charger or other device to monitor the voltage level of the individual cells. In the header picture, a battery tester is connected to the balance plug. 

### Voltage

Every LiPo cell has a "nominal" voltage level of 3.7V. "Nominal" just means the "at-rest" voltage of the cell. A cell can be charged to higher levels and can discharge to lower levels, but 3.7V is the reference value for comparison to other batteries. The battery in the header picture is labeled 14.8V because it contains four cells wired in series (4S). Four multiplied by 3.7V equals 14.8V.

The maximum safe charge of a single cell is about 4.2V, and the absolute minimum charge is about 3.0V. For safety, most people never discharge below about 3.2V per cell and I've known people who won't go below 3.3V. LiPo batteries are _very_ senstive to charge levels -- over-charging or over-discharging them will usually cause permanent damage, and extreme over-charging can result in fires or explosions.

In the drone world, available voltage is directly related to motor speed. Motor RPM is rated in kV which means RPM per volt. Our large camera drone has 920kV motors, which means this battery fully charged (4.2V x 4 cells = 16.8V) can spin the props at 15,456 RPM. (A device called the Eletronic Speed Controller, or ESC, pulses the power on and off to achieve different speeds during flight.)

In the header picture, the attached meter is showing the combined voltage of all cells at 16.0V, meaning the four cells are slightly below full-charge at 4.0V each. This type of meter will also show individual cell charge levels, which lets you check for damaged cells.

### Capacity

Capacity is measured in milliamp-hours, or mAh. "5000" is the mAh rating of the battery in the header picture. That's the same as 5 amp-hours, which means a healthy, fully-charged battery could continuously supply 1 amp of current for 5 hours. That's actually quite a bit of juice for such a small battery, but flight is a pretty power-intensive activity (the power-regulators on our racing drones sometimes spike to more than 125 amps, which is getting close to arc-welding levels of current, although at much lower voltages) which is why that battery only lasts about 10 minutes on a heavy camera drone.

### Discharge Ratings

The "C" rating on a LiPo battery tells you how quickly the battery can safely discharge. The "C" actually stands for capacity because the number (25C in the header picture) is multiplied by the battery's capacity in amps -- 5000mAh in our example, or 5A, meaning this battery can maintain a whopping 125A of current without being damaged.

Batteries also have a "peak" or "burst" -draw rating which is a number higher than the C-rating that they can sustain for up to 10 seconds. In racing drones you'll sometimes see these peaks under hard acceleration.

It can be difficult to figure out the C rating your application requires. It's usually easisest to just search for what has worked for others with similar needs. There's no harm in having a higher C rating than you need, but significantly underestimating your power requirements will probably damage the battery.

### Charging Rate

Although it isn't directly listed on the battery, the 5000mAh capacity also tells you the safe recharge rate of the LiPo -- 5 amps. Similarly a 3000mAh battery could be safely charged at 3A. Similar to the discharge C rating, this is sometimes referred to as "1C charging" (1A multiplied by the mAh capacity) but since it's the same for all LiPo batteries, it usually isn't printed on the battery.

Some newer batteries advertise higher charging rates, but based on what I've read most people still prefer the conservative 1C charging rate.

## Battery Charger Wattage

LiPo batteries require a charger specifically designed for lithium batteries. In the toolbox picture shown earlier, you can see that I use a [HiTec X4 AC](https://hitecrcd.com/products/chargers/discontinued-chargers-charging-accessories/x4-ac-plus-4-port-acdc-multi-charger/product). Although this charger is no longer sold, it is an example of pretty good entry-level charger. It was about $200 when new. It is rated at 50 watts per port, which limits our maximum rate of charge.

Wattage is determined by multiplying voltage by amperage. Using the rating information for the 4S battery in the article header, we know the nominal voltage is 14.8V and that it can charge at 1C which, based on the 5000mAh capacity is 5A. In watts this is 14.8V x 5A = 74W. This means the charger can't maintain a 1C rate of charge for this battery.

You can turn the equation around to find out how many amps it _can_ handle: 50W / 14.8V = 3.38A. All this means is that your battery will take a little longer to charge. 

In the toolbox photo, you can also see the labeling for one of the smaller 3S 2100mAh 11.1V 20C batteries we used on our smaller, lighter racing drones. These little batteries could push the drones to speeds of up to about 50 MPH with a run-time of about 8 or 9 minutes. Applying the same calculations, we see that the maximum 2.1A charge rate requires only 11.1V x 2.1A = 23.31W, which is well below the charger's 50W-per-port rating.

Not being able to maximize charge amperage is actually pretty common for larger batteries. There are chargers available that have very high output, but they're extremely expensive. There are even chargers that run in the 1000W range which use external power supplies similar to a computer's power supply.

## Parallel Charging

If your charger has excess power available, you can take advantage of a technique called parallel charging to charge more than one battery at a time. 

**Warning:** There are specific rules and procedures you must follow to safely charge lithium batteries in parallel. These will be covered in the next two sections.
{: .notice--warning}

As we now know, a 3S battery is really just three batteries (cells) wired together in series into a single battery pack. You _could_ wire two battery packs together in series -- two of those 3S batteries would then be charged as a 6S 22.2V 4200mAh battery, although series wiring is time-consuming and potentially error-prone, and my 50W charger isn't powerful enough to sustain 1C for a battery pack like that anyway.

Instead, we can wire two 3S batteries in parallel. We still double the mAh rating but the voltage remains the same: 11.1V x 4.2A = 46.62W, still below the charger's 50W rating. Some say you shouldn't exceed 80% of the rating, which would be 40W for my charger, but in reality the charger will automatically adjust the amperage on the fly anyway, and has over-temperature and over-current protection built in.

In the toolbox picture posted earlier, you can see a couple of boards with yellow and white jacks on them. These are parallel charging boards. They're cheap, and you don't need to waste money on the type with fuses or circuit breakers in them as long as you follow all the rules.

You'll notice my parallel boards can handle up to 6 batteries, and what isn't visible is that the parallel boards themselves can be daisy chained. Given the calculations we did in the section on charger wattage there is definitely a point of diminishing returns, but you can safely charge multiple batteries that would optimally require a higher wattage charger, it will just charge them at a slower rate (lower amperage) than the theoretical maximum.

## Parallel Charging Rules

The first three rules are non-negotiable. The fourth rule is, too, unless you're like me and only charge fully-depleted batteries (meaning 3.3V or 3.2V). The fifth rule is more of a rule of thumb -- just be smart about it.

### 1. Always Match Cell-Count (S-Rating)

A 3S battery should only be parallel charged with other 3S batteries, never with a 2S or 4S. Mixing cell counts will lead to overcharging and damaging the smaller battery, and very possibly starting a fire.

### 2. Always Match Battery Chemistry

Although I only use LiPo batteries, a lot of this information also applies to LiFe and Li-Ion batteries. Do not mix battery chemistries while parallel charging.

### 3. Match Battery Capacity (mAh-Rating)

You should only parallel charge batteries of _similar_ capacities. It would be ok to charge a 2000mAh battery with a 2200mAh battery, but not with a 10,000mAh battery.

### 4. Match Measured Voltage (Discharge Level)

All the batteries charged in parallel should be discharged to the same approximate level. By the numbers, the charge state should be within 25% of each other, although this isn't a linear voltage measurement. There are charts available around the 'net, but personally I only parallel charge fully-discharged batteries (3.3V or 3.2V), so I don't have a chart handy (or use it regularly).

If you violate this rule, the lower-charge battery will draw power from the higher-charged battery, probably at relatively high amperage, and probably causing damage to the lower-charge battery (the draw could easily exceed the 1C charge-rate limit).

### 5. Match Battery Age / Condition

As batteries age, their internal resistance goes up, which changes how difficult they are to recharge. You should try to keep batteries of similar age and condition together when parallel charging. The charger is "seeing" multiple batteries as if they're one big battery, so they need to consistently respond as if that is actually the case. "Condition" refers to how much use and abuse they've seen. You don't want to mix a brand new battery with one you bought a year ago and have crashed into trees twenty or thirty times.

It's also a good idea to avoid mixing batteries of different brands or product lines.

## Parallel Charging Procedures

These are all very important. Follow these procedures in the order shown. Be very careful that you plug all connectors in with the correct orientation. Virtually all types of power connectors are keyed and the balance plugs are keyed, but it's still possible to make momentary contact with the wrong orientation which is enough to cause a short.

### 1. Connect Power Plugs First

My batteries use the yellow XT-60 connectors. Starting with the battery with the highest charge (if applicable), connect ONLY the power plug to the parallel board -- do not connect the balance plug yet. Connect the other batteries' power plugs as well. After all the power plugs are connected, _do not_ connect the balance plugs yet.

### 2. Wait Five Minutes

Even if your meter shows the batteries are all at the same voltage, in reality they're not exactly the same. You're giving the batteries time to self-balance their voltages through the heavier gauge power wires. You don't want them doing this through the smaller balance wires. (If you've dug up a chart and are following the 25% limit from rule 4 in the previous section, you should probably wait for as much as 10 minutes. I use five minutes because I only recharge batteries at almost the same discharge level.)

### 3. Connect Balance Plugs

After the voltages have had a chance to equalize, you can connect the balance plugs to the boards. Again, pay attention to the way the plugs are keyed and oriented, you don't want to short a battery through the balance wires. After this, you can connect the parallel charging boards to the battery charger and turn on the power.

### 4. Add Battery mAh Capacities

Every charger has different setup procedures, be sure you understand how your charger works. For mine, I enter the charging amperage and the cell count (S-rating). This is the value used in the calculations to set your charger amperage. For example, parallel charging two of the batteries shown in the header picture of this article is 5000mAH x 2 = 10000 mAh. (If I had a sufficiently powerful charger, I could set it for 3S and 10A, but based on our earlier calculations that would require a charger with at least 148W of power.)

## Charging Safety

You've probably noticed this article is littered with safety warnings. Experienced RC hobbyists take lithium battery safety _very_ seriously. Many have built concrete-block charging "bunkers" and keep a bucket of sand to pour over a battery fire. LiPo fires are a chemical fire that are unaffected by regular fire extinguishers, but a fire extinguisher might protect other things nearby until the battery burns itself out. According to some sources, the actual lithium chemical-reaction fire is brief, a few seconds, most of the fire is plastics and other materials used to make the battery pack.

In any case, always charge on a non-flammable surface and away from flammable materials. Never leave the charger unattended.

The scary videos you'll find online of a LiPo bursting into big flames are staged for dramatic effect by overcharging the batteries to a ridiculous level, but I've read of damaged batteries blowing smoke that was measured at close to 1000F so the heat is very real.

## Battery Storage

You should store the batteries in a secure metal container on a non-flammable surface and away from other flammable objects. A metal toolbox on a metal rack is a good solution.

Lithium batteries have a "storage charge" voltage level that is close to the nominal charge level. Virtually all battery chargers have a feature which drains a battery to the storage charge. In theory this charge level is where the battery is most chemically stable. There are a lot of estimates out there about how quickly a fully-charged battery will discharge, and most sources will tell you that storing a battery with a full charge for any length of time may damage the battery. 

There's probably good reason for the claim and it may even be true, but I haven't found this to be the case. The battery pictured in the header was accidentally stored for _three years_ at full charge, and the 16.0V reading in the photo is what the battery was at when I found it. You should use the storage charge levels but don't feel like it's the end of the world if you don't always remember.

Many people store their batteries in a refrigerator. Heat is the enemy of lithium batteries, although I think the impact of this is also overstated. That same battery in the header picture wasn't just stored fully charged for three years, it was stored in a hot Florida garage for most of those three years (a relative did this, not knowing any better). 

Based on the charge and temperature guidance that battery should be dead as a doornail, but as far as I can tell it's ready to plug in and go to work.

Of course, there's no good reason to intentionally mistreat the batteries, but don't worry too much if you don't have a spare fridge around, or your garage is the only safe place away from flammables to store the batteries. You should, however, always set the batteries to their storage charge level -- it's easy and there are no drawbacks. (If you do have a spare fridge, always let the batteries come to room temperature before using or charging them.)

## Battery Disposal

If you have older batteries that can't hold a full charge, keep in mind you can still use them for less intensive things like lighting where maximum charge isn't as important.

But eventually you'll have to dispose of a battery. If it still has charge, you can fully discharge it by wiring it up to a 12V light. Treat this just like charging -- do it in a safe place, deeply discharging a lithium battery poses a small risk of failure just like charging does. Once the battery is at zero volts, just throw it in the trash, nothing in a lithium battery is toxic. Remember to clip off the power plug and balance plug for later re-use.

If you have a damaged battery, discharging it for disposal is more complex. You should dump it into a large amount of salt water and check it every 24 hours until it no longer holds a charge. I use a 5 gallon bucket from the hardware store, and I just keep adding salt until it no longer dissolves. One of the smaller 3S batteries (the type visible in the toolbox photo) took 48 hours to fully discharge this way. Unfortunately the salt water is also likely to corrode the plugs.

## Conclusion

Although this looks like a lot to consider, lithium technology batteries are actually pretty easy to safely use and maintain, and they have a great power-to-weight ratio -- a must for any RC, robotics, or other electronics hobbyist.
