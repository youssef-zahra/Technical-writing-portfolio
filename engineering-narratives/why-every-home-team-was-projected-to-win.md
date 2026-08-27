
# Why every home team was projected to win (and what that bug taught me)

I built a project that rates NFL teams using an Elo system, similar to what FiveThirtyEight and other models use, with the goal of predicting which teams were likely to make the playoffs based on projected win totals. Early on, I ran a batch of weekly predictions and noticed something wrong immediately. Every single home team was projected to win. Not most. Every one.

## The first clue

Home field advantage is real and my model accounted for it, adding a small boost to the home team's win probability to reflect it. That boost is supposed to be a nudge, not a guarantee. When I looked at the output, every game showed the home team favored by almost exactly the same margin, regardless of who was actually playing. A last place team hosting a projected division winner was still favored to win.

That pattern only makes sense if every team had the same underlying rating. If ratings were working correctly, a strong team should be favored on the road against a weak team, home field boost or not. The fact that home field advantage alone was deciding every single game meant the ratings feeding into the model were not doing anything at all.

## Tracing it back

I coded the pipeline to pull team ratings from one data source and the weekly schedule and matchup data from another. The ratings source used full team names, Seattle Seahawks, Kansas City Chiefs, and so on. The schedule source used shortened abbreviations instead, SEA, KC.

When the model tried to look up a team's rating using the abbreviation, it could not find a match against the full name in the ratings table. Instead of failing loudly, the lookup silently fell back to a default rating for every team that did not match. Since almost every team was affected the same way, they all ended up with the same default value, which meant the only thing left to differentiate any two teams was the home field boost. That is exactly the pattern I was seeing.

## The fix

I added a mapping layer in the code that translates each abbreviation to its corresponding full team name before the rating lookup runs, so both data sources are referring to the same team regardless of which naming convention they use.

```python
TEAM_NAME_MAP = {
    "SEA": "Seattle Seahawks",
    "KC": "Kansas City Chiefs",
    # ... full mapping for every team in the league
}

def get_team_rating(team_code, ratings_table):
    full_name = TEAM_NAME_MAP.get(team_code, team_code)
    if full_name not in ratings_table:
        raise ValueError(f"No rating found for {full_name} ({team_code})")
    return ratings_table[full_name]
```
I also added a validation check that runs before any predictions are generated. It confirms every team in the current week's matchups has a real rating on file, and raises an error immediately if anything is missing, rather than silently defaulting the way the original code did.

## What changed

Once the mapping was in place, ratings varied properly across teams again, and predictions started reflecting actual team strength rather than just home field advantage. Just as important, the validation check meant that if this class of bug ever came back, in a new season with a renamed team, or a new data source with a different naming convention, the pipeline would fail loudly and immediately instead of quietly producing wrong predictions that looked plausible.

## Why this matters beyond sports

The dangerous version of this bug was not that it broke the model. It was that it did not break anything visibly. No errors, no crashes, just quietly wrong output that still looked like a real prediction. Any system that joins data across two sources with different naming or ID conventions is exposed to this same failure mode. The fix here was small, a mapping table and a validation check, but finding it depended on noticing a pattern in the output that should not have been possible if the model were actually working.

