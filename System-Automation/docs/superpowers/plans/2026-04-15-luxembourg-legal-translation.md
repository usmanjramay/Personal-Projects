# Luxembourg Legal Translation Skill — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a Claude Code skill (`/translate-legal`) that translates English-drafted Luxembourg legal correspondence into Luxembourg-standard French through a multi-stage pipeline with structured quality review loops.

**Architecture:** A single Claude Code skill file (`SKILL.md`) drives a 7-stage pipeline (intake, translate, evaluate, refine, back-translate, rules compliance, output) within one conversation. Two reference files provide the Luxembourg legal glossary and the MQM evaluation rubric. All stages use Opus with distinct role prompts to avoid self-review blind spots.

**Tech Stack:** Claude Code skill system (SKILL.md + references/), no external APIs or infrastructure.

**Spec:** `docs/superpowers/specs/2026-04-14-luxembourg-legal-translation-design.md`

---

## File Map

| File | Responsibility |
|------|---------------|
| `.claude/skills/translate-legal/SKILL.md` | Skill definition: frontmatter, intake questions, pipeline stage instructions, stage prompt templates, loop control logic, output format |
| `.claude/skills/translate-legal/references/glossary-luxembourg-legal.md` | Luxembourg-specific legal terminology: false friends watchlist, Code Civil article mappings (pre-2016 vs France post-2016), required formulas by document type, register/style rules, non-lawyer conventions |
| `.claude/skills/translate-legal/references/mqm-legal-rubric.md` | MQM error taxonomy adapted for legal, severity weights, structured evaluation prompt template, back-translation verification prompt, rules compliance checklist |

---

## Task 1: Create the Luxembourg Legal Glossary Reference

This is the foundational reference file. The skill and rubric both depend on it.

**Files:**
- Create: `.claude/skills/translate-legal/references/glossary-luxembourg-legal.md`

- [ ] **Step 1: Create the references directory**

```bash
mkdir -p /Users/usmanramay/.claude/skills/translate-legal/references
```

- [ ] **Step 2: Write the glossary file**

Create `.claude/skills/translate-legal/references/glossary-luxembourg-legal.md` with the following complete content:

```markdown
# Luxembourg Legal Glossary

Reference file for the translate-legal skill. Contains terminology rules, false friends, required formulas, and conventions specific to Luxembourg legal correspondence.

---

## 1. Code Civil Article Mappings

Luxembourg retains the original Napoleonic Code Civil numbering. France renumbered in 2016 (Ordonnance 2016-131). Using French post-2016 numbers in a Luxembourg document is an immediate authenticity failure.

| Concept | Luxembourg (CORRECT) | France post-2016 (WRONG) |
|---------|---------------------|-------------------------|
| Binding force of contracts | Art. 1134 | Art. 1103 |
| Implied obligations | Art. 1135 | Art. 1194 |
| Mise en demeure form | Art. 1139 | Art. 1344 |
| Requirement of mise en demeure for damages | Art. 1146 | Art. 1231 |
| Contractual liability | Art. 1147 | Art. 1231-1 |
| Force majeure | Art. 1148 | Art. 1218 |
| Legal interest on monetary debts | Art. 1153 | Art. 1231-6 |
| Judicial resolution for non-performance | Art. 1184 | Art. 1224-1230 |
| Burden of proof | Art. 1315 | Art. 1353 |
| Tort — general fault | Art. 1382 | Art. 1240 |
| Tort — negligence | Art. 1383 | Art. 1241 |

Note: Luxembourg Code Civil reform is under discussion (steering committee formed 2022, Prof. David Hiez) but NO renumbering enacted as of April 2026.

---

## 2. False Friends Watchlist

These English-French cognates have different meanings in legal contexts. The WRONG column shows what an AI translator might naively produce. The CORRECT column shows what a Luxembourg legal document must use.

| English Term | WRONG French | CORRECT French | Why It Matters |
|-------------|-------------|---------------|----------------|
| evidence | évidence | preuve | "Évidence" means obviousness/self-evidence, not proof |
| delay (lateness) | délai | retard | "Délai" means deadline/time limit — correct for that meaning, wrong for "lateness" |
| crime (general offense) | crime | infraction / délit | French "crime" = only felony-level offenses (cour d'assises) |
| demand (to require) | demander | exiger / requérir | "Demander" = merely to ask/request, not to legally require |
| injury | injure | blessure / préjudice | "Injure" means insult/defamation |
| liable | liable | responsable | "Liable" is not a French legal term |
| sentence (legal judgment) | sentence | peine / condamnation | French "sentence" = maxim/saying |
| jurisdiction (power) | juridiction | compétence | "Juridiction" = the court body itself, "compétence" = the power/authority |
| actual | actuel | réel / effectif | "Actuel" means current/present |
| eventually | éventuellement | finalement | "Éventuellement" means possibly/perhaps |
| prejudice (bias) | préjudice | parti pris / préjugé | French "préjudice" = damage/harm |

Note: "délai" IS correct when the English means "deadline" or "time limit." It is only a false friend when translating "delay" meaning lateness/tardiness.

---

## 3. Luxembourg-Specific Institutional Terminology

| CORRECT (Luxembourg) | WRONG (France/other) | Why |
|----------------------|---------------------|-----|
| Tribunal d'arrondissement siégeant en matière commerciale | Tribunal de commerce | Luxembourg has no separate commercial court; commercial chambers sit within the district court |
| Nouveau Code de procédure civile (NCPC) | Code de procédure civile (CPC) | Luxembourg has its own procedural code, distinct from France's |
| Avocat à la Cour | Avocat (unqualified) | Specific title for List I lawyers who completed the stage judiciaire; List II lawyers are simply "Avocat" |
| Cour supérieure de Justice | Cour de cassation (standalone) | Luxembourg's supreme court comprises both the Cour d'appel and Cour de cassation |
| Loi modifiée du 10 août 1915 | Code de commerce (for company law) | Luxembourg's foundational company law statute |
| R.C.S. (Registre de Commerce et des Sociétés) | RCS (France) | Same abbreviation but different registry (Luxembourg Business Registers / LBR) |
| Mémorial (A, B, C) | Journal Officiel | Luxembourg's official gazette |
| Sàrl / Sàrl-S | SARL | Luxembourg uses accent on "à"; Sàrl-S is the simplified 1 EUR capital variant |
| Contredit | Opposition | Sole remedy against ordonnance de paiement since 2021 reform |

---

## 4. Key Legal Parameters (2026)

| Parameter | Value | Source |
|-----------|-------|--------|
| Justice de Paix competence threshold | 15,000 EUR | Loi du 15 juillet 2021 |
| Justice de Paix final instance (no appeal) | up to 2,000 EUR | Loi du 15 juillet 2021 |
| Ordonnance de paiement contredit period | 30 days | Loi du 15 juillet 2021 (raised from 15) |
| Consumer late payment interest rate (2026) | 3.75% | Règlement grand-ducal under loi du 18 avril 2004 |
| Commercial late payment interest rate | ECB reference rate + 8 percentage points | Loi du 29 mars 2013 (modifying loi du 18 avril 2004) |
| Flat B2B recovery fee | 40 EUR | Loi du 18 avril 2004 |
| Late payment interest formula | (Amount × Rate × Days late) / (365 × 100) | Standard |

---

## 5. Required Formulas by Document Type

### Mise en Demeure

**Opening/header:**
- "SOUS TOUTES RÉSERVES" (at top)
- "Objet : Mise en demeure de paiement" (or "...de [specific obligation]")
- "Lettre recommandée avec accusé de réception"

**Performative declaration (REQUIRED — this is what makes it legally a mise en demeure):**
- "La présente vaut mise en demeure de payer/de [obligation]..."
- OR: "Par la présente, nous vous mettons en demeure de..."

**Deadline (REQUIRED):**
- "...dans un délai de [X] jours à réception de la présente"

**Threat of legal action (REQUIRED):**
- "À défaut d'un paiement à la date du [date], nous serons contraints de procéder, à vos frais, au recouvrement de notre créance par voie de justice."
- OR: "Des procédures judiciaires pourront être intentées contre vous sans autre avis ni délai."

**Interest reference:**
- "Le présent courrier fait courir les intérêts légaux et conventionnels."

**Cross-payment courtesy (at bottom, in italic):**
- "Nous vous prions de ne pas tenir compte de la présente mise en demeure dans le cas où cette lettre croiserait votre paiement."

**Closing:**
- "Veuillez croire, Madame, Monsieur, en l'assurance de notre/ma considération distinguée."
- OR: "Veuillez agréer, Madame, Monsieur, l'expression de nos/mes salutations distinguées."

### Requête to Justice de Paix

**Required elements (Article 101 NCPC, sous peine de nullité):**
- Party identification: names, first names, profession, domicile of all parties
- "L'objet et l'exposé sommaire des moyens" (object and summary statement of grounds)
- Claim amount and legal basis (cite specific Code Civil articles)
- Filed in triplicate (original + copies for each party)

**Structure:**
- "Plaise au Tribunal" (opening)
- "Il est exposé ce qui suit" (introduces the facts)
- Facts section (chronological, with documentary evidence references)
- Legal basis section (cite Luxembourg Code Civil articles)
- "Par ces motifs" (introduces the prayer for relief / dispositif)
- Specific relief requested

### Court Conclusions (Formal Submissions)

- "Plaise au Tribunal"
- "Il est exposé ce qui suit"
- "Par ces motifs" (prayer for relief)

### General Formal Correspondence

**Salutations:**
- Unknown recipient: "Madame, Monsieur,"
- To a lawyer: "Maître," (replaces Monsieur/Madame entirely)
- To a judge: "Monsieur le Juge de Paix," / "Madame la Juge de Paix,"

**Closings:**
- Standard: "Veuillez agréer/croire, Madame, Monsieur, l'assurance de notre/ma considération distinguée."
- To a lawyer: "Je vous prie d'agréer, Maître, l'expression de mes salutations distinguées."
- To a judge: "Veuillez agréer, Monsieur le Juge / Madame la Juge, l'expression de ma considération distinguée."

### Disciplinary/Administrative Complaints

No pre-loaded formulas. The skill researches the specific body's conventions at intake time.

---

## 6. Register and Style Rules

- **Vouvoiement** exclusively — never tutoyement
- **Third-person/impersonal constructions** preferred: "il est porté à votre connaissance" over "je vous informe"
- **Subjunctive mood** used correctly in formal dependent clauses
- **No contractions** or informal language
- **"SOUS TOUTES RÉSERVES"** at top of legal correspondence
- **"Sous réserve de tous droits, moyens et actions"** as reservation of rights
- **Preserve author's tone**: firm stays firm, measured stays measured — do not soften demands or amplify aggression

---

## 7. Non-Lawyer Letter Conventions

The letter is written by a private individual, not a lawyer:
- **Signature block:** name, address, phone number — no professional title
- **No law firm letterhead** or bar registration references
- **No "Avocat à la Cour"** or any bar-related references anywhere
- **For businesses:** dénomination sociale, siège social, R.C.S. number, capital social (if Sàrl)
- **Delivery method stated:** "Lettre recommandée avec accusé de réception"

---

## 8. Key Luxembourg Legal References

- Code Civil: https://legilux.public.lu/eli/etat/leg/code/civil/20250420
- NCPC: https://legilux.public.lu/eli/etat/leg/code/procedure_civile/20231101
- Loi du 15 juillet 2021 (justice reform): https://gouvernement.lu/fr/actualites/toutes_actualites/communiques/2021/09-septembre/13-justice-nouvelles-dispositions.html
- Loi du 18 avril 2004 (late payment): https://legilux.public.lu/eli/etat/leg/loi/2004/04/18/n8/jo
- Loi du 10 août 1915 (companies): https://legilux.public.lu/eli/etat/leg/loi/1915/08/10/n1/jo
- Mise en demeure template: https://guichet.public.lu/dam-assets/catalogue-formulaires/creances/form-mise-demeure/modele-mise-demeure_FR.doc
- Justice de Paix: https://justice.public.lu/fr/organisation-justice/juridictions-judiciaires/justices-paix/tribunal-paix.html
- Interest rates: https://mj.gouvernement.lu/fr/service-citoyens/taux-interet-legal.html
- Language regime (Loi du 24 février 1984): https://legilux.public.lu/eli/etat/leg/loi/1984/02/24/n1/jo
```

- [ ] **Step 3: Verify the file was created correctly**

```bash
ls -la /Users/usmanramay/.claude/skills/translate-legal/references/glossary-luxembourg-legal.md
```

Expected: file exists, non-zero size.

- [ ] **Step 4: Commit**

```bash
git add .claude/skills/translate-legal/references/glossary-luxembourg-legal.md
git commit -m "feat: add Luxembourg legal glossary reference for translate-legal skill"
```

---

## Task 2: Create the MQM Legal Evaluation Rubric Reference

The structured evaluation prompts used by Stages 3, 5, and 6.

**Files:**
- Create: `.claude/skills/translate-legal/references/mqm-legal-rubric.md`

- [ ] **Step 1: Write the MQM rubric file**

Create `.claude/skills/translate-legal/references/mqm-legal-rubric.md` with the following complete content:

```markdown
# MQM Legal Translation Evaluation Rubric

Reference file for the translate-legal skill. Contains the error taxonomy, severity weights, and prompt templates for Stages 3 (Evaluate), 5 (Back-translate & Verify), and 6 (Rules Compliance).

---

## 1. Error Taxonomy (Legal-MQM Profile)

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

---

## 2. Severity Weights

| Severity | Weight | Definition |
|----------|--------|------------|
| Critical | 25 | Inhibits comprehension, could invalidate legal meaning, or poses risk of financial/legal harm |
| Major | 5 | Disrupts accuracy or flow but core meaning still understandable |
| Minor | 1 | Technically incorrect but doesn't hinder comprehension or legal validity |

**Pass threshold:** Zero critical errors AND zero major errors.
Minor errors are fixed in-line without triggering a re-evaluation loop.

---

## 3. Stage 3 Evaluation Prompt Template

Use this prompt template for the evaluation stage. Replace {placeholders} with actual content.

```
You are a senior Luxembourg legal translation reviewer. You have deep expertise in Luxembourg civil law (Code Napoléon, pre-2016 numbering), the NCPC, and Luxembourg legal correspondence conventions.

You are reviewing a translation of a {document_type} addressed to {recipient}.

Your task: identify ALL translation errors, classify each by category and severity, and assess overall quality.

ENGLISH ORIGINAL:
---
{english_original}
---

FRENCH TRANSLATION:
---
{french_translation}
---

EVALUATION DIMENSIONS:

1. ACCURACY — Check for mistranslations, omissions, additions, untranslated text. Pay special attention to:
   - Legal false friends (see glossary: evidence≠évidence, delay≠délai, etc.)
   - Jurisdictional incongruity (common law concepts rendered literally into civil law French)

2. TERMINOLOGY — Check for wrong legal terms, inconsistent usage, non-Luxembourg-standard terms. Specifically:
   - Code Civil article numbers must be pre-2016 Napoleonic (Art. 1134, not 1103; Art. 1382, not 1240)
   - Institutional names must be Luxembourg-specific (tribunal d'arrondissement siégeant en matière commerciale, not tribunal de commerce)
   - NCPC, not CPC

3. LEGAL-SPECIFIC — Check for:
   - Negation scope changes (CRITICAL — can reverse meaning of a clause)
   - Modality shifts: "shall" must become "doit/devra" (obligation), "may" must become "peut/pourra" (permission), "must" must become "doit" (requirement)
   - Ambiguity introduction where the source is clear
   - Cross-reference errors
   - Wrong Code Civil article numbers

4. FLUENCY — Grammar, spelling, punctuation. Elevate punctuation errors to MAJOR if they change clause structure.

5. REGISTER — Vouvoiement consistency, impersonal constructions, formality level appropriate for Luxembourg legal correspondence.

6. COMPLETENESS — All operative clauses present, all demands/deadlines preserved, all legal citations intact, all conditions/qualifications preserved.

SEVERITY LEVELS:
- CRITICAL (weight 25): Legal false friends, omitted operative clauses, negation scope changes, modality shifts, wrong article numbers, hallucinated content, demands/deadlines not preserved
- MAJOR (weight 5): General mistranslation, wrong terminology, register errors, ambiguity, inconsistent terms, untranslated text, legal citations not preserved
- MINOR (weight 1): Grammar, spelling, style, formatting

Respond with a structured evaluation. For each error found, provide:
1. The error span in the French translation
2. The corresponding source text in English
3. The error category (from the taxonomy above, e.g., "1.1.1 Legal false friend")
4. The severity (CRITICAL / MAJOR / MINOR)
5. An explanation of why this is an error
6. A suggested correction

After listing all errors, provide:
- Total error count by severity: Critical: X, Major: Y, Minor: Z
- Overall assessment: PASS (zero critical, zero major) or FAIL
- Confidence score: 1-5 (1 = very low, 5 = high)
- Any segments where you are uncertain and recommend human review, with explanation
```

---

## 4. Stage 4 Refinement Prompt Template

Use this prompt when the evaluation found critical or major errors.

```
You are a Luxembourg legal translation specialist. You are refining a French translation based on specific error feedback from a quality review.

ENGLISH ORIGINAL:
---
{english_original}
---

CURRENT FRENCH TRANSLATION:
---
{french_translation}
---

ERRORS TO FIX:
---
{error_list}
---

Instructions:
1. Address each flagged error individually. For each error:
   - Locate the error span in the translation
   - Apply the suggested correction, or if you have a better correction that is more natural in Luxembourg legal French, use that instead
   - Ensure the fix does not introduce new errors in surrounding text
2. Fix any MINOR errors you notice even if they were not flagged
3. Do not change any part of the translation that was not flagged unless it contains an obvious error
4. Preserve the author's tone and intent throughout
5. Output the complete revised French translation

Provide the full revised translation, then list the specific changes you made and why.
```

---

## 5. Stage 5 Back-Translation Prompt Template

Two prompts: one for the back-translation, one for the verification.

### Back-Translation Prompt

```
Translate the following French text into plain, clear English. Translate what the text actually says, not what you think it was supposed to say. Do not interpret legal formulas — render them literally. Do not add explanations or notes.

FRENCH TEXT:
---
{french_translation}
---

Provide only the English translation, nothing else.
```

### Verification Prompt

```
You are verifying a legal translation by comparing the original English text to a back-translation (French→English) of the French translation.

Your task: identify any segments where the meaning has drifted between the original and the back-translation.

ORIGINAL ENGLISH:
---
{english_original}
---

BACK-TRANSLATION (what the French version actually says in English):
---
{back_translation}
---

Compare section by section. For each segment, check for these five types of drift:

1. MEANING CHANGE — something says something different than the original intended
2. MISSING CONTENT — a point, demand, deadline, or condition was dropped
3. ADDED CONTENT — something appears that wasn't in the original
4. FORCE/TONE CHANGE — a firm demand became a polite request, or vice versa; a factual statement became a threat, or vice versa
5. LEGAL PRECISION CHANGE — a specific obligation became vague, a conditional became absolute, a deadline became approximate, or vice versa

For each drift found, provide:
- The original English segment
- The back-translated segment
- The drift type (from the 5 categories above)
- An explanation of what changed and why it matters

If no drift is found, state: "NO DRIFT DETECTED — back-translation matches original meaning."

At the end, provide:
- Total drift count
- Assessment: PASS (no drift) or FAIL (drift detected)
- Severity of each drift: CRITICAL (changes legal meaning) or MAJOR (changes tone/force but not legal meaning)
```

---

## 6. Stage 6 Rules Compliance Prompt Template

```
You are a Luxembourg legal proofreader. Your task is to check whether this French document could have been written by an informed private citizen in Luxembourg. You are NOT evaluating translation quality — you are checking Luxembourg authenticity.

FRENCH DOCUMENT:
---
{french_translation}
---

DOCUMENT TYPE: {document_type}
RECIPIENT: {recipient}

Run these 9 checks and report pass/fail for each:

1. CODE CIVIL ARTICLE NUMBERS — Are all article numbers pre-2016 Napoleonic numbering? Flag any post-2016 French numbers (e.g., 1240 instead of 1382, 1103 instead of 1134, 1231-1 instead of 1147).

2. INSTITUTIONAL NAMES — Are all institutions Luxembourg-specific? Flag any France-specific references (tribunal de commerce, tribunal judiciaire, cour d'appel de Paris, etc.).

3. PROCEDURAL REFERENCES — Is the NCPC (Nouveau Code de procédure civile) cited correctly? Flag any reference to the French CPC.

4. DOCUMENT TYPE FORMULAS — For the document type "{document_type}", are the required formulas present and correct? (Refer to the glossary for required formulas per document type.)

5. FALSE FRIENDS — Final sweep: does the document contain any term from the false friends watchlist used incorrectly? (évidence for proof, délai for lateness, crime for general offense, etc.)

6. NON-LAWYER CONVENTIONS — Does the document follow private citizen format? Flag any professional legal titles, bar references, or lawyer-specific formatting.

7. REGISTER CONSISTENCY — Is vouvoiement used throughout? Any informal language, contractions, or register breaks?

8. LEGAL REFERENCES — Are all cited laws Luxembourg laws? Flag any French law equivalents (e.g., referencing a French ordonnance or décret instead of Luxembourg equivalents).

9. FORMATTING — Is "SOUS TOUTES RÉSERVES" present? Is the delivery method stated? Are sender/recipient blocks properly formatted?

For each check, report:
- CHECK N: [name] — PASS or FAIL
- If FAIL: the specific violation and where it occurs in the document

Final assessment: PASS (all 9 checks pass) or FAIL (any check fails, with list of violations).
```

---

## 7. Confidence Score Rubric

| Score | Label | Definition | Action |
|-------|-------|------------|--------|
| 5 | High confidence | Translation is accurate, fluent, terminologically correct, and Luxembourg-authentic | Ready to send |
| 4 | Confident with minor issues | Substantially correct; minor stylistic issues that don't affect legal meaning | Ready to send after minor polish |
| 3 | Moderate confidence | Core meaning preserved but terminology or register uncertainties | Flag for human review |
| 2 | Low confidence | Significant uncertainty about legal equivalence or potential false friends | Require human review before sending |
| 1 | Very low confidence | Major accuracy concerns, meaning reversal risk, or untranslatable concepts | Do not send without expert legal translator review |
```

- [ ] **Step 2: Verify the file was created correctly**

```bash
ls -la /Users/usmanramay/.claude/skills/translate-legal/references/mqm-legal-rubric.md
```

Expected: file exists, non-zero size.

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/translate-legal/references/mqm-legal-rubric.md
git commit -m "feat: add MQM legal evaluation rubric for translate-legal skill"
```

---

## Task 3: Create the Skill File

The main skill definition that drives the pipeline.

**Files:**
- Create: `.claude/skills/translate-legal/SKILL.md`

- [ ] **Step 1: Write the skill file**

Create `.claude/skills/translate-legal/SKILL.md` with the following complete content:

```markdown
---
name: translate-legal
description: >
  Translate English-drafted Luxembourg legal correspondence into Luxembourg-standard
  French. Use this skill whenever the user wants to translate a legal letter, mise en
  demeure, court filing, disciplinary complaint, or formal correspondence into French
  for use in Luxembourg. Invoke when the user says "translate", "translate legal",
  "mise en demeure", "Justice de Paix", "translate to French", "legal translation",
  or describes needing a French version of a legal document for Luxembourg.
  Quality is the absolute priority — this skill runs a multi-stage review pipeline.
---

# Luxembourg Legal Translation Skill

Translates English-drafted Luxembourg legal correspondence into Luxembourg-standard French through a 7-stage quality pipeline. The output must read as if written by an informed, articulate private citizen in Luxembourg — not a lawyer, not a translation.

---

## Reference Files

Before translating, load and internalize both reference files:

| File | When to use |
|------|-------------|
| `references/glossary-luxembourg-legal.md` | ALWAYS — load before Stage 2. Contains false friends, Code Civil mappings, required formulas, register rules, non-lawyer conventions |
| `references/mqm-legal-rubric.md` | ALWAYS — load before Stage 3. Contains error taxonomy, evaluation prompt templates, back-translation prompts, rules compliance checklist |

**You MUST read both reference files before beginning Stage 2.** Never guess terminology, formulas, or article numbers.

---

## Pipeline Overview

```
Stage 1: INTAKE → Stage 2: TRANSLATE → Stage 3: EVALUATE
    ↓ (errors?) → Stage 4: REFINE → back to Stage 3
Stage 5: BACK-TRANSLATE + VERIFY
    ↓ (drift?) → Stage 4: REFINE → back to Stage 3
Stage 6: RULES COMPLIANCE
    ↓ (violations?) → Stage 4: REFINE → back to Stage 3
Stage 7: OUTPUT
```

**Loop rules:**
- Max 3 refinement passes total across all stages (single counter)
- After any refinement, always return to Stage 3 (Evaluate) before resuming
- After passing Stage 3, resume from Stage 5 (Back-translate)
- If max loops reached without convergence: STOP and report unresolved issues
- Minor errors (weight 1) are fixed in-line without triggering a re-loop

---

## Stage 1: Intake

Ask the user exactly two questions before beginning translation:

**Question 1:** "Who is this letter addressed to?"
- Justice de Paix (Luxembourg, Esch-sur-Alzette, or Diekirch)
- Opposing party directly (mise en demeure / formal demand)
- Ordre des Architectes et des Ingénieurs-Conseils (OAI)
- Consumer protection body (Ministère de la Protection des consommateurs)
- Other — please specify

**Question 2:** "What type of document is this?"
- Mise en demeure (formal demand letter)
- Requête to Justice de Paix
- Disciplinary complaint
- General formal correspondence
- Other — please specify

**If the user selects "Other" for either question:** Use WebSearch to research the specific recipient's or document type's filing conventions, required format, and terminology. Show the user what you found and ask for confirmation before proceeding.

After both answers, ask the user to provide the English text to translate. Do not ask any other questions.

---

## Stage 2: Translate

**Before this stage:** Read `references/glossary-luxembourg-legal.md` if you haven't already.

Translate the full English text into French in one pass.

**Your identity for this stage:** You are a legal translation specialist for Luxembourg. You produce French text that reads as if written by an informed, articulate private citizen in Luxembourg.

**Translation rules:**
1. The English input is Luxembourg legal correspondence drafted in English. It may already contain Luxembourg legal concepts (Code Civil articles, institution names, procedural terms) expressed in English. Render these in their native French legal form — do NOT translate them as English/common law concepts.
2. Apply the full glossary: false friends watchlist, correct Code Civil article numbers (pre-2016 only), Luxembourg-specific institutional terminology.
3. Use the formula templates appropriate for the document type identified at intake.
4. Maintain the author's tone and intent exactly. Firm stays firm. Measured stays measured. Do not soften demands or amplify aggression.
5. Use correct register throughout: vouvoiement, impersonal constructions, subjunctive where required, no contractions.
6. Follow non-lawyer conventions: no professional legal titles, no bar references, private citizen format.
7. If a legal concept does not map cleanly between English and Luxembourg civil law, translate as closely as possible AND flag it explicitly — do not silently approximate.

Present the complete French translation to proceed to evaluation.

---

## Stage 3: Evaluate

**Before this stage:** Read `references/mqm-legal-rubric.md` if you haven't already.

**Your identity for this stage:** You are a senior Luxembourg legal translation reviewer — a different person from the translator.

Use the Stage 3 Evaluation Prompt Template from the rubric reference file. Fill in the placeholders with the actual document type, recipient, English original, and French translation.

Evaluate across all six dimensions: accuracy, terminology, legal-specific, fluency, register, completeness.

**Decision after evaluation:**
- If zero critical AND zero major errors → PASS → proceed to Stage 5
- If any critical OR major errors → FAIL → proceed to Stage 4 (Refine)

---

## Stage 4: Refine

Use the Stage 4 Refinement Prompt Template from the rubric reference file.

After refinement:
- Increment the loop counter (max 3 total)
- If counter > 3: STOP. Report to user: "After 3 review passes, these issues remain:" and list the unresolved errors. Ask the user how to proceed.
- If counter <= 3: Return to Stage 3 (Evaluate) with the revised translation

---

## Stage 5: Back-Translate and Verify

**Your identity for the back-translation:** You are a general-purpose translator — NOT a legal specialist. This is deliberate to avoid blind spots.

Use the Stage 5 Back-Translation Prompt Template from the rubric reference file for the back-translation.

Then use the Stage 5 Verification Prompt Template to compare the original English, the back-translation, and check for drift across 5 categories: meaning change, missing content, added content, force/tone change, legal precision change.

**Decision after verification:**
- If NO DRIFT → PASS → proceed to Stage 6
- If DRIFT detected → FAIL → proceed to Stage 4 (Refine) with drift details as the error feedback. Same loop counter.

---

## Stage 6: Rules Compliance Check

**Your identity for this stage:** You are a Luxembourg legal proofreader — a different person from both the translator and the reviewer.

Use the Stage 6 Rules Compliance Prompt Template from the rubric reference file. Run all 9 checks.

**Decision after compliance:**
- If all 9 checks PASS → proceed to Stage 7
- If any check FAILS → proceed to Stage 4 (Refine) with violations as error feedback. Same loop counter.

---

## Stage 7: Output

Present the final deliverable to the user in this exact format:

---

### Final French Translation

[The complete French translation, ready to send]

---

### Back-Translation (for your verification)

[The English rendering from Stage 5 — what the French actually says]

---

### Confidence Summary

- **Confidence score:** [1-5, using the rubric from the MQM reference]
- **Review loops taken:** [number]
- **Overall assessment:** [brief statement]

---

### Human Review Flags

[If any segments were flagged for human review, list them here with explanations. If none, state: "No flags — translation passed all review stages with high confidence."]

---

## Critical Rules

### What Works
- Loading both reference files before starting
- Using distinct mental roles for each stage (translator ≠ reviewer ≠ proofreader)
- Using a general-purpose translator identity for back-translation (avoids blind spots)
- Fixing minor errors in-line without re-looping
- Stopping at max 3 loops and reporting honestly

### What Doesn't Work
- Skipping reference files and guessing terminology
- Using the same analytical frame for translation and evaluation
- Using legal specialist framing for back-translation
- Infinite review loops (diminishing returns after 3)
- Silently approximating concepts that don't map cleanly
- Softening the user's tone or amplifying it during translation
```

- [ ] **Step 2: Verify the file was created correctly**

```bash
ls -la /Users/usmanramay/.claude/skills/translate-legal/SKILL.md
```

Expected: file exists, non-zero size.

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/translate-legal/SKILL.md
git commit -m "feat: add translate-legal skill for Luxembourg legal translation"
```

---

## Task 4: Register the Skill and Verify End-to-End

Make sure Claude Code can discover and invoke the skill.

**Files:**
- Verify: `.claude/skills/translate-legal/SKILL.md`
- Verify: `.claude/skills/translate-legal/references/glossary-luxembourg-legal.md`
- Verify: `.claude/skills/translate-legal/references/mqm-legal-rubric.md`

- [ ] **Step 1: Verify directory structure**

```bash
find /Users/usmanramay/.claude/skills/translate-legal -type f | sort
```

Expected output:
```
/Users/usmanramay/.claude/skills/translate-legal/SKILL.md
/Users/usmanramay/.claude/skills/translate-legal/references/glossary-luxembourg-legal.md
/Users/usmanramay/.claude/skills/translate-legal/references/mqm-legal-rubric.md
```

- [ ] **Step 2: Verify skill frontmatter is valid**

Check that the SKILL.md file starts with valid YAML frontmatter containing `name` and `description` fields, matching the pattern of existing skills.

```bash
head -12 /Users/usmanramay/.claude/skills/translate-legal/SKILL.md
```

Expected: YAML frontmatter block with `name: translate-legal` and a `description:` field.

- [ ] **Step 3: Verify reference files are non-empty and well-formed**

```bash
wc -l /Users/usmanramay/.claude/skills/translate-legal/references/*.md
```

Expected: both files have substantial content (glossary ~200+ lines, rubric ~200+ lines).

- [ ] **Step 4: Test skill invocation**

In a Claude Code session, run `/translate-legal` and verify:
1. The skill is discovered and loads
2. The intake questions appear (Q1: who is this addressed to, Q2: what type of document)
3. Cancel after confirming intake works — no need to run a full translation for this verification

- [ ] **Step 5: Final commit with all files**

```bash
git add .claude/skills/translate-legal/
git commit -m "feat: complete translate-legal skill with glossary and rubric references"
```

---

## Plan Self-Review

**Spec coverage check:**
- Intake phase (Stage 1) → Task 3, Stage 1 section ✓
- Translate (Stage 2) → Task 3, Stage 2 section ✓
- Evaluate (Stage 3) → Task 2 (prompt template) + Task 3 (stage logic) ✓
- Refine (Stage 4) → Task 2 (prompt template) + Task 3 (stage logic) ✓
- Back-translate (Stage 5) → Task 2 (prompt templates) + Task 3 (stage logic) ✓
- Rules compliance (Stage 6) → Task 2 (prompt template) + Task 3 (stage logic) ✓
- Output (Stage 7) → Task 3, Stage 7 section ✓
- Luxembourg legal ruleset → Task 1 (glossary) ✓
- MQM error taxonomy → Task 2 (rubric) ✓
- False friends watchlist → Task 1, Section 2 ✓
- Code Civil article mappings → Task 1, Section 1 ✓
- Required formulas by document type → Task 1, Section 5 ✓
- Non-lawyer conventions → Task 1, Section 7 ✓
- Loop control (max 3, single counter) → Task 3, Pipeline Overview ✓
- Three distinct roles → Task 3 (translator, reviewer, proofreader) ✓
- Back-translation uses different role → Task 3, Stage 5 ✓
- Web research for unknown recipients → Task 3, Stage 1 ✓
- Output includes back-translation → Task 3, Stage 7 ✓
- Human review flags → Task 3, Stage 7 ✓
- Confidence scoring → Task 2, Section 7 ✓

**Placeholder scan:** No TBDs, TODOs, or "implement later" found. All content is complete.

**Type/name consistency:** Document type and recipient variables are consistently named `{document_type}` and `{recipient}` across all prompt templates. Stage references are consistent throughout.
