# How to submit a FIP (Fluss Improvement Proposal)

Reference: https://cwiki.apache.org/confluence/display/FLUSS/Fluss+Improvement+Proposals

This note is for the Delta Lake tiering proposal in this directory,
but the steps are the same for any FIP.

## 1. Get a wiki account

The FLUSS Confluence space does not allow self sign-up (spam).
Request an account through the ASF "Selfserve wiki signup" form.
It usually takes one or two days.

If you cannot wait, use the fallback in step 2b.

## 2a. Create the wiki page (with wiki permission)

1. Open the FLUSS space and press "Create".
2. Pick the "FIP-Template" template, not "Blank page".
3. Put the page under "Apache Fluss Home", next to the FIP index.
4. Use the next free number from the index. The draft in this
   directory uses 60, re-check the index at that time.
5. Copy the sections from the markdown into the template.
6. Set the stage label to "discussion".

## 2b. Fallback (no wiki permission)

1. Put the proposal into a Google Doc, view-only for everyone.
2. Title it `FIP-XXX: Support tiering Fluss data to Delta Lake`.
3. Share the link in the [DISCUSS] mail (step 3).
4. A committer or PMC member moves it to Confluence and assigns the number.

## 3. Start the discussion

1. Subscribe to the dev list if needed: send an empty mail to
   dev-subscribe@fluss.apache.org and reply to the confirmation.
2. Send a mail to dev@fluss.apache.org with subject
   `[DISCUSS] FIP-60 Support tiering Fluss data to Delta Lake`.
3. Discussion happens on the list, not in wiki comments.
   When the design changes, update the page and announce it in the thread.

## 4. Vote

1. When the discussion settles, start a `[VOTE] FIP-60 ...` thread
   on dev@fluss.apache.org.
2. It needs at least 3 binding +1 (committers / PMC) and no binding veto.
3. Report the result in the vote thread.
4. Update the wiki page label: discussion -> accepted (or rejected).
   After release, set it to released.
