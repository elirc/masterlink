# User Stories

## Story 1: Rename Empty Recent Links Prompt
**Difficulty:** Easy  
**Estimated Time:** 1 hour  
**Skills You'll Practice:** reading JSX, i18n keys, visual verification  
**The Story:** As a new user, I want the empty dashboard recent-links prompt to be clearer so that I know adding my first link will populate the dashboard.

**Acceptance Criteria:**
- [ ] The empty recent-links section still appears only when dashboard data is not loading and there are no recent links.
- [ ] The primary and secondary text use clearer copy without changing layout.
- [ ] The "Add link" and import actions still appear and work.

**Files You'll Likely Touch:**  
`apps/web/pages/dashboard.tsx` - empty recent-links state and add-link modal trigger live here [apps/web/pages/dashboard.tsx:287-317](../../apps/web/pages/dashboard.tsx#L287).  
`apps/web/public/locales/en/common.json` - translation strings are used through `t(...)` in dashboard UI [apps/web/pages/dashboard.tsx:296-311](../../apps/web/pages/dashboard.tsx#L296).

**High-Level Implementation Plan:**
1. Inspect `apps/web/pages/dashboard.tsx` around the `RECENT_LINKS` case to find `view_added_links_here` and `view_added_links_here_desc`.
2. Update the English locale keys used by those `t(...)` calls.
3. Run the app and verify the empty state after creating a fresh account or clearing dashboard data.

**Tips:**
- The dashboard section switch is driven by `DashboardSectionType` [apps/web/pages/dashboard.tsx:242-428](../../apps/web/pages/dashboard.tsx#L242).
- Do not change `setNewLinkModal(true)`; it opens the existing link modal [apps/web/pages/dashboard.tsx:303-312](../../apps/web/pages/dashboard.tsx#L303).
- Keep translation key names stable unless you update every locale.

**What Could Go Wrong:**
- Changing JSX instead of locale text may leave other languages inconsistent.
- Removing the `Button` trigger can break the first-link path.

**Stretch Goal:** Add a more precise empty prompt for pinned links too [apps/web/pages/dashboard.tsx:341-356](../../apps/web/pages/dashboard.tsx#L341).

**Connects To:** Story 2, because both teach small safe UI changes.

## Story 2: Show Dashboard Metric Tooltips
**Difficulty:** Easy  
**Estimated Time:** 1.5 hours  
**Skills You'll Practice:** component props, reusable UI, dashboard rendering  
**The Story:** As a user scanning my dashboard, I want metric icons to explain what they count so that the stats section is easier to understand.

**Acceptance Criteria:**
- [ ] Each dashboard stat icon has an accessible title or tooltip.
- [ ] The visual layout of `DashboardItem` remains stable.
- [ ] Existing metric names and values still render.

**Files You'll Likely Touch:**  
`apps/web/components/DashboardItem.tsx` - renders metric icon/name/value [apps/web/components/DashboardItem.tsx:1-23](../../apps/web/components/DashboardItem.tsx#L1).  
`apps/web/pages/dashboard.tsx` - passes stat names/icons to `DashboardItem` [apps/web/pages/dashboard.tsx:242-270](../../apps/web/pages/dashboard.tsx#L242).

**High-Level Implementation Plan:**
1. Add an optional `title` or `aria-label` prop to `DashboardItem`.
2. Pass the translated metric name as the label/title from each `DashboardItem` usage.
3. Verify stats render in the `STATS` dashboard section.

**Tips:**
- `DashboardItem` has only three current props, which makes this a good prop-contract exercise [apps/web/components/DashboardItem.tsx:1-9](../../apps/web/components/DashboardItem.tsx#L1).
- The icon is a Bootstrap class string from the caller [apps/web/pages/dashboard.tsx:246-267](../../apps/web/pages/dashboard.tsx#L246).
- Keep the component presentational; do not fetch data inside it.

**What Could Go Wrong:**
- Adding required props can break all callers if you miss one.
- Tooltip text may duplicate visible text; keep accessibility useful but not noisy.

**Stretch Goal:** Use the existing tooltip UI primitives if they fit this use case [apps/web/components/ui/tooltip.tsx:6-35](../../apps/web/components/ui/tooltip.tsx#L6).

**Connects To:** Story 3, because it prepares you to modify modal form labels.

## Story 3: Add Helper Text To New Link URL Field
**Difficulty:** Easy  
**Estimated Time:** 1.5 hours  
**Skills You'll Practice:** form UI, validation awareness, modal layout  
**The Story:** As a user adding a link, I want brief helper text under the URL field so that I know the link must be a valid URL.

**Acceptance Criteria:**
- [ ] Helper text appears under the URL input in `NewLinkModal`.
- [ ] The URL field still focuses when the modal opens.
- [ ] Existing validation still uses `PostLinkSchema`.

**Files You'll Likely Touch:**  
`apps/web/components/ModalContent/NewLinkModal.tsx` - owns the create-link form and submit validation [apps/web/components/ModalContent/NewLinkModal.tsx:84-118](../../apps/web/components/ModalContent/NewLinkModal.tsx#L84).  
`apps/web/public/locales/en/common.json` - add or reuse helper text key.

**High-Level Implementation Plan:**
1. Find the URL `TextInput` in `NewLinkModal`.
2. Add a small translated helper line under it.
3. Verify `useLayoutEffect` still focuses the input and submit validation remains unchanged.

**Tips:**
- `inputRef.current?.focus()` is in `useLayoutEffect` [apps/web/components/ModalContent/NewLinkModal.tsx:84-86](../../apps/web/components/ModalContent/NewLinkModal.tsx#L84).
- Client validation uses `PostLinkSchema.safeParse(link)` [apps/web/components/ModalContent/NewLinkModal.tsx:88-96](../../apps/web/components/ModalContent/NewLinkModal.tsx#L88).
- The backend validates the same schema again [apps/web/lib/api/controllers/links/postLink.ts:16-25](../../apps/web/lib/api/controllers/links/postLink.ts#L16).

**What Could Go Wrong:**
- Placing helper text inside the wrong grid cell can disrupt collection selection layout.
- Copy that promises more than validation does can confuse users.

**Stretch Goal:** Add helper text for optional tags in the expanded options area [apps/web/components/ModalContent/NewLinkModal.tsx:132-163](../../apps/web/components/ModalContent/NewLinkModal.tsx#L132).

**Connects To:** Story 4, because it teaches the modal before adding UI that reads existing data.

## Story 4: Show Selected Link Count In Edit Mode
**Difficulty:** Medium  
**Estimated Time:** 2 hours  
**Skills You'll Practice:** Zustand state, conditional UI, list interactions  
**The Story:** As a user bulk-editing links, I want to see how many links are selected so that I do not accidentally act on the wrong set.

**Acceptance Criteria:**
- [ ] When edit mode is active, the UI displays the selected link count.
- [ ] Count updates when selecting/deselecting links.
- [ ] Count resets when edit mode turns off.

**Files You'll Likely Touch:**  
`apps/web/store/links.ts` - selected IDs and `selectionCount` live here [apps/web/store/links.ts:3-42](../../apps/web/store/links.ts#L3).  
`apps/web/components/LinkViews/Links.tsx` - clears selection when edit mode ends and passes selection helpers to views [apps/web/components/LinkViews/Links.tsx:397-403](../../apps/web/components/LinkViews/Links.tsx#L397).  
`apps/web/pages/links/index.tsx` - owns the all-links page `editMode` state and passes it into `LinkListOptions` and `Links` [apps/web/pages/links/index.tsx:30-65](../../apps/web/pages/links/index.tsx#L30).

**High-Level Implementation Plan:**
1. Locate where `editMode` is toggled for the link listing page.
2. Read `selectionCount` from `useLinkStore`.
3. Render a compact count only while `editMode` is true.
4. Verify the existing `clearSelected` effect resets count.

**Tips:**
- `selectionCount` is already maintained by the store [apps/web/store/links.ts:17-39](../../apps/web/store/links.ts#L17).
- `Links` already reads `clearSelected`, `isSelected`, and `toggleSelected` [apps/web/components/LinkViews/Links.tsx:397-403](../../apps/web/components/LinkViews/Links.tsx#L397).
- Keep server state out of this; selection is local UI state.

**What Could Go Wrong:**
- Reading only `selectedIds` and recalculating count can drift from store logic.
- Rendering count outside edit mode may confuse normal browsing.

**Stretch Goal:** Disable bulk action buttons when `selectionCount` is zero.

**Connects To:** Story 5, because both use existing data/state without adding backend.

## Story 5: Add A Dashboard Section Summary Row
**Difficulty:** Medium  
**Estimated Time:** 2.5 hours  
**Skills You'll Practice:** derived data, dashboard sections, component composition  
**The Story:** As a dashboard user, I want a small summary row above custom sections so that I can quickly see which sections are enabled.

**Acceptance Criteria:**
- [ ] Summary lists enabled dashboard section types in display order.
- [ ] Summary updates when `dashboardSections` changes.
- [ ] Existing dashboard sections still render unchanged.

**Files You'll Likely Touch:**  
`apps/web/pages/dashboard.tsx` - owns `dashboardSections`, `orderedSections`, and section rendering [apps/web/pages/dashboard.tsx:49-93](../../apps/web/pages/dashboard.tsx#L49), [apps/web/pages/dashboard.tsx:150-172](../../apps/web/pages/dashboard.tsx#L150).  
`packages/prisma/schema.prisma` - section enum values are defined here [packages/prisma/schema.prisma:289-294](../../packages/prisma/schema.prisma#L289).

**High-Level Implementation Plan:**
1. Use existing `orderedSections` memoized value.
2. Render a compact row before the section map.
3. Convert `DashboardSectionType` values into translated labels.
4. Verify empty/loading behavior remains intact.

**Tips:**
- `orderedSections` already sorts by `order` [apps/web/pages/dashboard.tsx:87-93](../../apps/web/pages/dashboard.tsx#L87).
- Section types are `STATS`, `RECENT_LINKS`, `PINNED_LINKS`, and `COLLECTION` [packages/prisma/schema.prisma:289-294](../../packages/prisma/schema.prisma#L289).
- Avoid changing the `Section` switch for this summary.

**What Could Go Wrong:**
- Rendering before data loads can show an empty summary too early.
- Collection sections need collection names from `collections`.

**Stretch Goal:** Make collection sections show their collection icon/color in the summary.

**Connects To:** Story 6, because dashboard layout work leads naturally to an API-backed preference.

## Story 6: Add API Endpoint To Return Link Creation Defaults
**Difficulty:** Medium  
**Estimated Time:** 4 hours  
**Skills You'll Practice:** API route creation, auth, shared hook, defaults  
**The Story:** As a user opening the create-link modal, I want defaults loaded from the server so that future preference-based defaults can be added safely.

**Acceptance Criteria:**
- [ ] New authenticated endpoint returns default collection name and link type.
- [ ] New router hook fetches the endpoint only when authenticated.
- [ ] `NewLinkModal` can use the endpoint without breaking existing collection-page preselection.
- [ ] Endpoint returns 401 through existing auth behavior when unauthenticated.

**Files You'll Likely Touch:**  
`apps/web/pages/api/v1/links/defaults.ts` - new API route following existing route patterns.  
`apps/web/lib/api/verifyUser.ts` - use existing auth helper [apps/web/lib/api/verifyUser.ts:14-72](../../apps/web/lib/api/verifyUser.ts#L14).  
`packages/router/links.tsx` - add hook near link hooks [packages/router/links.tsx:826-875](../../packages/router/links.tsx#L826).  
`apps/web/components/ModalContent/NewLinkModal.tsx` - consume defaults carefully [apps/web/components/ModalContent/NewLinkModal.tsx:63-82](../../apps/web/components/ModalContent/NewLinkModal.tsx#L63).

**High-Level Implementation Plan:**
1. Add a `GET` API route under `apps/web/pages/api/v1/links/defaults.ts`.
2. Call `verifyUser` before returning defaults.
3. Add `useLinkDefaults` in `packages/router/links.tsx`.
4. In `NewLinkModal`, prefer collection-page preselection over generic defaults.
5. Add a focused test or manual verification for unauthenticated access.

**Tips:**
- Follow `useDashboardData` for auth-enabled query shape [packages/router/dashboardData.tsx:6-35](../../packages/router/dashboardData.tsx#L6).
- Do not replace the existing collection route preselection effect blindly [apps/web/components/ModalContent/NewLinkModal.tsx:63-82](../../apps/web/components/ModalContent/NewLinkModal.tsx#L63).
- Response shape should be boring and consistent.

**What Could Go Wrong:**
- Defaults fetch can overwrite a collection chosen from the current route.
- Hook may run before session is authenticated if `enabled` is missing.

**Stretch Goal:** Include the user's preferred collection if such a preference is later added.

**Connects To:** Story 7, because it introduces a new server-state hook.

## Story 7: Persist Last Used Link View Per User Locally
**Difficulty:** Medium  
**Estimated Time:** 4 hours  
**Skills You'll Practice:** Zustand settings, localStorage, view mode, cache invalidation awareness  
**The Story:** As a user switching between card/list/masonry, I want Linkwarden to remember my last chosen view locally so that browsing feels consistent.

**Acceptance Criteria:**
- [ ] Changing view mode updates local settings through the existing store.
- [ ] Dashboard and link listing initialize from the same local setting.
- [ ] The setting survives refresh.
- [ ] No server API or Prisma change is introduced.

**Files You'll Likely Touch:**  
`apps/web/store/localSettings.ts` - local settings and `localStorage` persistence [apps/web/store/localSettings.ts:28-123](../../apps/web/store/localSettings.ts#L28).  
`apps/web/pages/dashboard.tsx` - currently reads `localStorage.getItem("viewMode")` directly [apps/web/pages/dashboard.tsx:57-59](../../apps/web/pages/dashboard.tsx#L57).  
`apps/web/pages/links/index.tsx` - initializes link-list view mode from `localStorage` and passes `setViewMode` down [apps/web/pages/links/index.tsx:17-19](../../apps/web/pages/links/index.tsx#L17), [apps/web/pages/links/index.tsx:38-45](../../apps/web/pages/links/index.tsx#L38).  
`apps/web/components/ViewDropdown.tsx` - controls view-mode changes and already calls `updateSettings({ viewMode })` [apps/web/components/ViewDropdown.tsx:20-35](../../apps/web/components/ViewDropdown.tsx#L20).

**High-Level Implementation Plan:**
1. Inspect `ViewDropdown` to see how it sets view mode.
2. Route view changes through `useLocalSettingsStore.updateSettings`.
3. Replace direct `localStorage` reads where appropriate with store settings.
4. Verify dashboard and link pages behave consistently after refresh.

**Tips:**
- `updateSettings` already persists `viewMode` [apps/web/store/localSettings.ts:46-54](../../apps/web/store/localSettings.ts#L46).
- `setSettings` initializes default view mode from `localStorage` [apps/web/store/localSettings.ts:84-90](../../apps/web/store/localSettings.ts#L84).
- `ViewMode` enum values are shared [packages/types/global.ts:81-85](../../packages/types/global.ts#L81).

**What Could Go Wrong:**
- Reading `localStorage` during render can create hydration fragility.
- Multiple components can disagree if some use store and some read localStorage directly.

**Stretch Goal:** Add a small migration fallback if `localStorage.viewMode` contains an invalid value.

**Connects To:** Story 8, because local preferences prepare you for persisted user preferences.

## Story 8: Add Optional Link Note Field
**Difficulty:** Hard  
**Estimated Time:** 8 hours  
**Skills You'll Practice:** Prisma migration, Zod schema, controller update, UI form, API hook  
**The Story:** As a user saving research links, I want a private note field on a link so that I can capture why I saved it.

**Acceptance Criteria:**
- [ ] Database stores an optional note per link.
- [ ] Create-link modal can submit a note.
- [ ] API validates note length and persists it.
- [ ] Link detail/card UI can display the note when present.
- [ ] Existing link creation without a note still works.

**Files You'll Likely Touch:**  
`packages/prisma/schema.prisma` - add field to `Link` [packages/prisma/schema.prisma:166-198](../../packages/prisma/schema.prisma#L166).  
`packages/lib/schemaValidation.ts` - add note to create/update schemas [packages/lib/schemaValidation.ts:125-179](../../packages/lib/schemaValidation.ts#L125).  
`apps/web/lib/api/controllers/links/postLink.ts` - persist note in `prisma.link.create` [apps/web/lib/api/controllers/links/postLink.ts:104-150](../../apps/web/lib/api/controllers/links/postLink.ts#L104).  
`apps/web/components/ModalContent/NewLinkModal.tsx` - add form field in expanded options [apps/web/components/ModalContent/NewLinkModal.tsx:132-163](../../apps/web/components/ModalContent/NewLinkModal.tsx#L132).  
`apps/web/components/LinkDetails.tsx` or link card components - display note.

**High-Level Implementation Plan:**
1. Add nullable `note` field to Prisma `Link`.
2. Generate migration with `yarn prisma:dev`.
3. Add `note` to `PostLinkSchema` and update schema if editing supports it.
4. Add `note` to modal state and expanded form UI.
5. Persist `note` in `postLink`.
6. Display note in the link detail/card area.
7. Verify create, read, and no-note paths.

**Tips:**
- `description` is a nearby pattern but do not confuse user-facing description with private note [packages/lib/schemaValidation.ts:129](../../packages/lib/schemaValidation.ts#L129).
- `postLink` already writes `description` into Prisma data [apps/web/lib/api/controllers/links/postLink.ts:106-109](../../apps/web/lib/api/controllers/links/postLink.ts#L106).
- Search currently includes `description`; decide explicitly whether notes should be searchable [apps/web/lib/api/controllers/search/searchLinks.ts:173-178](../../apps/web/lib/api/controllers/search/searchLinks.ts#L173).

**What Could Go Wrong:**
- Forgetting Zod means UI compiles but API rejects or drops data.
- Forgetting display/query type assumptions can make the field appear missing.
- Making notes searchable may expose private intent in places users do not expect.

**Stretch Goal:** Add note editing to the existing link edit modal.

**Connects To:** Story 9, because both are full-stack domain extensions.

## Story 9: Add Collection-Level Default Tags
**Difficulty:** Hard  
**Estimated Time:** 10 hours  
**Skills You'll Practice:** data modeling, permissions, collection forms, link creation logic  
**The Story:** As a collection owner, I want default tags for a collection so that new links saved there are automatically categorized.

**Acceptance Criteria:**
- [ ] Collection can store default tag names or relations.
- [ ] Collection create/edit UI can manage default tags.
- [ ] Creating a link in that collection applies defaults plus explicitly chosen tags.
- [ ] Duplicate tag names are not created for the same owner.
- [ ] Member permissions remain respected.

**Files You'll Likely Touch:**  
`packages/prisma/schema.prisma` - model collection/tag relation or default tag storage [packages/prisma/schema.prisma:126-149](../../packages/prisma/schema.prisma#L126), [packages/prisma/schema.prisma:200-218](../../packages/prisma/schema.prisma#L200).  
`packages/lib/schemaValidation.ts` - collection schemas and link schema [packages/lib/schemaValidation.ts:210-241](../../packages/lib/schemaValidation.ts#L210), [packages/lib/schemaValidation.ts:125-148](../../packages/lib/schemaValidation.ts#L125).  
`apps/web/components/ModalContent/NewCollectionModal.tsx` - collection create form state and submit path [apps/web/components/ModalContent/NewCollectionModal.tsx:20-61](../../apps/web/components/ModalContent/NewCollectionModal.tsx#L20).  
`apps/web/components/ModalContent/EditCollectionModal.tsx` - collection edit form state and update mutation path for existing collections [apps/web/components/ModalContent/EditCollectionModal.tsx:19-53](../../apps/web/components/ModalContent/EditCollectionModal.tsx#L19).  
`apps/web/lib/api/controllers/collections/postCollection.ts` - persist defaults during create [apps/web/lib/api/controllers/collections/postCollection.ts:84-132](../../apps/web/lib/api/controllers/collections/postCollection.ts#L84).  
`apps/web/lib/api/controllers/links/postLink.ts` - merge default tags with submitted tags [apps/web/lib/api/controllers/links/postLink.ts:120-137](../../apps/web/lib/api/controllers/links/postLink.ts#L120).

**High-Level Implementation Plan:**
1. Choose model design: relation table or string array, then update Prisma.
2. Update collection create/update schemas.
3. Add tag selection UI to collection create/edit modals.
4. Persist defaults in collection controllers.
5. In `postLink`, after `setCollection`, load collection defaults and merge with submitted tags.
6. Use existing `connectOrCreate` pattern to avoid duplicate tags.
7. Add tests around default + explicit tag merge.

**Tips:**
- `Tag` is unique by name and ownerId [packages/prisma/schema.prisma:216-217](../../packages/prisma/schema.prisma#L216).
- `postLink` uses `linkCollection.ownerId` when connecting/creating tags [apps/web/lib/api/controllers/links/postLink.ts:120-137](../../apps/web/lib/api/controllers/links/postLink.ts#L120).
- `setCollection` already ensures create permission [apps/web/lib/api/setCollection.ts:24-38](../../apps/web/lib/api/setCollection.ts#L24).

**What Could Go Wrong:**
- Default tags could be created under the wrong owner for shared collections.
- Merging tags by object identity instead of normalized name can duplicate tags.
- Collection members may see defaults they cannot edit; UI must respect permissions.

**Stretch Goal:** Add a setting to optionally apply defaults to existing links in the collection.

**Connects To:** Story 10, because default tags can interact with caching/search/indexing.

## Story 10: Build A Link Preservation Status Center
**Difficulty:** Expert  
**Estimated Time:** 16+ hours  
**Skills You'll Practice:** architecture design, caching, worker/API boundaries, performance, UX  
**The Story:** As a power user, I want a preservation status center so that I can see which links are waiting, complete, unavailable, or failed across my archive.

**Acceptance Criteria:**
- [ ] New UI view summarizes preservation status counts by format.
- [ ] Data is fetched from an authenticated API endpoint.
- [ ] Endpoint avoids loading full `textContent`.
- [ ] UI updates without aggressive polling.
- [ ] Design accounts for worker-updated fields.
- [ ] Tests cover API response shape and at least one permission boundary.

**Files You'll Likely Touch:**  
`packages/prisma/schema.prisma` - preservation fields live on `Link` [packages/prisma/schema.prisma:183-192](../../packages/prisma/schema.prisma#L183).  
`apps/worker/worker.ts` and worker files - preservation/indexing updates happen in worker jobs [apps/worker/worker.ts:15-19](../../apps/worker/worker.ts#L15).  
`apps/web/pages/api/v1/worker/index.ts` - existing authenticated admin-only worker stats route pattern [apps/web/pages/api/v1/worker/index.ts:1-20](../../apps/web/pages/api/v1/worker/index.ts#L1).  
`apps/web/lib/api/controllers/worker/getWorkerStats.ts` - existing worker stats controller that counts pending/done/failed preservation and search work [apps/web/lib/api/controllers/worker/getWorkerStats.ts:5-69](../../apps/web/lib/api/controllers/worker/getWorkerStats.ts#L5).  
`packages/router/worker.tsx` or new router hook - existing worker query/mutation pattern [packages/router/worker.tsx:9-36](../../packages/router/worker.tsx#L9).  
`apps/web/pages/settings/worker.tsx` - currently redirects worker settings to admin background jobs [apps/web/pages/settings/worker.tsx:1-14](../../apps/web/pages/settings/worker.tsx#L1).  
`apps/web/pages/admin/background-jobs.tsx` - existing admin background jobs UI that reads `useWorker` stats and shows pending/done counts [apps/web/pages/admin/background-jobs.tsx:16-35](../../apps/web/pages/admin/background-jobs.tsx#L16), [apps/web/pages/admin/background-jobs.tsx:87-100](../../apps/web/pages/admin/background-jobs.tsx#L87).

**High-Level Implementation Plan:**
1. Inspect existing worker stats endpoint and controller.
2. Design a new summary query over `Link` preservation fields: `preview`, `image`, `pdf`, `readable`, `monolith`, `lastPreserved`, `indexVersion`.
3. Add authenticated API endpoint with `verifyUser`.
4. Ensure Prisma query omits `textContent` and filters only accessible collections.
5. Add router hook with a conservative refetch interval or manual refresh.
6. Build UI in settings or a dedicated status page.
7. Add Vitest coverage for response shape and inaccessible collection exclusion.
8. Document caching and polling decisions in the PR.

**Tips:**
- Existing link queries intentionally omit `textContent` [apps/web/lib/api/controllers/search/searchLinks.ts:126-139](../../apps/web/lib/api/controllers/search/searchLinks.ts#L126), [apps/web/lib/api/controllers/search/searchLinks.ts:228-240](../../apps/web/lib/api/controllers/search/searchLinks.ts#L228).
- Worker stats already have a shared type shape [packages/types/global.ts:214-224](../../packages/types/global.ts#L214).
- Avoid naive 5-second polling for the whole archive; current link-list polling is scoped to visible pending previews [apps/web/components/LinkViews/Links.tsx:405-425](../../apps/web/components/LinkViews/Links.tsx#L405).
- Permission filters should mirror owner/member access [apps/web/lib/api/controllers/search/searchLinks.ts:197-213](../../apps/web/lib/api/controllers/search/searchLinks.ts#L197).

**What Could Go Wrong:**
- Loading every link row with large text fields can hurt performance.
- Polling too often can punish large self-hosted instances.
- Counting shared collections incorrectly can leak status for inaccessible links.

**Stretch Goal:** Add a manual "retry failed preservation" action that queues work through an existing worker route.

**Connects To:** This is the capstone story; it combines data modeling, API design, worker awareness, caching, and UX.
