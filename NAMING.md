# Mnemosys Project — Naming Convention

**Established:** 2026-08-09
**Status:** Foundational. This document governs the naming of every tool in the
`mnemosys-project` organization.

## Table of Contents

- [1. The Thesis](#1-the-thesis)
- [2. The Elder Muses](#2-the-elder-muses)
- [3. Assigned Names](#3-assigned-names)
- [4. The Roster](#4-the-roster)
- [5. Rules](#5-rules)
- [6. Sources](#6-sources)

## 1. The Thesis

The organization is named for **memory**. Each tool under it is named for a
**Muse**.

This is not decoration. The predecessor project (`wphillipmoore/mnemosys-core`)
established that the domain is not practice but *retention* — skills are memory
structures with half-lives, and the hard problem is decay, not acquisition. The
name MNEMOS was chosen to make that failure mode explicit.

The Muses are the daughters of Mnemosyne. A naming scheme in which the
organization is memory and its tools are her daughters is therefore
structurally true to the domain, not merely thematic. Every tool the project
ever builds is a particular faculty descending from memory.

## 2. The Elder Muses

The nine Olympian Muses are the familiar set. But an **older Boeotian
tradition**, recorded by Pausanias, names only three, worshipped on Mount
Helicon:

| Greek | Transliteration | Domain |
| --- | --- | --- |
| Μνήμη | **Mneme** | memory |
| Ἀοιδή | **Aoede** | song |
| Μελέτη | **Melete** | practice, study, deliberate exercise |

This elder triad is the heart of the scheme, because all three map exactly onto
the project's actual concerns:

- **Mneme** — memory. The organization itself.
- **Melete** — practice. The generation of daily exercise material.
- **Aoede** — song. Repertoire: the acquisition and retention of actual music.

The correspondence is close enough to be slightly uncanny. *Melete* does not
mean "practice" loosely; it means deliberate, effortful study — recall under
constraint. That is precisely the thing MNEMOS was built to describe.

Prefer names from the elder triad where they fit. Fall back to the nine
Olympians for tools the triad does not cover.

## 3. Assigned Names

| Name | Pronunciation | Tool | Status |
| --- | --- | --- | --- |
| **`melete`** | MEL-uh-tee | Practice exercise generator. Parameterized generation of bass exercises rendered to notation and tablature. | **Assigned** 2026-08-09 |
| **`aoede`** | ay-EE-dee | Repertoire management. Acquisition, decay modeling, and maintenance scheduling for learned material. Descends from the MNEMOSYS RPM design. | **Reserved** 2026-08-09 |
| **`urania`** | yoo-RAY-nee-uh | Performance measurement and analysis. Records a performance, compares it against the machine-readable ideal, quantifies the deviation, and localizes where playing diverged from a correct rendition. Astronomy measures the residual between a predicted position and an observed one; scoring a performance against its score is the same operation. | **Reserved** 2026-08-13 |

`mneme` is conceptually the organization itself and is **not** available on
PyPI. The org uses `mnemosys` / MNEMOS instead; `mneme` is not assigned to any
tool.

## 4. The Roster

Names available for future tools, with PyPI availability checked 2026-08-09.
PyPI is the binding constraint — repository names in this org are all free.

### Elder triad

| Name | Domain | PyPI |
| --- | --- | --- |
| Melete | practice, study | **assigned** |
| Aoede | song | **reserved** |
| Mneme | memory | taken |

### The nine Olympians

| Name | Domain | PyPI | Plausible fit |
| --- | --- | --- | --- |
| **Terpsichore** | dance | available | rhythm, time, groove, meter |
| **Urania** | astronomy | **reserved** | performance measurement and analysis — assigned as the third tool (§3) |
| **Erato** | lyric and love poetry | available | composition, songwriting |
| **Melpomene** | tragedy | available | — |
| **Thalia** | comedy, idyllic poetry | available | — |
| Calliope | epic poetry | taken | — |
| Clio | history | taken | — |
| Euterpe | music, lyric poetry | taken | — |
| Polyhymnia | sacred hymn, eloquence | taken | — |

That Euterpe — the *music* Muse — is taken on PyPI is unfortunate but
immaterial; the elder triad serves the project better anyway.

### Cicero's four

A separate late tradition (Cicero) names four Muses as daughters of Zeus:
Thelxinoe, Aoede, Arche, and Melete. Two overlap the Boeotian triad.

| Name | PyPI |
| --- | --- |
| **Thelxinoe** | available |
| Arche | taken |

## 5. Rules

1. **One Muse, one tool.** Names are not recycled or reassigned.
2. **Reserve before you need it.** A name is claimed in this document the moment
   its tool is conceived, not when work begins. `aoede` was reserved the day the
   scheme was established, precisely so it would not be rediscovered later and
   argued about.
3. **Check PyPI before assigning.** PyPI is the scarce namespace, not GitHub.
4. **The name should mean the thing.** The scheme's value is that *Melete*
   actually means practice. A Muse chosen only for sounding pleasant weakens
   every other name in the set. If no Muse fits, that is a signal worth heeding —
   possibly the tool is misconceived, or belongs inside an existing one.
5. **The organization keeps the memory name.** Mnemosys / MNEMOS is the parent;
   the Muses descend from it. Do not name the organization after a Muse.
6. **Record pronunciation.** These are unfamiliar words and will be spoken aloud
   to other people.

## 6. Sources

- **Pausanias**, *Description of Greece* 9.29.2 — the Boeotian triad of Mneme,
  Aoede, and Melete worshipped on Mount Helicon. This is the primary attestation
  for the elder tradition.
- **Hesiod**, *Theogony* 77–79 — the canonical nine Olympian Muses, daughters of
  Zeus and Mnemosyne.
- **Cicero**, *De Natura Deorum* 3.21 — the alternative set of four.
- Etymology: Μελέτη from *meletaō*, "to care for, to practice, to study";
  Μνήμη from the root of *mnēmē*, memory, which also yields *mnemonic*. Both
  descend from the same conceptual root as Mnemosyne.

The naming rationale is also recorded in melete's design specification
([`epics/1-org-bootstrap-melete-v1/spec.md`](epics/1-org-bootstrap-melete-v1/spec.md),
§2 Name and Lineage) and as decision #1 in that document's decision table.
**This file is the authoritative version.**

A note on what this table does *not* record: the tool a name is assigned to is
described by what it produces, never by the library that produces it. An earlier
version of melete's row named LilyPond as the renderer, which put a replaceable
implementation choice in the organization's most permanent document. The
renderer is now being replaced (epic #1 spec §4, *The renderer boundary*) and the
row needed no change.
