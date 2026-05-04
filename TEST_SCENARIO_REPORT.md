# Test Scenario Report: CentralView Dashboard Applications
## High-Risk Area Analysis & Recommended Test Coverage

---

## 1. VAULT & SESSION TOKEN MANAGEMENT (index.html)

### Risk Summary
The vault system uses AES-GCM encryption with PBKDF2 key derivation to store API tokens in localStorage. Tokens are decrypted into sessionStorage on unlock and passed to child dashboards. Failures here compromise security and cross-page functionality.

### Critical Areas to Test

#### 1.1 Vault Encryption/Decryption Cycle
**Risk:** Incorrect key derivation, IV handling, or salt management could result in unrecoverable tokens or security vulnerabilities.

**Test Cases:**
- **Encrypt-Decrypt Roundtrip**: Save token "test_abc123", unlock with correct password, verify token value matches exactly
  - Type: Unit
  - Inputs: Plain token string, master password
  - Expected: Decrypted token = original token
  
- **Incorrect Password Rejection**: Save token with password "pass1", attempt unlock with "pass2", verify error message and vault remains locked
  - Type: Unit
  - Inputs: Token, incorrect password
  - Expected: "Incorrect password" error, vault stays locked
  
- **Empty Vault Unlock**: Unlock empty vault (no tokens saved), verify no error and vault opens successfully
  - Type: Unit
  - Inputs: Master password, zero tokens
  - Expected: Vault opens, "No tokens yet" message shown
  
- **Multi-Token Same Type Replacement**: Save GitHub token "token_old", save new GitHub token "token_new" with same type, verify only new token stored
  - Type: Unit
  - Inputs: Two tokens with same type tag
  - Expected: Old token removed, new token accessible via getByType()

#### 1.2 Vault Unlock and Auto-Assignment Flow
**Risk:** Tokens may not be properly assigned to tiles, or old assignments may persist after lock/unlock cycles.

**Test Cases:**
- **Auto-Assign on Unlock**: Save GH + ZH tokens both with correct types, unlock vault, verify both tiles render token chips
  - Type: Integration
  - Inputs: Vault with typed tokens, unlock password
  - Expected: Tile chips show assigned token names, loadAssignments returns populated mappings
  
- **Tile Readiness Check**: Tile config requires ['gh', 'zh'], only 'gh' assigned, verify tileIsReady() returns false
  - Type: Unit
  - Inputs: Partial token assignment, tile config
  - Expected: tileIsReady returns false, "needs: ZenHub" message on click
  
- **Lock Clears Assignments**: Unlock vault → auto-assign → lock vault, verify loadAssignments is cleared
  - Type: Integration
  - Inputs: Locked state operation sequence
  - Expected: Assignments empty after lock, vault icon returns to 🔒

#### 1.3 Session Storage Token Passing
**Risk:** Tokens stored in sessionStorage could persist across page navigations or be leaked if session not properly cleared.

**Test Cases:**
- **Token Set in Session on Launch**: Launch tile with GH + ZH tokens, verify sessionStorage contains 'cv_session_gh' and 'cv_session_zh'
  - Type: Integration
  - Inputs: Vault unlocked, tile launch
  - Expected: sessionStorage keys exist with decrypted values
  
- **Lock Removes Session Password**: Unlock vault → lock vault, verify 'cv_vault_pw' cleared from sessionStorage
  - Type: Integration
  - Inputs: Lock action
  - Expected: sessionStorage.getItem('cv_vault_pw') returns null
  
- **Dashboard Checks Token Presence**: Navigate to dashboard with no sessionStorage token, verify "Tokens required" error shown
  - Type: E2E
  - Inputs: Direct navigation without unlock
  - Expected: Error modal renders, link to index.html shown

---

## 2. PROFILE SYSTEM & REPOSITORY CONFIGURATION

### Risk Summary
Profile system stores repository and ZenHub workspace associations in localStorage. Multi-repo groups and per-repo workspace mappings introduce state complexity. Corrupted profiles can prevent dashboard load.

### Critical Areas to Test

#### 2.1 Profile Load and Validation
**Risk:** Invalid profile data, missing required fields, or corrupted JSON could crash dashboard or cause silent failures in API calls.

**Test Cases:**
- **Single Profile Auto-Load**: Create 1 profile for "owner/repo", load dashboard, verify S.repos = ["owner/repo"], no profile selector shown
  - Type: Integration
  - Inputs: Single valid profile in localStorage
  - Expected: Dashboard loads with correct repos array, profile selector hidden
  
- **Multiple Profile Picker Display**: Create 2 profiles, load dashboard, verify profile dropdown appears with both options
  - Type: UI/Integration
  - Inputs: Multiple profiles
  - Expected: Dropdown rendered, selection changes S.repos array
  
- **Zero Profile Setup Mode**: Clear all profiles, load dashboard, verify setup screen shown with "Create new profile" option
  - Type: Integration
  - Inputs: No profiles
  - Expected: Setup UI displayed, can create new profile
  
- **Corrupt JSON Recovery**: Save malformed profile JSON to localStorage, load dashboard, verify loads with empty array (no crash)
  - Type: Integration
  - Inputs: Invalid JSON in PROFILES_KEY
  - Expected: loadProfiles() returns [], no exception thrown
  
- **Workspace ID Per-Repo Mapping**: Create group profile with repos [R1 (ws1), R2 (ws2)], verify S.repoWorkspaceMap['R1'] = 'ws1' and S.repoWorkspaceMap['R2'] = 'ws2'
  - Type: Unit
  - Inputs: Group profile with per-repo workspaces
  - Expected: repoWorkspaceMap populated correctly, ZH API calls use correct workspace

#### 2.2 Profile Edit Operations
**Risk:** Editing profiles with prompts can accidentally corrupt state if user cancels mid-dialog or enters invalid data.

**Test Cases:**
- **Edit Cancel Mid-Flow**: Start editing profile, cancel at repo prompt, verify profile unchanged
  - Type: E2E
  - Inputs: Edit action, cancel dialog
  - Expected: localStorage profile unchanged, S.activeProfile unchanged
  
- **Edit Triggers ZH Reload**: Edit profile and change workspace ID, verify zhLoad() called automatically
  - Type: Integration
  - Inputs: Workspace ID change action
  - Expected: zhLoad() invoked, "ZenHub loading..." status shown
  
- **Edit Validates Required Fields**: Try to save profile with empty name, verify error shown and save blocked
  - Type: E2E
  - Inputs: Incomplete form submission
  - Expected: Alert shown, localStorage unchanged

---

## 3. GITHUB API INTEGRATION & PAGINATION (All Dashboards)

### Risk Summary
`ghFetchAll()` implements pagination with hardcoded maxPages=10 (1000 issues). Missing or incomplete data can cause silent filtering errors. Authorization header failures are critical.

### Critical Areas to Test

#### 3.1 Pagination Behavior
**Risk:** Repositories with >1000 issues will silently cap at page 10, causing data loss without warning.

**Test Cases:**
- **Pagination Stops at Max Pages**: Call ghFetchAll('/repos/owner/repo/issues', 2) with 150 issues, verify stops at page 2, returns 200 items
  - Type: Unit
  - Inputs: maxPages=2, 150 issues per page
  - Expected: Fetch calls pages 1-2 only, result length <= 200
  
- **Empty Page Stops Iteration**: Call ghFetchAll with repo returning 50 items page 1, empty page 2, verify stops iterating
  - Type: Integration
  - Inputs: API returns empty array on page 2
  - Expected: Loop exits, result includes only page 1 data
  
- **1000+ Issues Warning**: Load dashboard with repo having 1050 issues, verify warning displayed (or log message if no UI)
  - Type: Integration
  - Inputs: Real repo or mock with >1000 issues
  - Expected: Developer console shows warning or UI badge appears

#### 3.2 Token Authorization
**Risk:** Expired or invalid token will cause all API calls to fail. Missing Authorization header causes 404 instead of 401.

**Test Cases:**
- **Invalid Token Rejection**: Set S.ghToken = "invalid_token_12345", call ghFetch('/user'), verify error thrown with "401" or "403" in message
  - Type: Integration
  - Inputs: Invalid token
  - Expected: Fetch rejects with 401/403 error
  
- **Missing Token Fallback**: Set S.ghToken = "", call ghFetch(), verify header not sent or fallback to public API (limit 60 req/hr)
  - Type: Integration
  - Inputs: Empty token
  - Expected: Either error thrown or calls made without token (rate-limited)
  
- **Header Format Correct**: Mock fetch, call ghFetch('/repos/owner/repo'), verify Authorization header is "token <value>" not "Bearer"
  - Type: Unit
  - Inputs: Valid token
  - Expected: Fetch headers include 'Authorization: token <ghToken>'

#### 3.3 Error Handling in Parallel Loads
**Risk:** Promise.all() in session bootstrap means any fetch failure (label colors, issues, milestones) blocks entire load.

**Test Cases:**
- **One Parallel Fetch Fails**: Load sequence: loadLabelColors() succeeds, loadIssues() fails, loadMilestones() succeeds, verify whole sequence rejected
  - Type: Integration
  - Inputs: One failed promise in Promise.all()
  - Expected: .catch() triggered, error message shown in mainContent
  
- **Network Timeout Handling**: Mock fetch with 30s timeout on one repo, verify timeout error is caught and user sees message (not hung UI)
  - Type: E2E
  - Inputs: Slow/timeout network request
  - Expected: Error UI shown, not infinite loading spinner
  
- **Milestones Optional**: If loadMilestones() fails, but issues load, verify dashboard still usable (milestones = {})
  - Type: Integration
  - Inputs: Milestone fetch failure
  - Expected: Issues render, milestone columns empty/disabled

---

## 4. ZENHUB API INTEGRATION & PIPELINE OPERATIONS

### Risk Summary
ZenHub pipeline moves return 204 No Content. S.zhIssueMap uses "issueNumber_repo" keys for O(1) lookups. Workspace ID mismatches or network failures during movePipeline can leave issues in incorrect state.

### Critical Areas to Test

#### 4.1 Pipeline Data Mapping
**Risk:** Incorrect key format in zhIssueMap will cause lookups to fail silently, leaving issues without pipeline info.

**Test Cases:**
- **zhIssueMap Key Format**: Fetch board with issue #42 in repo "org/repo", verify S.zhIssueMap contains key "42_org/repo" not "42" or "org/repo_42"
  - Type: Unit
  - Inputs: ZenHub board response with issues
  - Expected: zhIssueMap['42_org/repo'] = {pipelineId, pipelineName, workspace}
  
- **Null Pipeline Handling**: Issue has no pipeline in ZenHub, load board, verify issue.pipeline is null/undefined, not {pipelineName: ""}
  - Type: Integration
  - Inputs: ZenHub board with unpipelined issues
  - Expected: issue.pipeline is falsy, exception checks skip gracefully
  
- **Multi-Repo Pipeline Separation**: Load 2 repos, both have issue #1, verify zhIssueMap distinguishes "1_repo1" from "1_repo2"
  - Type: Integration
  - Inputs: Two repos with duplicate issue numbers
  - Expected: Both keyed separately, mapped correctly in S.issues array

#### 4.2 Move Pipeline Operation
**Risk:** movePipeline() updates local state but if API fails, state and remote become inconsistent. Position='top' assumed but not validated.

**Test Cases:**
- **Successful Move Updates Local State**: Move issue #10 to pipeline "In Progress", verify S.issues[idx].pipeline.pipelineName = "In Progress"
  - Type: Integration
  - Inputs: Valid issue, valid pipeline ID
  - Expected: Local state updated, renderMain() shows new pipeline
  
- **Move Failure Reverts State**: Mock zhPost to fail after state update, verify issue reverts to old pipeline
  - Type: Integration
  - Inputs: Move operation with API failure
  - Expected: Alert shown, issue.pipeline remains unchanged
  
- **Missing Workspace ID**: Set S.repoWorkspaceMap[repo] = "" (empty), call movePipeline(), verify early return (no API call)
  - Type: Unit
  - Inputs: Missing workspace ID
  - Expected: Function returns early, no fetch call made
  
- **204 No Content Handling**: zhPost returns 204, verify code doesn't try to parse response body as JSON
  - Type: Unit
  - Inputs: 204 response
  - Expected: zhPost() returns {}, no JSON.parse error

#### 4.3 Bulk Move Operation
**Risk:** bulkMove() loops calling movePipeline() serially (await in loop). Network latency or partial failures could leave half the issues moved.

**Test Cases:**
- **Bulk Move Confirmation**: Select 5 issues, click bulk move to pipeline X, verify confirm() shows "Move 5 issue(s)"
  - Type: UI/Integration
  - Inputs: Multiple issues, bulk move action
  - Expected: Confirmation modal shows correct count
  
- **Bulk Move Partial Failure**: Move 3 issues, 1st succeeds, 2nd fails, 3rd called, verify alert shown and state is 1 moved + 2 unmoved
  - Type: Integration
  - Inputs: Failing API on 2nd call
  - Expected: Error alert after failure, remaining calls continue
  
- **Bulk Move Reset Dropdown**: After bulk move, select element value reset to "", verify user must re-select to move again
  - Type: UI
  - Inputs: Bulk move completion
  - Expected: select.value = '', user can't accidentally duplicate move

#### 4.4 ZenHub Data Load Errors
**Risk:** S._zhErrors array can grow unbounded. Partial failures (1 of 2 repos) could be silently ignored if error display buggy.

**Test Cases:**
- **ZH Load with 1 Repo Error**: 2 repos configured, ZH API fails for repo 1 but succeeds for repo 2, verify S._zhErrors contains repo 1 error and repo 2 still mapped
  - Type: Integration
  - Inputs: One repo ZH API failure
  - Expected: Error logged, other repo's issues mapped, user sees partial warning
  
- **Zero Issues Mapped Warning**: ZH succeeds but board returns empty pipelines, verify status shows "0 issues mapped — verify Workspace ID"
  - Type: Integration
  - Inputs: Empty ZH board
  - Expected: Status message alerts user to check configuration
  
- **ZH Connection Without Workspace ID**: Token present, but workspace ID not set, verify status shows "ZenHub token found but no Workspace ID" warning
  - Type: Integration
  - Inputs: Token but no workspace
  - Expected: User directed to edit profile to add workspace

---

## 5. EXCEPTION DETECTION LOGIC (All Dashboards)

### Risk Summary
Exception logic is complex with multiple branches per issue type. Incorrect branching can cause issues to show/hide warnings unexpectedly. Logic differs between CIS, Power Search, and Customer Portal.

### Critical Areas to Test

#### 5.1 Customer Portal Exception Logic
**Risk:** issueHasException() branches on isCsTaskItem → isEPA → general checks. Pipeline state required. Dismissed warnings can mask real issues.

**Test Cases:**

**CS Task Item Branch:**
- **CS Task Item in Wrong Pipeline**: Issue has "cs task item" label, pipeline = "On Deck Priorities" (not "CS Tracking Issues"), verify issueHasException() = true
  - Type: Unit
  - Inputs: CS Task Item, wrong pipeline
  - Expected: isCsTaskItemNotTracking() = true, exception shown
  
- **CS Task Item Correct Pipeline**: Issue has "cs task item" label, pipeline = "CS Tracking Issues", verify issueHasException() = false
  - Type: Unit
  - Inputs: CS Task Item, correct pipeline
  - Expected: No exception raised
  
- **CS Task Item Warning Dismissed**: Issue flagged, user clicks Dismiss, reload, verify S.warnDismissed contains key and issueHasException() = false
  - Type: Integration
  - Inputs: Exception issue, dismiss action
  - Expected: Key added to array, exception check returns false

**EPA Branch:**
- **EPA Not in New Issues**: Issue has "enhancement pending approval" label, pipeline = "Backlog" (not "New Issues"), verify isEPANotInNewIssues() = true
  - Type: Unit
  - Inputs: EPA label, wrong pipeline
  - Expected: Exception raised, warning shown
  
- **EPA Missing Milestone**: Issue has ["cs item", "enhancement pending approval"], milestone != "Orbit One Enhancements Pending Approval", verify isEPAMilestoneMissing() = true
  - Type: Unit
  - Inputs: EPA + CS Item, missing milestone
  - Expected: Exception raised
  
- **EPA Milestone Orphan**: Issue in "Orbit One Enhancements Pending Approval" milestone but missing both "cs item" and "enhancement pending approval" labels, verify isMilestoneOrphan() = true
  - Type: Unit
  - Inputs: Milestone set, labels missing
  - Expected: Exception raised for orphan milestone

**General Checks Branch:**
- **Assigned in New Issues**: Issue assigned to user, pipeline = "New Issues", verify isAssignedInNIP() = true, exception = true
  - Type: Unit
  - Inputs: Assigned issue in NIP
  - Expected: Exception raised
  
- **Assigned but Not In Progress**: Issue assigned, pipeline = "Backlog" (neither NIP nor In Progress), verify isAssignedNotInProgress() = true
  - Type: Unit
  - Inputs: Assigned issue, wrong pipeline
  - Expected: Exception raised
  
- **Unassigned Not On Deck**: Issue unassigned, not EPA, pipeline = "Backlog" (not "On Deck Priorities"), verify isUnassignedNotOnDeck() = true
  - Type: Unit
  - Inputs: Unassigned issue, wrong pipeline
  - Expected: Exception raised
  
- **Unassigned EPA Exemption**: Issue unassigned, EPA label, pipeline = "Backlog", verify isUnassignedNotOnDeck() returns false (EPA exempted)
  - Type: Unit
  - Inputs: Unassigned EPA issue
  - Expected: isUnassignedNotOnDeck() = false, no exception for pipeline

#### 5.2 CIS Exception Logic
**Risk:** isException() has different branching: auto-checked parents, "cs in testing" label handling, tracking issue references. Each branch must be tested independently.

**Test Cases:**
- **CS In Testing with Auto-Checked Parent**: Issue has "cs in testing" label, parent found via findAutoCheckedParents(), verify isException() = true
  - Type: Unit
  - Inputs: CS In Testing issue with parent
  - Expected: Exception = true due to parent check
  
- **CS Item Not Referenced in Tracking**: Issue has "cs item" label, isReferencedInTracking(id) = false, verify isException() = true
  - Type: Unit
  - Inputs: CS Item not in tracking issue body
  - Expected: Exception raised
  
- **Still in New Issues Pipeline**: Issue open, not exempt, pipeline = "New Issues", verify isException() = true
  - Type: Unit
  - Inputs: NIP issue
  - Expected: Exception = true
  
- **CS Task Item Excluded**: Issue has "cs task item" label (regardless of pipeline), verify isException() = false
  - Type: Unit
  - Inputs: CS Task Item
  - Expected: isException() = false (early return in function)

#### 5.3 Exception Count and Stat Badges
**Risk:** exceptionCount() filters by issueHasException(). Stat tile badge can overflow or show stale count if cache not refreshed.

**Test Cases:**
- **Exception Count Accuracy**: Load 10 issues, 3 have exceptions, verify exceptionCount(issues) = 3, badge shows "3"
  - Type: Integration
  - Inputs: 10 issues, 3 with exceptions
  - Expected: Badge displays "3"
  
- **All Clear Badge**: Load 10 issues with no exceptions, verify exceptionCount = 0, no red badge shown
  - Type: UI/Integration
  - Inputs: All issues compliant
  - Expected: Badge hidden or shows "0" in neutral color
  
- **Exception Count After Dismiss**: Issue has exception, click dismiss, verify exceptionCount() recalculated and badge updated
  - Type: Integration
  - Inputs: Exception dismiss action
  - Expected: Count decrements, badge reflects new count

---

## 6. DATA CACHING & CACHE INVALIDATION

### Risk Summary
saveCache() and loadCache() persist issues to localStorage. Cache marked as "stale" after certain time or operations, but invalidation logic can be incomplete.

### Critical Areas to Test

#### 6.1 Cache Load on Startup
**Risk:** Loading stale cache without validation can show outdated issue state, especially pipeline and label assignments.

**Test Cases:**
- **Valid Cache Loaded**: Save cache with 5 issues, reload dashboard, verify S.issues populated immediately and "Cached (Xm ago)" shown
  - Type: Integration
  - Inputs: Valid cache in localStorage
  - Expected: Dashboard renders from cache, background refresh triggered
  
- **Missing parentsLoaded Flag**: Cache has issues but parentsLoaded = false, verify cache is ignored and fresh load triggered
  - Type: Integration
  - Inputs: Cache without parentsLoaded
  - Expected: "Loading..." overlay shown, cache bypassed
  
- **Cache Corruption Recovery**: Save malformed cache JSON, reload, verify not used and fresh load starts
  - Type: Integration
  - Inputs: Invalid cache JSON
  - Expected: loadCache() returns null, fresh load

#### 6.2 Cache Invalidation
**Risk:** Some operations (refresh, profile change) should clear cache but might not.

**Test Cases:**
- **Refresh Clears Cache**: Click refresh button, verify localStorage CACHE_KEY removed
  - Type: Integration
  - Inputs: Refresh action
  - Expected: localStorage[STORAGE_KEY] cleared
  
- **Profile Change Doesn't Auto-Clear**: Switch profile, verify old cache not used (new repos loaded instead)
  - Type: Integration
  - Inputs: Profile switch
  - Expected: Dashboard loads issues for new repos, cache invalidated implicitly
  
- **Closed Issues in Cache**: Close an issue on GitHub, cache loaded, verify closed issue appears in closedIssues, not open
  - Type: Integration
  - Inputs: Closed issue state in GitHub
  - Expected: Cache updated on next bgRefresh()

---

## 7. FILTER APPLICATION & TAB RENDERING

### Risk Summary
Multiple filter types (labels, assignee, milestone) and tabs (all, closed, ignored, epa, enhancement, csother for CP; all, closed, taskitem, csitem for CIS/PS). Filter state can leak across tabs.

### Critical Areas to Test

#### 7.1 Tab-Specific Filtering
**Risk:** applyFilters() applied to tab-specific source array, but if source array wrong, filtering will miss/show wrong issues.

**Test Cases:**

**Customer Portal:**
- **EPA Tab Filter**: Click EPA tab, verify shown issues = issues with "enhancement pending approval" label AND not ignored, count matches tabCount('epa')
  - Type: Integration
  - Inputs: EPA tab selection
  - Expected: Correct subset rendered
  
- **Enhancement Tab**: Click Enhancement tab, verify shown issues have "enhancement" label but NOT "enhancement pending approval", grouped by pipeline
  - Type: Integration
  - Inputs: Enhancement tab selection
  - Expected: Correct grouping by pipeline, sorted by creation date within group
  
- **CS Items Tab**: Verify shown issues have CS labels but not EPA/Enhancement, NOT in ignored list
  - Type: Integration
  - Inputs: CS Items tab selection
  - Expected: Correct subset

**CIS / Power Search:**
- **CS Task Item Tab**: Verify shown issues have "cs task item" label, not ignored
  - Type: Integration
  - Inputs: Task Item tab selection
  - Expected: Only CS Task Items shown
  
- **CS Item Tab**: Verify shown issues have "cs item" label (but not "cs task item"), not ignored
  - Type: Integration
  - Inputs: CS Item tab selection
  - Expected: Only CS Items shown

#### 7.2 Ignore/Restore Functionality
**Risk:** toggleIgnore() adds/removes key from S.ignored. Incorrect key format can fail to hide issue.

**Test Cases:**
- **Ignore Key Format**: Click ignore on issue #5 in "owner/repo", verify key "5_owner/repo" added to S.ignored
  - Type: Unit
  - Inputs: Issue ignore action
  - Expected: S.ignored array contains correct key
  
- **Issue Hidden After Ignore**: Ignore issue, verify disappears from current view, re-renders correctly
  - Type: UI/Integration
  - Inputs: Ignore action
  - Expected: Issue removed from rendered list
  
- **Ignored Tab Shows Only Ignored**: Click Ignored tab, verify only issues with keys in S.ignored array shown
  - Type: Integration
  - Inputs: Ignored tab view
  - Expected: Count matches S.ignored.length
  
- **Restore Clears Ignored Key**: Click restore on ignored issue, verify key removed from S.ignored, issue reappears in main tabs
  - Type: Integration
  - Inputs: Restore action
  - Expected: Issue visible again, counts updated

#### 7.3 Multi-Label Grouping (by Label View)
**Risk:** byLabelView() creates label groups. Incorrect grouping can cause duplicates or omissions.

**Test Cases:**
- **Duplicate Issues in Multiple Groups**: Issue has 2 labels ["bug", "urgent"], verify appears in both "bug" and "urgent" groups (not duplicated in either)
  - Type: Integration
  - Inputs: Issue with multiple labels
  - Expected: Issue present in all applicable label groups
  
- **Unlabeled Group**: Issue has no labels, verify appears in "Unlabeled" section
  - Type: Integration
  - Inputs: Unlabeled issues
  - Expected: Separate "Unlabeled" section rendered
  
- **Label Color Lookup**: Label "custom-label" with no custom color in S.labelColors, verify fallback color 'e5e7eb' used
  - Type: Unit
  - Inputs: Unknown label
  - Expected: Fallback color applied

---

## 8. PROFILE & MILESTONE MANAGEMENT (Customer Portal)

### Risk Summary
setMilestone() calls GitHub PATCH, updates local issue.milestone, but if local update misses, issue state diverges from remote.

### Critical Areas to Test

#### 8.1 Milestone Setting
**Risk:** GitHub API call succeeds but local issue object not found (wrong ID or repo), leaving remote and local inconsistent.

**Test Cases:**
- **Milestone Set Successfully**: Set milestone 3 on issue #10, repo "org/repo", verify ghPatch called with milestone: 3, local issue.milestone updated to title
  - Type: Integration
  - Inputs: Valid issue, valid milestone
  - Expected: Local state updated, renderMain() shows new milestone
  
- **Milestone Cleared (null)**: Set milestone to "" (empty), verify ghPatch called with milestone: null
  - Type: Integration
  - Inputs: Milestone clear action
  - Expected: issue.milestone set to null, displayed as empty
  
- **Issue Not Found in Array**: Milestone set, but issue not found in S.issues (wrong ID), verify local update skipped (API call already succeeded, so remote changed)
  - Type: Unit
  - Inputs: Non-existent issue ID
  - Expected: Local array not modified, API call made (inconsistency, but safe)
  
- **API Failure Alert**: GitHub PATCH fails, verify alert shown and renderMain() called (state unchanged)
  - Type: Integration
  - Inputs: Failed API call
  - Expected: Alert shown, state unchanged

#### 8.2 Milestone Availability Per-Repo
**Risk:** S.milestones[repo] may be empty or undefined, affecting dropdown options.

**Test Cases:**
- **Milestone Dropdown Populated**: Load dashboard with milestones loaded, open milestone select, verify options include all loaded milestones for issue's repo
  - Type: UI/Integration
  - Inputs: Milestones loaded
  - Expected: Dropdown shows all milestones
  
- **No Milestones Loaded**: Milestone load fails or repo has none, verify dropdown disabled or shows "No milestones"
  - Type: Integration
  - Inputs: Empty milestones
  - Expected: Dropdown disabled or empty

---

## 9. POWER SEARCH: PARENT ISSUE TRACKING

### Risk Summary
Power Search tracks parent issues for sub-issues. isMissingParent() checks both issue.parent and body regex. Incomplete parent linking can incorrectly flag issues.

### Critical Areas to Test

#### 9.1 Parent Issue Detection
**Risk:** Parent detection via both object (issue.parent) and body regex. Regex mismatch or missing parent object causes false positives.

**Test Cases:**
- **Parent Via Object**: Issue has issue.parent = {issueNumber: 42}, isMissingParent() = false, no warning shown
  - Type: Unit
  - Inputs: Issue with parent object
  - Expected: No missing parent warning
  
- **Parent Via Body Regex**: Issue.parent = null, body contains "#42" (regex match), isMissingParent() should check, verify logic
  - Type: Unit
  - Inputs: Parent in body text
  - Expected: Regex parser identifies parent
  
- **CS Item No Parent**: CS Item issue, no parent object and body has no parent reference, verify isMissingParent() = true, warning shown
  - Type: Integration
  - Inputs: CS Item without parent
  - Expected: Warning: "no parent issue found"
  
- **CS Task Item Exempted**: CS Task Item issue with no parent, verify isMissingParent() = false (early return)
  - Type: Unit
  - Inputs: CS Task Item
  - Expected: No missing parent check

---

## 10. ASSIGNMENT LOGIC (Customer Portal / Power Search)

### Risk Summary
Assignee dropdown calls updateAssignee() (if exists) or updateLabels(). Changing assignee should update pipeline suggestions per exception logic.

### Critical Areas to Test

#### 10.1 Assignee Update
**Risk:** Assignee change may not trigger pipeline exception recalculation or display.

**Test Cases:**
- **Unassigned to Assigned**: Issue unassigned, assign to user "alice", verify issue now flagged as "Assigned but not in In Progress" if pipeline != In Progress
  - Type: Integration
  - Inputs: Assignee change action
  - Expected: Exception check re-runs, warning shown if applicable
  
- **Assigned to Unassigned**: Unassign user, verify "Unassigned but not on On Deck" check applies (if not EPA)
  - Type: Integration
  - Inputs: Unassignment action
  - Expected: Exception check updated
  
- **Assignee Persistence**: Change assignee, reload, verify persisted correctly in issue data
  - Type: Integration
  - Inputs: Assignee update, page reload
  - Expected: Assignee maintained

---

## 11. NETWORK & PERFORMANCE EDGE CASES

### Risk Summary
Large datasets (1000+ issues), slow networks, and concurrent requests can cause UI hangs or incomplete renders.

### Critical Areas to Test

#### 11.1 Large Dataset Handling
**Risk:** Rendering 1000+ issues in a table can freeze UI. No virtual scrolling implemented.

**Test Cases:**
- **1000 Issues Load**: Mock 1000 issues, load dashboard, measure render time, verify not >5 seconds (acceptable for one-time load)
  - Type: Performance
  - Inputs: 1000 issues
  - Expected: Renders within reasonable time
  
- **Table Sorting with 1000 Issues**: Sort by ID, verify sort completes quickly
  - Type: Performance
  - Inputs: Large dataset, sort action
  - Expected: Sort completes in <500ms

#### 11.2 Concurrent Request Handling
**Risk:** bgRefresh() calls multiple fetches. If one stalls, others may timeout or cancel.

**Test Cases:**
- **Partial Timeout on bgRefresh()**: During auto-refresh, one repo's issue fetch times out, others succeed, verify dashboard shows "partially updated" or correct count for successful repos
  - Type: Integration
  - Inputs: Timeout on one fetch
  - Expected: Other data loaded, partial update shown
  
- **Refresh Interrupt**: User clicks refresh while previous refresh is in progress, verify second request doesn't duplicate work
  - Type: Integration
  - Inputs: Rapid refresh clicks
  - Expected: Only one refresh in progress

---

## 12. LABEL & METADATA CONSISTENCY

### Risk Summary
Label colors loaded from GitHub API. If load fails, fallback colors used but may not reflect GitHub UI. Label name casing handled with toLowerCase() but could mismatch.

### Critical Areas to Test

#### 12.1 Label Name Casing
**Risk:** Exception checks use toLowerCase() for "cs item" comparison. If issue has "CS Item" (different case), might not match.

**Test Cases:**
- **Case-Insensitive Label Match**: Issue has label "CS Item" (capital), condition checks l.toLowerCase() === 'cs item', verify matches correctly
  - Type: Unit
  - Inputs: Label with different casing
  - Expected: Case-insensitive match succeeds
  
- **Label Display Name Preservation**: Load labels from GitHub, display uses S.labelNames (original case), verify UI shows correct casing even if internal checks use lowercase
  - Type: Integration
  - Inputs: Labels loaded with mixed casing
  - Expected: UI shows original casing, logic uses lowercase

#### 12.2 Color Fallback
**Risk:** Label color not found in S.labelColors, fallback 'e5e7eb' used. No warning if color load failed.

**Test Cases:**
- **Fallback Color On Missing**: Label "future-feature" not in S.labelColors, verify badge uses fallback color
  - Type: UI
  - Inputs: Unknown label
  - Expected: Fallback color applied, no crash

---

## Test Execution Priority

### Phase 1: Critical (Must Pass)
1. Vault encryption/decryption roundtrip
2. Exception detection logic (all three branches per dashboard)
3. GitHub API token authorization
4. Pipeline move operations
5. zhIssueMap key format and lookup
6. Profile load and validation
7. Cache load on startup
8. Tab filtering correctness

### Phase 2: High (Should Pass)
1. Multi-repo workspace mapping
2. Bulk move operation
3. ZenHub partial failures
4. Auto-refresh background behavior
5. Ignore/restore functionality
6. Assignee update and pipeline exception recalculation
7. Milestone setting persistence

### Phase 3: Medium (Nice to Have)
1. Large dataset performance
2. Concurrent request handling
3. Label color fallback
4. Closed exception logic (CIS)
5. Parent issue detection (Power Search)
6. Financials dashboard settings

---

## Testing Infrastructure Recommendations

1. **Mock Layers**: Create mock for fetch() to simulate GitHub/ZenHub API responses
2. **Fixture Data**: Prepare test issues with various label combinations, pipeline states, assignees
3. **State Assertions**: Helper functions to check S.issues array state, localStorage contents, DOM rendering
4. **Error Injection**: Ability to fail API calls at specific points to test error handling
5. **localStorage Isolation**: Clear localStorage before each test to prevent state leakage
6. **sessionStorage Isolation**: Clear session storage before E2E tests

