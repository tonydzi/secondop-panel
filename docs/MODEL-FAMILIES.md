# How to choose 3–4 model families

Quorum counts **families**, so the only question that matters is: *would these two reviewers fail the same way?*

## The rule

**One rail per lab.** Two models from the same lab share training data, RLHF taste and blind spots. They agree with each other far more than they agree with an outsider — and agreement between correlated reviewers reads exactly like confirmation while carrying almost no information.

A "family" is the lab, not the model: `google/gemini-2.5-flash-lite` and a local `gemini` CLI are one family. In `panel.json`, `family` is yours to declare — declare it honestly, because the whole guarantee rests on that string.

## The floor: three, and why not two

Two rails tell you *that* they disagree. Three tell you *who is the outlier*. Our own runs kept producing three different lists from three rails with a small overlap — each rail owned a class the others missed. Below three you are not running a panel, you are running a coin flip with extra steps.

Four is the practical ceiling for a ritual you run on every change: past that, latency and your own reading time cost more than the marginal finding.

## What to select for

1. **Will it admit it has no data?** The single best predictor of a reviewer that will not fabricate. Test it in one command:
   ```bash
   python secondop_panel.py t3 --paths-only --files src/whatever.py --rails <rail>
   ```
   A model that answers "I cannot see the code" passes. A model that returns five confident findings about a file it never saw fails, and it will fail the same way when your prompt is merely *incomplete* rather than empty. Re-run this when you swap a model — it is a property of the model, not of your prompt.
2. **Different failure modes, not different logos.** Mix a strong-reasoning model with a fast code-specialised one: the first catches design and race conditions, the second catches API misuse and dead branches. Two "flagship generalists" from two labs are still more useful than two models from one lab, but they overlap more than you would like.
3. **Different plumbing.** If all four rails go through one gateway, that gateway is your single point of failure — its outage looks exactly like "the code is fine, nobody objected". Keep at least one rail on a different path (a local CLI, a direct vendor endpoint). Then `doctor` can tell you which door is closed instead of the panel silently shrinking.
4. **Cost you will not flinch at.** A panel you skip because it is expensive is a panel you do not have. Small/fast tier models are enough for `t3` breaking: a typical panel (≈2k-token prompt, ≈800-token answers, 4 models) costs well under a cent on a pay-per-token gateway. Spend the expensive reasoning budget on `t1` and `t2`, where a wrong plan is the costly mistake.
5. **A vendor you can drop.** Availability changes: quotas burn out, frontends break, endpoints move. Because rails are data in a JSON file and the quorum rule never learns vendor names, replacing one is one edit and no code change. Design for that from day one.

## A concrete starting panel

| rail | family | why it is there |
|---|---|---|
| a strong reasoning model | lab A | design flaws, races, "what happens on the second call" |
| a code-specialised model | lab B | API misuse, dead branches, wrong types |
| a fast cheap generalist | lab C | breadth; catches the obvious thing the deep models talked themselves out of |
| your own assistant's vendor — **as the fourth, never the first** | lab D | it already knows the codebase; it also wrote it, so it is the most compromised reviewer on the list |

That last row is the whole thesis in one line: the model that wrote the code is allowed to speak, but it is never allowed to be the quorum.

## What NOT to do

* Do not fill the panel with four models from one lab because they are convenient. That is one opinion with four invoices.
* Do not let a rail fall back to another vendor when it fails — a verdict from A under C's name makes the independence fake. Skip, and let the exit code be 3.
* Do not add a fifth and sixth rail hoping for consensus. Panels are for finding candidates, not for voting truth into existence: one rail naming one real defect beats four rails agreeing on nothing.
