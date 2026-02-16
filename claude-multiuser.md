# Command Center — Multi-User Mode Implementation

## Objective

Transform Command Center from a single-user app into a multi-user app. Multiple people log in with their Google account, each sees their own workspace (apps, projects, deploy history, sessions), and secrets stay in their browser. New users get a guided setup wizard.

## Current State

### Already UID-Scoped (No Changes Needed)
All v8+ Firebase services write to `command-center/{uid}/...` and are fully isolated:
- IssueService, WorkItemService, SessionService, ActivityLogService
- WorkStreamService, StreamInterfaceService, DependencyService, DependencyAlertService
- ConceptManager, IdeaManager, CompletionFileService, OrphanDetectionService
- TeamService (deferred — infrastructure exists but not being enabled yet)

### NOT UID-Scoped (Must Fix)
**FirebaseConfigSync** writes to `command-center/{dataKey}` instead of `command-center/{uid}/{dataKey}`. Affected data:
- `config` — app definitions, projects, repos, version tracking
- `deploy-history` — full deployment log
- `session-log` — development session records
- `rollback-snapshots` — pre-deploy state captures
- `rules-history` and `deletion-history`

### Secrets
Stored in localStorage, excluded from Firebase via PROTECTED_KEYS. Correct pattern but not namespaced by UID — two users on the same browser would share secrets.

---

## Phase 1: FirebaseConfigSync UID Scoping (P0)

**What:** Add a `uid` property to FirebaseConfigSync and inject it into all Firebase ref paths.

**Changes:**
1. Add `uid` property and `setUid(uid)` method to the FirebaseConfigSync object
2. Update `init(database)` — it should no longer auto-enable sync; wait for `setUid()` call
3. Update ALL `ref()` calls — there are ~5-6 of them (push, pull, clearAll, getDataSize, recovery check):
   ```js
   // Before
   this.db.ref(`${this.BASE_PATH}/${dataKey}`)
   
   // After
   if (!this.uid) return;
   this.db.ref(`${this.BASE_PATH}/${this.uid}/${dataKey}`)
   ```
4. Wire into `onAuthStateChanged` — call `FirebaseConfigSync.setUid(u.uid)` on sign-in, `setUid(null)` on sign-out

**Migration (one-time for Dave's existing data):**
1. On first sign-in after this change, check if `command-center/{uid}/config` exists
2. If not, check if `command-center/config` exists (legacy global data)
3. If legacy data found, copy all data keys to `command-center/{uid}/...`
4. Optionally delete legacy global data after confirmed migration

**Testing:**
- Dave signs in → existing global config migrates to UID-scoped path automatically
- All deploy history, session log, rollback snapshots appear correctly
- No data loss

---

## Phase 2: localStorage Secret Namespacing (P0)

**What:** Create a `SecretManager` utility that namespaces all secret localStorage keys by Firebase UID.

**Changes:**
1. Create `SecretManager` object with:
   - `SecretManager.uid` — active UID, set from auth
   - `SecretManager.setUid(uid)` — called from `onAuthStateChanged`
   - `SecretManager.get(key)` — reads `cc_{key}_{uid}` from localStorage
   - `SecretManager.set(key, value)` — writes `cc_{key}_{uid}` to localStorage
   - `SecretManager.has(key)` — checks existence (used by wizard gating)
2. Update ALL secret read/write callsites to use SecretManager instead of raw localStorage

**Affected keys:**
```
cc_token            →  cc_token_{uid}
cc_token_expires    →  cc_token_expires_{uid}
cc_api_key          →  cc_api_key_{uid}
cc_anthropic_api_key →  cc_anthropic_api_key_{uid}
cc_firebase_sa      →  cc_firebase_sa_{uid}
cc_firebase_uid     →  cc_firebase_uid_{uid}
```

**Migration:** On first sign-in, check for un-namespaced keys. If found and no namespaced key exists, copy the value to the namespaced key. Leave old keys in place temporarily.

**Testing:**
- Sign out User A, sign in User B on same browser → User B does NOT see User A's GitHub token
- Sign back in as User A → secrets still there

---

## Phase 3: First-Run Setup Wizard (P0)

**What:** A guided onboarding flow for new users. Gates the main UI until minimum credentials are established.

**Entry condition — three-state render at App level:**
```js
if (!firebaseUid)                    → Show sign-in screen
if (firebaseUid && !hasRequiredKeys) → Show setup wizard
if (firebaseUid && hasRequiredKeys)  → Show main CC UI
```
`hasRequiredKeys` = `SecretManager.has('token')` — GitHub token is the minimum viable credential.

### Wizard Steps

**Step 1 — Welcome:**
Brief intro: "Set up your Command Center workspace." Shows user's Google avatar and name from Firebase auth. Single "Get Started" button.

**Step 2 — GitHub Token (Required):**
- Input field for Personal Access Token
- Link to GitHub's token creation page
- Guidance: "Create a fine-grained token with repo and pages access"
- "Validate" button hits GitHub API, shows username/avatar on success
- Cannot proceed without valid token

**Step 3 — API Keys (Optional):**
- Anthropic API key field (enables AI features)
- Any other service keys
- Skip button available
- Brief explanation of what each key enables

**Step 4 — Add Your First App (P1, can be deferred):**
- Lists repos from validated GitHub token
- User selects repos to register as CC apps
- Auto-detect project grouping from repo naming conventions
- Skip to add later

**Step 5 — Done:**
- Summary of what was configured
- "Launch Command Center" button saves everything and loads main UI

### Design Notes
- Full-screen overlay, same dark theme as main CC UI
- Progress indicator (step dots) at the top
- Back/Next navigation with validation gating on required steps
- All data saved to UID-namespaced localStorage on completion
- Returning users who clear localStorage re-enter the wizard

---

## Phase 4: Auth Flow Updates (P0)

**What:** Wire the new components into the existing `onAuthStateChanged` handler.

### Sign-In Flow
1. User clicks "Sign In" → Google popup via Firebase
2. `onAuthStateChanged` fires with user object
3. `setFirebaseUid(u.uid)`
4. `SecretManager.setUid(u.uid)`
5. `FirebaseConfigSync.setUid(u.uid)`
6. Check for legacy un-namespaced secrets → migrate if present
7. Evaluate `hasRequiredKeys`:
   - No → render setup wizard
   - Yes → `FirebaseConfigSync.pullAll()`, load main UI
8. All v8+ service listeners already subscribe via uid (no changes needed)

### Sign-Out Flow
1. Clear all React state (already implemented)
2. `SecretManager.setUid(null)`
3. `FirebaseConfigSync.setUid(null)`
4. Do NOT clear localStorage — secrets persist for next sign-in

---

## Phase 5: Firebase Security Rules (P0)

Deploy rules to enforce data isolation at the database level:

```json
{
  "rules": {
    "command-center": {
      "$uid": {
        ".read": "auth.uid === $uid",
        ".write": "auth.uid === $uid",
        "teamMembership": {
          ".read": "auth.uid === $uid",
          ".write": "auth != null"
        }
      }
    }
  }
}
```

Each user can only read/write their own data. The `teamMembership` path allows any authenticated user to write (needed for future team invites).

---

## Phase 6: Testing (P0)

### Migration
- [ ] Dave signs in → existing global config migrates to UID-scoped path
- [ ] Deploy history, session log, rollback snapshots all correct
- [ ] Existing localStorage secrets migrate to namespaced keys
- [ ] No data loss

### New User Experience
- [ ] New Google account signs in → setup wizard appears
- [ ] GitHub token validation works (shows username/avatar)
- [ ] Skipping optional steps works
- [ ] After wizard, main UI loads with selected apps
- [ ] Sign out and back in skips wizard (secrets persist)

### User Isolation
- [ ] User A's apps/config/history not visible to User B
- [ ] User A deploying doesn't affect User B's history
- [ ] Same browser: sign out A, sign in B → B sees only their data
- [ ] Work items, sessions, activity logs are independent

### Edge Cases
- [ ] User clears localStorage → re-enters wizard
- [ ] User signs in on new browser → wizard for secrets, Firebase data pulls correctly
- [ ] Firebase offline/error → app works from localStorage cache
- [ ] Sign-in while wizard active doesn't create duplicate state

---

## Effort Estimates

| Phase | Work | Effort |
|-------|------|--------|
| Phase 1 | ConfigSync UID scoping + migration | 1–2 hrs |
| Phase 2 | SecretManager + key namespacing | ~1 hr |
| Phase 3 | Setup wizard (steps 1–3) | 2–3 hrs |
| Phase 3b | Wizard step 4: repo picker (P1) | 1–2 hrs |
| Phase 4 | Auth flow integration | ~1 hr |
| Phase 5 | Firebase security rules | ~30 min |
| Phase 6 | Testing with second account | 1–2 hrs |
| **Total** | | **7–11 hrs** |

---

## Out of Scope (Future)

- **Team workspace sharing** — TeamService is built, enable later
- **Workspace switching UI** — needed when users can own + be member of workspaces
- **Per-app deploy permissions** — only matters at scale
- **Server-side SA key** — only needed for untrusted users; localStorage is fine for known testers
