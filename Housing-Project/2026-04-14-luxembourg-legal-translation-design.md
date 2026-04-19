# Luxembourg Legal Translation System — Design Spec

**Date:** 2026-04-14
**Status:** Approved

## What This Is

A Claude Code skill (`/translate-legal`) that translates English-drafted Luxembourg legal correspondence into Luxembourg-standard French. The output must read as if written by an informed, articulate private citizen in Luxembourg — not a lawyer, not a translation.

The user (Usman) drafts legal letters in English because he does not speak French. The English drafts already contain Luxembourg legal concepts (Code Civil articles, Luxembourg institutions, procedural terms) expressed in English. The system renders these in their natural French legal form.

## What This Is Not

- Not a drafting tool — the user provides a finished English letter
- Not a legal advice tool — it does not evaluate or modify the user's legal arguments
- Not a general-purpose translator — it is specifically tuned for Luxembourg legal correspondence
- Not a lawyer simulator — the output follows non-lawyer (private citizen) conventions

## Use Cases

1. **Mise en demeure (formal demand letters)** — e.g., demanding an architect deliver contracted services
2. **Requete to Justice de Paix** — small claims filings up to 15,000 EUR
3. **Disciplinary complaints** — e.g., to the Ordre des Architectes et des Ingenieurs-Conseils (OAI)
4. **Administrative complaints** — e.g., to consumer protection bodies
5. **General formal legal correspondence** — any formal letter requiring Luxembourg legal register

Expected frequency: approximately 4 times per year. Quality is the priority; cost and convenience are secondary.

## Pipeline Architecture

```
/translate-legal invoked
        |
  Stage 1: INTAKE (interactive)
  - Who is this addressed to?
  - What type of document?
  - (If unknown recipient: web research on conventions)
        |
  Stage 2: TRANSLATE (Opus)
  - Full English -> French in one pass
  - Luxembourg legal system prompt + glossary + formulas
        |
  Stage 3: EVALUATE (Opus, fresh reviewer role)
  - MQM rubric: accuracy, terminology, legal-specific,
    fluency, register, completeness
  - Structured JSON error report
        |
    Errors found? --yes--> Stage 4: REFINE (Opus)
        |                     Fix flagged errors
       no                     Loop back to Stage 3
        |                     (max 3 loops total)
        |
  Stage 5: BACK-TRANSLATE + VERIFY (Opus)
  - FR -> EN with different prompt (general translator, not legal)
  - Compare to original: meaning, content, force/tone, legal precision
        |
    Drift found? --yes--> Back to Stage 4: REFINE
        |                  (same loop counter)
       no
        |
  Stage 6: RULES COMPLIANCE (Opus, fresh proofreader role)
  - Luxembourg authenticity check
  - Article numbers, institutions, formulas, false friends, conventions
        |
    Violations? --yes--> Back to Stage 4: REFINE
        |                (same loop counter)
       no
        |
  Stage 7: OUTPUT
  1. Final French translation
  2. Back-translation (English rendering for user verification)
  3. Confidence summary (1-5 score)
  4. Human review flags (if any)
```

### Key Design Properties

- **Single loop counter across all stages** — max 3 refinement passes total, not 3 per stage. Prevents runaway token usage while giving enough room to converge.
- **Three distinct Opus roles** — translator, reviewer, proofreader. Each has a different system prompt and perspective to avoid self-review blind spots.
- **The back-translation is always included in output** — this is the user's verification tool since he does not read French.
- **Human review flags are conservative** — the system flags when uncertain, not just when errors are found. Better to over-flag than to miss something in a legal document.
- **If max loops reached without convergence** — the system stops and reports the specific unresolved issues rather than producing a potentially flawed translation silently.

## Stage Details

### Stage 1: Intake

Two questions only:

**Q1: "Who is this letter addressed to?"**
- Justice de Paix (Luxembourg, Esch-sur-Alzette, or Diekirch)
- Opposing party directly (mise en demeure / formal demand)
- Ordre des Architectes et des Ingenieurs-Conseils (OAI)
- Consumer protection body (Ministere de la Protection des consommateurs)
- Other (free text — system researches conventions before proceeding)

**Q2: "What type of document is this?"**
- Mise en demeure (formal demand letter)
- Requete to Justice de Paix
- Disciplinary complaint
- General formal correspondence
- Other (free text — system researches before proceeding)

For unknown recipients or document types, the system does web research on the specific body's filing conventions, required format, and terminology. It shows the user what it found and asks for confirmation before translating.

No other questions. The user has already written the letter.

### Stage 2: Translate

Single-pass translation using Opus with a Luxembourg legal system prompt.

**Translator identity:** Legal translation specialist for Luxembourg. Produces French text that reads as if written by an informed, articulate private citizen in Luxembourg.

**System prompt includes:**
- The full Luxembourg legal glossary (false friends, terminology, Code Civil mappings)
- Formula templates appropriate for the document type identified at intake
- Instructions to recognize Luxembourg legal concepts already in the English text and render them in native French form
- Register guidance: vouvoiement, impersonal constructions, subjunctive, no contractions
- Non-lawyer conventions: no professional titles, private citizen format
- Instruction to preserve the author's tone and intent (firm stays firm, measured stays measured)
- Instruction to flag (not silently approximate) legal concepts that don't map cleanly

### Stage 3: Evaluate

Fresh prompt with distinct role: "Senior Luxembourg legal translation reviewer."

Receives: English original + French translation.

Produces: structured JSON error report.

**Six evaluation dimensions:**

1. **Accuracy** — mistranslations, omissions, additions, untranslated text. Sub-categories: legal false friends, jurisdictional incongruity.
2. **Terminology** — wrong legal terms, inconsistent usage, non-Luxembourg-standard terms.
3. **Legal-specific** — negation scope changes, modality shifts (shall/may/must), ambiguity introduction, cross-reference errors, wrong Code Civil article numbers.
4. **Fluency** — grammar, spelling, punctuation (elevated to major if punctuation changes clause structure).
5. **Register** — formality level, vouvoiement consistency, impersonal constructions.
6. **Completeness** — all operative clauses present, all demands/deadlines preserved, all legal citations intact.

**Severity levels:**
- Critical (weight 25): legal false friends, omitted operative clauses, negation changes, modality shifts, wrong article numbers, hallucinated content
- Major (weight 5): general mistranslation, wrong terminology, register errors, ambiguity, inconsistent terms
- Minor (weight 1): grammar, spelling, style, formatting

### Stage 4: Refine

Triggered when Stage 3, 5, or 6 finds critical or major errors.

- Receives the translation plus the structured error feedback
- Addresses each flagged error individually
- Revised translation always returns to Stage 3 (Evaluate) for re-evaluation, regardless of which stage triggered the refinement. After passing Stage 3, the pipeline resumes from Stage 5 (Back-translate).
- Minor errors fixed in-line without triggering re-loop
- All refinement passes share a single counter — max 3 total across the entire pipeline

### Stage 5: Back-Translate and Verify

**Back-translation prompt uses a deliberately different role** — a general-purpose translator, not a legal specialist. This avoids the blind spot where the same legal framing that produced the translation would also produce a matching back-translation.

Back-translator instruction: "Translate this French text into plain, clear English. Translate what the text actually says, not what you think it was supposed to say. Do not interpret legal formulas — render them literally."

**Verification compares three texts side by side:**
1. Original English letter
2. French translation
3. Back-translation

**Five drift categories:**
- Meaning change — something says something different
- Missing content — a point, demand, deadline, or condition dropped
- Added content — something inserted that wasn't in the original
- Force/tone change — firm demand became polite request, or vice versa
- Legal precision change — specific obligation became vague, conditional became absolute

Flagged segments shown with both versions side by side and an explanation of the drift.

### Stage 6: Rules Compliance Check

Fresh prompt with distinct role: "Luxembourg legal proofreader checking whether this document could have been written by an informed private citizen in Luxembourg."

**Nine specific checks:**

1. Code Civil article numbers — any post-2016 French numbering?
2. Institutional names — any France-specific courts or institutions?
3. Procedural references — NCPC cited correctly?
4. Document type formulas — required formulas present and correct?
5. False friends — final sweep against watchlist
6. Non-lawyer conventions — no professional titles, private citizen format
7. Register consistency — vouvoiement throughout, correct closing formula
8. Legal references — Luxembourg laws cited, not French equivalents
9. Formatting — "SOUS TOUTES RESERVES" header, delivery method stated, proper sender/recipient blocks

Output: pass or fail with specific violations listed.

### Stage 7: Output

Final deliverable to the user:

1. **Final French translation** — ready to send
2. **Back-translation** — English rendering of what the French actually says
3. **Confidence summary** — score 1-5, number of review loops taken, any remaining concerns
4. **Human review flags** — specific passages where the system recommends a Luxembourg lawyer review before sending, with explanation

## Luxembourg Legal Ruleset

### Terminology Rules

- Pre-2016 Code Civil article numbers exclusively:
  - Art. 1134 (binding force of contracts) — NOT France's 1103
  - Art. 1147 (contractual liability) — NOT France's 1231-1
  - Art. 1382 (tort/general fault) — NOT France's 1240
  - Art. 1383 (tort/negligence) — NOT France's 1241
  - Art. 1153 (legal interest on monetary debts) — NOT France's 1231-6
  - Art. 1184 (judicial resolution for non-performance) — NOT France's 1224-1230
  - Art. 1315 (burden of proof) — NOT France's 1353
  - Art. 1139 (mise en demeure form)
  - Art. 1146 (requirement of mise en demeure for damages)
  - Art. 1148 (force majeure)
- NCPC (Nouveau Code de procedure civile) — never the French CPC
- "Tribunal d'arrondissement siegeant en matiere commerciale" — never "tribunal de commerce" (Luxembourg has no separate commercial court)
- Justice de Paix competence: 15,000 EUR (raised from 10,000 by Loi du 15 juillet 2021). Final instance (no appeal) up to 2,000 EUR.
- Ordonnance de paiement opposition (contredit): 30 days (changed from 15 by Loi du 15 juillet 2021). Contredit is the sole remedy (opposition suppressed in 2021).
- Late payment interest: cite "loi modifiee du 18 avril 2004". Consumer rate 2026: 3.75%. Commercial: ECB reference rate + 8 percentage points.
- Company law: cite "loi modifiee du 10 aout 1915 concernant les societes commerciales"
- Note: Luxembourg Code Civil reform is under discussion (steering committee formed 2022, headed by Prof. David Hiez) but NO renumbering has been enacted as of April 2026.

### False Friends Watchlist

| English | WRONG French | CORRECT French |
|---------|-------------|---------------|
| evidence | evidence | preuve |
| delay | delai | retard |
| crime (general) | crime | infraction / delit |
| demand (to require) | demander | exiger / requerir |
| injury | injure | blessure / prejudice |
| liable | liable | responsable |
| sentence (legal) | sentence | peine / condamnation |
| jurisdiction (power) | juridiction | competence |
| actual | actuel | reel / effectif |
| eventually | eventuellement | finalement |
| prejudice (bias) | prejudice | parti pris |

Note: "delai" IS correct when meaning "deadline/time limit" — it is only a false friend when translating the English word "delay" (meaning lateness/retard).

### Required Formulas by Document Type

**Mise en demeure:**
- "La presente vaut mise en demeure de payer/de..."
- Specific deadline: "dans un delai de xx jours a reception de la presente"
- Threat of legal action: "proceder au recouvrement par voie de justice" or "des procedures judiciaires pourront etre intentees contre vous sans autre avis ni delai"
- Interest reference: "le present courrier fait courir les interets legaux et conventionnels"
- Cross-payment courtesy (at bottom, in italic): "Nous vous prions de ne pas tenir compte de la presente mise en demeure dans le cas ou cette lettre croiserait votre paiement."
- Closing: "Veuillez croire/agreer, Madame, Monsieur, en l'assurance de notre/ma consideration distinguee."

**Requete to Justice de Paix:**
- Party identification (names, addresses, professions, domiciles)
- "L'objet et l'expose sommaire des moyens"
- Claim amount and legal basis
- Filed in triplicate

**Court conclusions:**
- "Plaise au Tribunal"
- "Il est expose ce qui suit"
- "Par ces motifs" (introduces the prayer for relief)

**General formal correspondence:**
- Salutation: "Madame, Monsieur," (unknown) / "Maitre," (to a lawyer) / "Monsieur le Juge de Paix," (to a judge)
- Closing: "Veuillez agreer/croire... l'assurance de notre/ma consideration distinguee."

**Disciplinary/administrative complaints:**
- System researches the specific body's conventions at intake time

### Register and Style Rules

- Vouvoiement exclusively
- Third-person/impersonal constructions preferred ("il est porte a votre connaissance" over "je vous informe")
- Subjunctive mood used correctly in formal dependent clauses
- No contractions or informal language
- "SOUS TOUTES RESERVES" at top of legal correspondence
- "Sous reserve de tous droits, moyens et actions" as a reservation of rights

### Non-Lawyer Letter Conventions

- Letter written by a private individual, not a lawyer
- Signature block: name, address, phone number — no professional title
- No law firm letterhead or bar registration references
- For businesses: denomination sociale, siege social, R.C.S. number, capital social (if Sarl)
- Delivery method stated: "Lettre recommandee avec accuse de reception"
- No "avocat a la Cour" or any bar-related references anywhere in the document

## MQM Error Taxonomy (Legal Profile)

```
1. ACCURACY
   1.1 Mistranslation
       1.1.1 Legal false friend (CRITICAL)
       1.1.2 Jurisdictional incongruity (CRITICAL)
       1.1.3 General mistranslation (MAJOR)
       1.1.4 Hallucination (CRITICAL)
   1.2 Omission
       1.2.1 Operative clause omission (CRITICAL)
       1.2.2 Modifier/qualifier omission (MAJOR)
       1.2.3 Minor element omission (MINOR)
   1.3 Addition
       1.3.1 Obligation/condition addition (CRITICAL)
       1.3.2 Redundant addition (MINOR)
   1.4 Untranslated text (MAJOR)

2. TERMINOLOGY
   2.1 Inappropriate for legal context (MAJOR)
   2.2 Inconsistent use within document (MAJOR)
   2.3 Non-standard for Luxembourg legal system (MAJOR)

3. FLUENCY
   3.1 Grammar (MINOR)
   3.2 Spelling (MINOR)
   3.3 Punctuation (MINOR; MAJOR if changes clause structure)
   3.4 Register (MAJOR)
   3.5 Inconsistency (MINOR)

4. STYLE
   4.1 Awkward/non-idiomatic French (MINOR)
   4.2 Literalness / calque from English (MINOR; MAJOR if misleading)

5. LEGAL-SPECIFIC
   5.1 Negation scope change (CRITICAL)
   5.2 Ambiguity introduction (MAJOR)
   5.3 Modality shift — shall/may/must (CRITICAL)
   5.4 Reference/cross-reference error (MAJOR)
   5.5 Wrong Code Civil article number (CRITICAL)

6. COMPLETENESS
   6.1 Demand/deadline not preserved (CRITICAL)
   6.2 Legal citation not preserved (MAJOR)
   6.3 Condition/qualification not preserved (MAJOR)
```

**Scoring:**
- Minor = 1, Major = 5, Critical = 25
- Pass threshold: zero critical errors, zero major errors
- Minor errors fixed in-line without re-looping

## Files to Create

| File | Location | Purpose |
|------|----------|---------|
| `translate-legal` skill | `.claude/skills/translate-legal/` | Pipeline logic, stage prompts, flow control |
| `glossary-luxembourg-legal.md` | `.claude/skills/translate-legal/references/` | False friends, terminology, Code Civil mappings, formulas |
| `mqm-legal-rubric.md` | `.claude/skills/translate-legal/references/` | Error taxonomy, severity weights, evaluation prompt template |

## Key Luxembourg Legal References

- Code Civil (pre-2016 Napoleonic numbering): [Legilux consolidated](https://legilux.public.lu/eli/etat/leg/code/civil/20250420)
- NCPC: [Legilux consolidated](https://legilux.public.lu/eli/etat/leg/code/procedure_civile/20231101)
- Loi du 15 juillet 2021 (justice reform): [Government communique](https://gouvernement.lu/fr/actualites/toutes_actualites/communiques/2021/09-septembre/13-justice-nouvelles-dispositions.html)
- Loi modifiee du 18 avril 2004 (late payment): [Legilux](https://legilux.public.lu/eli/etat/leg/loi/2004/04/18/n8/jo)
- Loi modifiee du 10 aout 1915 (companies): [Legilux](https://legilux.public.lu/eli/etat/leg/loi/1915/08/10/n1/jo)
- Mise en demeure template: [Guichet.lu](https://guichet.public.lu/dam-assets/catalogue-formulaires/creances/form-mise-demeure/modele-mise-demeure_FR.doc)
- Justice de Paix procedures: [Justice.public.lu](https://justice.public.lu/fr/organisation-justice/juridictions-judiciaires/justices-paix/tribunal-paix.html)
- Interest rates: [Ministere de la Justice](https://mj.gouvernement.lu/fr/service-citoyens/taux-interet-legal.html)
- Language regime: [Loi du 24 fevrier 1984](https://legilux.public.lu/eli/etat/leg/loi/1984/02/24/n1/jo) — French is the authoritative language for all Luxembourg legal documents
