Read [](file:///home/apic/codeSPACE/stuFlux/apps/api/src/modules/listings/repository.ts)

Remaining gaps for the listings module to be “complete” vs the spec:

- **DB migrations**: need a migration for `listing_photos` matching the new schema (table + indexes). Current schema.ts is updated, but migration isn’t generated/applied.
- **Ownership-enforced updates**: repository uses `and(eq(listings.id, id), eq(listings.owner_id, ownerId))`—good—but service currently reloads via `getListing(updatedId)` which increments view counts again; could avoid double-increment by not calling detail immediately or skipping increment when requester is owner.
- **Owner avatar**: placeholder `avatar: sql`NULL``; when user avatars are stored, update the select.
- **Photos rules**: service caps 5 photos; repo doesn’t enforce a max beyond what’s passed. If photos are omitted on update, existing photos remain—currently update deletes and replaces only when `photos` provided; no soft-delete flag used.
- **Publish guard**: enforced in service, but no DB constraint for status vs required fields/photos (acceptable, but note it’s app-level only).
- **Filters**: free-text only on title; spec wanted title/description—currently only title is ilike. Add description ilike for better search.
- **Sorting**: “popular” uses view_count desc; if you want booking_count as tiebreaker, add it.
- **Pagination/meta in controller**: browse now returns `{data, meta}`, but detail/create/update still wrap `{ data }`. That’s fine, but ensure frontend expects this shape.
- **Tests**: no integration/unit tests yet for create/publish/list/detail, per-user limit, photo rules, status transitions, view count increment.
- **Category FK validation**: relies on DB FK; no app-level check for existence beyond FK error handling.
- **Specs size**: enforced in service; repo doesn’t recheck—okay, but be aware.
- **Max listings per user**: enforced on create; not rechecked on publish (current code does not recheck when updating status to active; if you want to enforce count at publish too, add check).

If you want, I can generate the `listing_photos` migration, add description search, and add tests for the critical paths.