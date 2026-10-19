---
title: "Building the TrafficG Frontend with React and TypeScript"
description: "A development diary covering the screens I built, styling problems I encountered, and planned improvements."
publishDate: 2025-05-25
updatedDate: 2026-10-19
tags: [trafficg, react, typescript, frontend]
draft: false
---

> English translation of the [original Korean frontend diary](https://tyge-ottesen-brahe.tistory.com/39). Features described as future work in the original are still labeled as future work here. Original screenshots have not yet been imported.

After roughly two weeks of work, I wanted to document the screens I had built. I was surprised by how difficult it was to make an interface look the way I imagined. The result was rougher than I had hoped, but I planned to refine it gradually.

To make the ticketing project more enjoyable, I chose a playful *Bikini Bottom* theme inspired by SpongeBob rather than a generic concert reservation interface. This was a personal prototype theme, not a commercial affiliation.

**Stack:** React, TypeScript, CSS Modules, and `json-server`.

## Login

I implemented the overall visual direction and login screen. There was no major technical blocker here, but I spent about three hours adjusting the appearance because I wanted it to feel right.

*Screenshot placeholder: original login screen.*

## Registration

While building registration, I extracted shared input and button components. Repeating font declarations across those components became inconvenient, so I moved the font setup into a global style.

I then discovered that the global font did not behave as expected in buttons. The component-specific button styles were overriding the general styling, so I needed to review CSS inheritance and the cascade.

I also saw an unusual password field issue: characters appeared as blank or invisible marks. The font I had selected lacked a suitable glyph for the masking character in that context. I adjusted the font fallback so the password field would render correctly.

*Screenshot placeholder: original registration screen.*

## Performance list

I built a list of performances and experimented with showing personalized text after login, such as welcoming the signed-in user. I was initially unsure how best to store and retrieve images, so I used a local server to serve image files during development.

I still wanted to add proper search, category filtering, date sorting, pagination, and more consistent card dimensions.

*Screenshot placeholder: original performance list.*

## Performance details

I implemented a page showing details for a selected performance. Two issues remained: line breaks in a stored description were not displayed as expected, and the planned waiting-number display did not yet decrease in increments of 100 per second.

Future work included date-specific booking buttons, proper multiline rendering, a calendar, and venue information.

*Screenshot placeholder: original performance detail.*

## Seat selection

The seat-selection UI allowed choosing up to four seats. I ran into another styling issue with button inheritance and nested components. I tried resetting button styles using `all: unset`, which helped in the immediate case.

The interface still needed visual improvements, and handling the case where another user had already taken a seat remained planned work.

**Technical review:** `all: unset` also removes useful browser defaults, so accessible focus indicators, keyboard behavior, and button semantics need explicit testing after applying it.

*Screenshot placeholder: original seat-selection screen.*

## Payment and confirmation

I built a simulated payment page and a success screen. Passing route parameters and working with radio-button inputs proved more difficult than I expected. I also considered adding a history of completed reservations after confirmation.

*Screenshot placeholders: original simulated payment and confirmation pages.*

## Remaining work at the time of writing

- Administrative functionality.
- A user profile and reservation history.
- Fixing a blank white area when scrolling.
- Preserving login state.
- Preventing unauthenticated users from entering booking or payment flows.

This diary captures a point in the project's development. It is not a statement that every item on the list was later completed. I want the post to preserve the small implementation problems and decisions, because those details show what I learned while actually building the interface.