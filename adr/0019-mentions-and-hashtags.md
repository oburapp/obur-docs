# ADR-0019: Mentions and Hashtags

**Date:** 2026-11-13
**Status:** Accepted; implemented in `obur-backend` Phase 12

## Context

PDD §6, §7, §11, and §12 settle what mentions and hashtags are: mentions
live in a structured `CHECKIN_MENTION` table, only between mutual
followers, notify without granting visibility, and are purged on block;
hashtags are free text, at most five per check-in or list, shared through
one `HASHTAG` table, discoverable on a public-only page, and must not
split one tag into two rows over Turkish dotted/dotless I.

What the PDD left open is how either is written, stored, and kept
correct over time. Usernames can change (§6), so a mention stored as
`@username` text goes stale. A check-in's note and visibility can be
edited after posting (ADR-0011), so a mention valid at creation can stop
being valid. And "Turkish-aware normalisation" collides with English:
lowercasing `I` to `ı` splits `#IPA` from `#ipa` instead. These choices
shape the API contract `obur-web` and `obur-mobile` build against, so
they are settled here.

The overall model follows Instagram, the reference the product already
uses for this interaction: mentions and hashtags are typed inline in the
text, not entered in separate fields.

## Decision

### Both are inline in the text

- A check-in's mentions and hashtags are written inside its `note`; a
  list's hashtags inside its `description`. Lists carry no mentions
  (PDD §7 defines `CHECKIN_MENTION` only).
- The server parses the text on every create and every edit and
  recomputes the link rows from it. The text is the source of truth; the
  link tables are derived from it.
- Drafts (ADR-0018) need no extra fields: the note already carries both.
  Mentions in a draft are validated at submission, not while drafting.

### Mentions

- **Stored as an id token, rendered as the current username.** The note
  stores `@[<user_id>]`. Responses return the note unchanged plus a
  `mentions` array resolving each token to the person's current username,
  so a username change updates every past mention automatically.
- **Who can be mentioned: only people who can see the check-in**, on top
  of the mutual-follow rule:
  - `public`: any mutual follower.
  - `close_friends`: a mutual follower who is on the author's close
    friends list.
  - `private`: nobody.
- **An ineligible mention rejects the request with 422.** The client
  only offers eligible people (below), so this only happens on a client
  bug or a race; the server never trusts the client's filtering.
- **At most 10 mentions per check-in**; more is a 422.
- **A mentionable-users endpoint** returns the author's eligible people
  for a given visibility, filtered by a username prefix, for the `@`
  dropdown.
- **Editing the note:** a newly added person gets a `mention`
  notification; a removed person's mention row is deleted and their
  existing notification is left alone; someone removed and added back is
  notified again.
- **Only newly added tokens are validated on edit.** A token already in
  the previous note is never a reason to reject the edit. If its mention
  was removed earlier (a "dormant" token), the edit re-checks it: an
  eligible person gets the mention back silently, without a second
  notification; an ineligible one stays a placeholder. Without this, a
  dormant token would either block every future edit of the note with a
  422, or make that person permanently unmentionable in it.
- **Unblocking does not restore a mention by itself.** Blocking ended
  the follow in both directions, so after an unblock the two people are
  not mutual followers until they follow each other again. The mention
  comes back on the author's next edit once the person is eligible.
- **A mention that stops being valid is removed, the text is kept.**
  This happens when:
  - the check-in's visibility changes so the person can no longer see it,
  - the person is removed from the author's close friends list while
    mentioned in a `close_friends` check-in (which also covers unfollow,
    since unfollowing revokes close-friend status by cascade),
  - the two people block each other (PDD §11's existing purge).
  A frozen or suspended account's mentions are returned the same way
  while the account is hidden.
  The token stays in the note and is returned with no username; the
  client shows a generic, localised placeholder in its place ("Obur
  kullanıcısı" / "Obur user"), the same way Instagram shows "Instagram
  user". The person's name is not shown to viewers after they lose
  access.
- **Unfollowing alone does not remove a mention** on a `public`
  check-in: the person can still see it, and the mention recorded a real
  relationship at the time.

### Hashtags

- **Character set (Instagram's):** a tag is `#` followed by Unicode
  letters, combining marks, digits, and underscore. Any other character,
  including whitespace, punctuation, and emoji, ends the tag. A tag of
  only digits is not a tag.
- **At most 50 characters per tag** and **at most 5 distinct tags per
  check-in or list**; more is a 422. Repeating the same tag in one text
  counts once.
- **Matching is case-insensitive; display is as typed.** The match key
  is Unicode NFC normalisation, then Unicode default case folding, with
  one added rule applied first: `İ` becomes `i`. `HASHTAG.tag` stores
  that key (unique), and the hashtag page's heading shows it. Inside a
  check-in or list, a tag shows exactly as its author typed it, because
  it is part of their text and is never rewritten.
- **A tag no longer used anywhere is kept**, a harmless catalog row.

### Hashtag discovery

- Two endpoints, one per content type: a tag's check-ins and a tag's
  lists, each most recent first and paginated. The tag in the path is
  normalised the same way, so `/hashtags/Kahve` and `/hashtags/kahve`
  are the same page. Whether a client shows
  them as tabs or one merged feed is a UI decision the API doesn't make.
- Only `public`, non-deleted content from visible accounts.
- **Blocked users' content is hidden in both directions; muted users'
  content is shown.** Muting affects only the muter's own feed
  (PDD §11), and a hashtag page is discovery, not feed.

### Row Level Security

- `hashtags`: readable by everyone; insertable by any authenticated
  user; never updated or deleted by the application.
- `checkin_hashtags`, `list_hashtags`, `checkin_mentions`: readable when
  the parent check-in or list is; written only by its owner. The block
  purge function (`rls_purge_interactions_between`) is extended to cover
  mentions.
- **Close-friend removal is enforced by a database trigger.** A
  `close_friends` row disappears three ways: the owner removes the
  person, the person unfollows, or the owner removes them as a follower.
  The last two happen through a foreign-key cascade the application
  never sees. An `AFTER DELETE` trigger on `close_friends` removes that
  person's mentions from the owner's `close_friends` check-ins however
  the row was deleted. This is a third, named exception to the
  backend's no-raw-SQL rule, next to PostGIS queries and RLS policy DDL,
  for the same reason as RLS itself: a guarantee no code path can forget.

## Rationale

**Id token over username text.** Usernames are changeable by design.
Text that names a person by handle silently breaks on rename; an id is
the only stable reference, and resolving it at read time is what keeps
every past mention current.

**Inline over separate fields.** It is what users already know from
Instagram, it keeps mentions and hashtags consistent with each other, and
it means drafts need nothing new: the note carries everything.

**Visibility-gated mentions.** The PDD rule "a mention notifies but never
grants visibility" left a notification that opens to a 404 whenever the
check-in was `private` or `close_friends`. Allowing mentions only of
people who can see the check-in removes that state entirely instead of
explaining it to users.

**Generic placeholder after removal.** Keeping the text preserves what the
author wrote; replacing the name with a placeholder keeps a person who was
blocked or lost access from staying visibly attached to content they can
no longer see.

**Case folding with one Turkish rule.** Instagram matches tags
case-insensitively but does not publish its exact folding, so the closest
standard is used: NFC plus Unicode default case folding. A tag page
belongs to everyone who used the tag, so its heading shows the shared
normalised form rather than any one author's spelling. That standard
handles English and most Turkish correctly, but folds `İ` to `i` plus a
combining dot, which would make `#İstanbul` and `#istanbul` different
tags, a highly visible failure for a Turkey-first product. One explicit
rule fixes it without changing anything else.

### Alternatives considered

- **Separate `mentions` / `hashtags` fields beside the note.** Rejected:
  two sources that can disagree, and drafts would need new fields.
- **Server resolves `@username` at write time and stores the text.**
  Rejected: stale on the first rename.
- **No case normalisation.** Rejected: `#Kahve` and `#kahve` would be two
  pages, and neither would show the other's posts.
- **Turkish-locale lowercasing.** Rejected: it breaks English tags the
  other way (`#IPA` → `ıpa`).
- **Silently dropping ineligible mentions.** Rejected: the author would
  believe they tagged someone who was never told.
- **Notifying only when the mentioned person can see the check-in.**
  Superseded by gating the mention itself on visibility.

## Consequences

- Every create and edit of a check-in note or list description parses
  text and rewrites link rows; a visibility change or close-friend
  removal triggers a mention cleanup.
- The note's stored form (`@[<user_id>]`) is part of the API contract.
  Clients send and receive tokens and render them using the `mentions`
  array.
- Turkish uppercase words containing `I` meant as dotless (`#IŞIK` vs
  `#ışık`) can still match as different tags. The correct folding depends
  on the language of the word, which isn't known; accepted as rare.
- A removed mention's notification stays and may lead to a check-in the
  person can no longer open. Accepted by choice as a minor edge case.
- PDD §7, §11, and §12 are updated to match.
