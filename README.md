# XYGO validation with randomized block order

Web questionnaire for the XYGO wind rose (gender identity and sexual orientation on two continuous axes, -3 to +3), used in a test-retest study in which the order of the two blocks is randomized. Developed by Andrea Sansone within ENDOSEX, University of Rome Tor Vergata (principal investigator: Prof. Emmanuele A. Jannini).

## Study design

Each participant completes the questionnaire twice, at least 10 days apart. The questionnaire has two blocks: the XYGO scales (gender and orientation roses) and the self-reported labels (gender identity and sexual orientation chosen from a fixed list). The order of the blocks is assigned at T1 and kept identical at T2, so that any order effect does not contaminate the test-retest estimate. The order is stored with every row.

This repository is a separate study from the fixed-order validation (86 T1 compilations). It uses its own spreadsheet, its own Apps Script deployment and its own web address; the data of the two studies must not be merged.

## Randomization rule

The order is derived from the token, not from a server-side counter. The token has 6 characters drawn from a 32-symbol alphabet (ABCDEFGHJKLMNPQRSTUVWXYZ23456789; O, I, 0 and 1 are excluded to avoid transcription errors). The last character decides the arm:

- first 16 symbols (ABCDEFGHJKLMNPQR): XYGO scales first, labels second (`XYGO_first`);
- last 16 symbols (STUVWXYZ23456789): labels first, XYGO scales second (`LABELS_first`).

Because the alphabet has 32 symbols, the split is exactly 50:50 for a uniform draw. Tokens are generated with `crypto.getRandomValues` at the start of T1 (byte modulo 32 is unbiased, since 256 is a multiple of 32). A simulation of 200,000 tokens gave 49.92% `XYGO_first`, with no invalid or duplicate tokens in 20,000 draws.

At T2 the participant types the token received at T1; the app rebuilds the order from the last character, so no information needs to be stored on the participant's device. A token that is not 6 valid characters blocks the questionnaire.

## What the participant sees

The page sequence is: introduction, biological sex and age, then the two blocks in the assigned order, then the thank-you page. The submit button appears on the last page of the second block. The XYGO notation (for example `X; G+1; O-2`) is shown only on the final page, so that it cannot anchor the answers given afterwards. Values shown on screen are unsigned; the signs are kept in the payload and in the notation.

## Data columns

Sheet `XYGO_Randomized` (no e-mail addresses):

| Column | Content |
|---|---|
| id | unique identifier of the row |
| timestamp | ISO 8601 time of submission |
| timepoint | T1 or T2 |
| token | 6-character key linking T1 and T2 |
| order | XYGO_first or LABELS_first |
| lang | it or en |
| sex | X (assigned female at birth) or Y (assigned male at birth) |
| age | years |
| genderLabel, orientLabel | self-reported labels |
| genderScores, orientScores | signed integers, comma-separated (text, to avoid date conversion in Italian-locale Sheets) |
| fluidGender, fluidOrient | TRUE when the scores are the ends of a continuous range |
| notation | XYGO notation computed by the app |
| backNav | number of times the participant went back from one block to the other |
| tBlockStart | time at which the first block was shown |
| tBlockMid | time at which the first block was left |
| version | webapp version (rand-v1) |

Sheet `XYGO_Reminders` holds token, e-mail address and sending status. It is separate from the data so that the psychometric rows can never be linked to an address; rows are cleared after the reminders have been sent.

## Setup

1. Create a new Google Spreadsheet.
2. Open Extensions > Apps Script, paste `gas/Code.gs` and save.
3. Set `STUDY_URL` at the top of `Code.gs` to the GitHub Pages address of this repository.
4. Deploy > New deployment > Web app, execute as "Me", access "Anyone". Copy the web app URL.
5. Run `testReminder()` once from the editor (after putting your own address in it) and accept the authorization prompts for Sheets and Mail. Delete the test rows (token TESTAA) afterwards.
6. Run `createDailyTrigger()` once; it schedules the daily check that sends the reminders.
7. In `index.html`, replace the value of `WEBHOOK` with the web app URL. While the placeholder is in place the app refuses to submit and shows an explicit error.
8. Push the repository and enable GitHub Pages (Settings > Pages > branch `main`, folder root).

Every later change to `Code.gs` requires a new deployment and a new `WEBHOOK` value; an updated old deployment keeps serving the previous code.

## E-mails

After the participant leaves an address, a thank-you message is sent immediately, with the token and the link. A first reminder is sent 10 days after T1; a second one 4 days later (day 14) only if no T2 with that token has arrived. Until `STUDY_URL` is set, the daily check logs a warning and sends nothing.

## Analysis notes

- Merge T1 and T2 by token: `inner_join(t1, t2, by = "token", suffix = c("_t1", "_t2"))`.
- Check that `order` is identical at T1 and T2 for every token and that it agrees with the last character of the token. A mismatch means a mistyped or reused token.
- Rows with `backNav` > 0 can be analyzed in a sensitivity analysis excluding them.
- `tBlockMid - tBlockStart` gives the time spent on the first block; the time on the second block is `timestamp - tBlockMid`.
- With 100 to 150 compilations at T1 and a T2 completion of roughly 40 to 50%, each arm has about 50 to 75 participants at T1 and fewer at T2. The order effect is therefore best treated as exploratory; the suggested primary outcome is the concordance between label and score at T1, and the secondary outcome the difference in ICC between arms. Both should be pre-registered.
- The consent form and the ethics documentation should state that the order of the two blocks varies between participants.

```r
df <- googlesheets4::read_sheet("<spreadsheet url>", sheet = "XYGO_Randomized",
                                col_types = "c")
t1 <- dplyr::filter(df, timepoint == "T1")
t2 <- dplyr::filter(df, timepoint == "T2")
merged <- dplyr::inner_join(t1, t2, by = "token", suffix = c("_t1", "_t2"))
```

## Repository structure

```
index.html     webapp (single file, React via CDN)
gas/Code.gs    Google Apps Script webhook and reminder logic
README.md      this file
```
