# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository manages event materials and slides for two developer communities targeting new graduates and young developers:

1. **24th Dev** - A networking community for 24th-year graduate engineers and their peers (25th~23rd graduates)
2. **wakate.rb** - A Ruby community for young Rubyists (students and recent graduates) aimed at connecting young Ruby developers

## Repository Structure

- `events/` - Event planning materials and templates
  - `24th-dev/` - 24th Dev community event materials
  - `wakate-rb/` - wakate.rb community event materials
- `slides/` - Presentation slides using Marp
  - `24th-dev/` - 24th Dev presentations (opening and LT slides)
  - `wakate-rb/` - wakate.rb presentations (opening and LT slides)  
- `images/` - Shared image assets for presentations

## Template System

Both communities use template-based event management:

### Event Templates
- Event description templates in `events/[community]/template.md`
- These contain TODO placeholders that need to be replaced when creating new events:
  - `sponsor` → sponsor company name
  - `theme` → event theme
  - `number` → event number
  - `reception_start_at` → reception start time (default: 18:30)
  - `reception_end_at` → reception end time (default: 19:15)
  - `receptionist` → receptionist Twitter account ID

### Slide Templates
- Marp-based presentation templates in `slides/[community]/opening/template.md`
- Use consistent styling with green color scheme (#008800)
- Standard slide structure includes: title, self-introduction, content overview, timetable

## Creating New Events

1. Copy the appropriate template from `events/[community]/template.md`
2. Replace all TODO placeholders with actual event information
3. Create corresponding slides by copying from `slides/[community]/opening/template.md`
4. Update slide content with event-specific information
5. Use appropriate images from the `images/` directory

## Marp Configuration

All slides use Marp with consistent styling:
- H1: #008800, 64px
- H2: #008800, 48px  
- H3: 40px
- P/LI: 32px

Images are typically positioned as `![bg right:40% 80%]` for consistent layout.

## Event Management Best Practices

- Each event should have both an event description (in `events/`) and opening slides (in `slides/`)
- LT (Lightning Talk) slides are stored separately in `lt/` subdirectories
- All templates contain extensive comments with examples and guidelines
- Event descriptions include comprehensive information about venue, schedule, participation guidelines, and anti-harassment policies