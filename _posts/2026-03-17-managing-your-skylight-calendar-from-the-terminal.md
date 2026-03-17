---
layout: post
title: "Managing Your Skylight Calendar from the Terminal"
date: 2026-03-17
tags: [go, cli, homelab, tooling]
excerpt: "go-skylight brings your family's Skylight Calendar to the command line — manage events, chores, rewards, meals, and more without touching the app."
---

## Introduction

The [Skylight Calendar](https://www.ourskylight.com/) is a touchscreen frame that lives in your kitchen and keeps the family organized — events, chores, reward points, grocery lists, meal plans. It's great. But if you want to automate anything — reset chore statuses on a schedule, pull today's events into a script, bulk-add grocery items — you're stuck tapping the app.

[go-skylight](https://github.com/sebrandon1/go-skylight) fixes that. It's a CLI (and underlying Go library) that gives you full access to your Skylight frame from the terminal.

## Installation

```bash
go install github.com/sebrandon1/go-skylight@latest
```

Or grab a pre-built binary from the [releases page](https://github.com/sebrandon1/go-skylight/releases) — Linux and macOS, amd64 and arm64.

## Authentication

The easiest way to get started is to log in and save your credentials:

```bash
go-skylight login --email you@example.com --password yourpassword --save
```

This writes your token and user ID to `~/.skylight/config` so you don't have to pass credentials on every command. You'll also want to set your frame ID — find it with:

```bash
go-skylight get frame info --email you@example.com --password yourpassword
```

Add `SKYLIGHT_FRAME_ID=yourframeid` to `~/.skylight/config` and you're set. Every subsequent command picks it up automatically.

## Daily Dashboard

The quickest way to see what's going on:

```bash
go-skylight today
```

This pulls today's calendar events, outstanding chores, reward point totals, meal plan, and active lists in one shot — useful as a morning summary or a cron job that posts to Slack.

## Managing the Calendar

```bash
# List today's events
go-skylight get calendar list

# List events in a date range
go-skylight get calendar list --start 2026-03-17 --end 2026-03-21

# Add an event
go-skylight get calendar create --title "Soccer practice" --start-at "2026-03-18T17:00:00" --end-at "2026-03-18T18:30:00"

# Delete an event
go-skylight get calendar delete --event-id abc123
```

## Chores

```bash
# See all chores
go-skylight get chore list

# Add a chore
go-skylight get chore create --title "Unload dishwasher" --points 5 --assignee-id kidid123

# Mark it done (status: completed)
go-skylight get chore update --chore-id xyz789 --status completed

# Delete it
go-skylight get chore delete --chore-id xyz789
```

Update commands only send the fields you explicitly pass — so `--status completed` won't touch the title or points.

## Rewards and Points

```bash
# List rewards
go-skylight get reward list

# Check a family member's points
go-skylight get reward points --user-id kidid123

# Redeem a reward
go-skylight get reward redeem --reward-id rwdabc --user-id kidid123
```

## Grocery List and Meals

```bash
# View meal categories
go-skylight get meal categories

# Add an item to the grocery list
go-skylight get meal add-to-grocery --name "Oat milk" --quantity 2

# List this week's meal sittings
go-skylight get meal sittings
```

## Lists

```bash
# See all lists
go-skylight get list all

# Add an item to a list
go-skylight get list add-item --list-id listid123 --title "Call the dentist"

# Mark it complete
go-skylight get list update-item --list-id listid123 --item-id itemid456 --completed true
```

## Scripting and Automation

Since every command outputs JSON, go-skylight pairs well with `jq`. A few examples:

```bash
# Get just the titles of today's events
go-skylight get calendar list | jq '.[].title'

# Count incomplete chores
go-skylight get chore list | jq '[.[] | select(.status != "completed")] | length'

# Post the daily dashboard to a webhook
go-skylight today | curl -s -X POST -H "Content-Type: application/json" \
  -d @- https://your-webhook-url
```

Or set up a cron job to reset chore statuses at the start of each week, add recurring grocery items, or sync an external calendar to your Skylight frame.

## Conclusion

If your family uses Skylight and you're comfortable with a terminal, go-skylight opens up a lot of possibilities — automation, scripting, bulk edits, and integrations that would otherwise require tapping through the app. The full command reference is in the [repo README](https://github.com/sebrandon1/go-skylight).
