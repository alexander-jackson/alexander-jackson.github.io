---
title: "Blackboards: Warwick Barbell Management"
date: 2021-03-13
draft: false
tags:
- rust
- postgresql
- oauth1
---

Blackboards is a website written for Warwick Barbell during my time as the
powerlifting captain. It handles registration for club sessions, displaying
member personal bests and running the annual position elections.

<!--more-->

Source: https://github.com/alexander-jackson/blackboards/

## Overview

Each year, Warwick Barbell runs taster sessions that allow newer lifters to try
out powerlifting and weightlifting while being coached by existing members. Due
to the ongoing pandemic, we needed to ensure that attendance at these sessions
was both recorded for contact tracing and capped for social distancing. While
this could be achieved through approaches such as Google Forms or other
services, I took the opportunity to work on it as a personal project to learn
about websites and deployment. This also allowed more strict constraints on
sign-ups, such as preventing multiple session bookings and non-Warwick
students.

The name `blackboards` was chosen as this was initially a project to display
members' personal bests on a website, similar to how powerlifting gyms may have
a physical blackboard that people can write their lifts on. The domain name
`blackboards.pl` was also free, with `.pl` somewhat standing for
'powerlifting'.

### Initial Usage

For the first round of taster sessions, users could click on a session and
enter their `warwick.ac.uk` email alongside their name. The system would then
automatically send them an email with a confirmation link which they could
click to fully register for the session. This ensured that only Warwick
students could sign-up but required them to enter the same information every
time they booked a session in later versions.

### Deployment

Deployment was handled through Digital Ocean, as I had prior experience and
enjoyed the simplistic usage. Being written in Rust, the compiled binary was
also capable of handling several hundred requests per second even on the
cheapest offering.

### OAuth1

Warwick provides an API for users to sign in with their Warwick account to
other websites using OAuth1. Members of the club had previously complained that
they needed to enter their information every time they registered for a
session, so I began to explore other options. Allowing them to sign-in through
Warwick's single sign-on was easier for users, more secure and provided better
consistency as I could work with their Warwick identifiers instead of names.

After applying for a token to use on the domain `blackboards.pl`, I began
experimenting with the system. As most major implementations now use OAuth2 and
Warwick's documentation was written at a higher level than Rust's
`oauth1-request` provided, this required a lot of trial and error. However,
once it began working, users could automatically be authenticated when visiting
the page and redirected back to the site where I could work with their user
identifier instead. This meant that sessions could be booked through a simple
button click and all pages could be aware of who was currently viewing them.

### Continuous Deployment

`blackboards` itself was the main driver for another of my projects called
`fisherman`. Every time changes were made to `blackboards` itself, someone
would need to SSH into the server, pull the changes and wait for them to build
before restarting the program with `supervisorctl restart blackboards`.

Eventually this became cumbersome, so `fisherman` was written to handle this
process for me. It would listen for GitHub's webhook messages and decide
whether to pull or not before building a new binary and deploying it. This
allowed for much faster development of features with them being automatically
deployed to the production instance.

### Elections

Towards the end of second term, each society and club typically hold elections
where members can vote on who they want in each executive position for the
coming year. When held in person, this is done by paper balloting similar to
general elections, however the ongoing pandemic made this not possible.
Speeches were instead done through Microsoft Teams and we needed a way to
ensure the election ran smoothly. Additionally, we wanted to hold Warwick
Barbell's first ranked choice voting election, something other clubs had found
success with. For these reasons, I began working on extending Blackboards to
support running an election.
