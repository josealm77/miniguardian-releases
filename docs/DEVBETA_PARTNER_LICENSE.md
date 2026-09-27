# Jayce Automata Research Dev Beta Partner License v1.0

Copyright (c) 2026 Jose Almendarez / Jayce Automata Research
All rights reserved.

This License governs the grant of the Software to Development Beta Partners
("Partners") of the Jayce Automata Research program. It is a single,
all-components license: it covers the MiniGuardian program, the Jayce kernel,
the Jayce SDK, and every other unique component of the Jayce system, whether
delivered as source, sealed artifact, or documentation.

---

## 1. DEFINITIONS

**"Software"** means, collectively and individually, all components of the
Jayce system provided by Jayce Automata Research to Partners under this
License, including but not limited to:

- **The MiniGuardian program** — the daemon, sentinel, status consoles,
  saddle operations consoles, terminals, and all associated workspace crates
  (miniguard, miniguard-status, trust_core, trust_terminal, jvc_compressor_rt,
  jayce_kernel, cosmic_map, apodeixis, astraxis, mg-supply, mg_collaboration,
  and any others).
- **The Jayce kernel** — the bare-metal deterministic kernel, its
  multi-architecture HALs, engine registry, source tree, build scripts, and
  any sealed kernel images or artifacts derived from them.
- **The Jayce SDK** — the SDK bundle and all first-party crates and materials
  within it (TRust workspace crates, apodeixis, astraxis,
  jvc_compressor_rt, OS templates, scripts).
- **Other unique components** — the Shim ABI, the JAYCE_TLM telemetry format,
  the capability model, the engine-lineage protocol, documentation, manuals,
  whitepapers, branding, scripts, and any other component authored by Jayce
  Automata Research and delivered under this License.

**"Sealed Components"** means the Jayce kernel and the MiniGuardian daemon as
deployed (read-only, immutable, encrypted or statically linked binaries and
images). Sealed Components are provided as artifacts only; their internal
source is not part of this grant.

**"Non-Sealed Components"** means all parts of the Software other than Sealed
Components, including the SDK crates, templates, source trees, and scripts.

**"Jayce Automata Research"** means Jose Almendarez and the Jayce Automata
Research project.

**"Partner"** means a natural person or legal entity accepted into the Jayce
Automata Research Development Beta program and to whom this License is granted
in writing by Jayce Automata Research.

**"Beta Period"** means the term of the Partner's acceptance into the
Development Beta program, as stated in the Partner's acceptance or program
notice, until terminated under Section 9.

**"Field Trial"** means deployment and operation of the Software, in whole or
in part, on a limited, bounded number of devices controlled by the Partner for
evaluation purposes during the Beta Period.

**"Commercial Use"** means any use of the Software, in whole or in part, for
direct or indirect commercial advantage or monetary compensation, including
resale, licensing, hosting for third parties, or use in the Partner's
revenue-generating products or services.

**"Weaponization"** means any use of the Software, in whole or in part, to
design, develop, test, deploy, or support offensive weapons, autonomous
attack systems, surveillance infrastructure used to harm individuals, or any
system intended to cause physical, digital, or psychological harm to persons.

---

## 2. GRANT OF RIGHTS

Subject to the terms and conditions of this License, Jayce Automata Research
grants each Partner a limited, non-exclusive, non-transferable, revocable
right, during the Beta Period only, to:

1. **View and study** the Software for learning, research, and development.
2. **Modify, fork, and create derivative works** of the Non-Sealed Components
   for the purpose of integrating, testing, and building the Partner's own
   systems on the Jayce substrate. This right does NOT extend to Sealed
   Components: Partners may use Sealed Components as provided but may not
   read, patch, hook, decompile, or bypass them.
3. **Build, compile, and run** the Software in development and test
   environments, and in Field Trials as defined in Section 4.
4. **Integrate** the Software with the Partner's own systems through the
   documented integration seams (Shim surface, AI adapter, alien bridge,
   multiverse engine, telemetry, and cohost management socket).
5. **Provide feedback** to Jayce Automata Research on the Software's
   functionality, performance, and security.

No other rights are granted. In particular, Partners may not redistribute,
publish, sublicense, sell, or make publicly available the Software or any
derivative work without prior written permission from Jayce Automata Research.

---

## 3. PARTNER OBLIGATIONS

All exercise of rights granted above must comply with all of the following:

**a. ATTRIBUTION.** Any build, derivative work, or integration of the
Software must prominently display: "Built on Jayce — Copyright (c) 2026 Jose
Almendarez / Jayce Automata Research. All rights reserved."

**b. NO COMMERCIAL USE.** Commercial Use of the Software, in whole or in part,
is not authorized under this License and requires a separate written
commercial agreement with Jayce Automata Research. To request a commercial
license, contact: <mailto:JayceAutomataResearch@proton.me>

**c. NO REDISTRIBUTION.** Partners may not redistribute, publish, sublicense,
sell, or make publicly available the Software, any Sealed Component, or any
derivative work, in whole or in part, without prior written permission from
Jayce Automata Research.

**d. NO WEAPONIZATION.** Weaponization of the Software is strictly and
unconditionally prohibited. This prohibition is non-waivable and cannot be
overridden by any commercial license, agreement, or permission from Jayce
Automata Research. No party has authority to grant permission for
Weaponization under any circumstance.

**e. GATE INTEGRITY.** Any public claim of gate scores, security scores,
determinism scores, certification readiness, or promotion status derived from
the Software must be backed by artifacts produced by running the official
gate and rubric scripts in the Software unmodified. Fabricated or manually
adjusted score claims are prohibited.

**f. ETHICS PRESERVATION.** Any derivative work, personality layer, or system
built above the Software must not remove, suppress, or circumvent the
fail-closed promotion policy, the capability gate model, the weaponization
prohibition, the no-weaponization clause, or the constitutional invariant
constraints documented in the Software.

**g. SEALED COMPONENT INTEGRITY.** Partners must not attempt to read, patch,
hook, decompile, or bypass Sealed Components, and must not defeat or weaken
the anti-tamper or anti-reverse-engineering protections of any Sealed
Component.

**h. FIELD TRIAL BOUNDS.** Field Trials are permitted only during the Beta
Period, on a limited number of devices, solely for evaluation. Field Trial
deployments must not be made available to the public or to third parties.

**i. NON-DISCLOSURE.** The Software and all materials provided under this
License are unpublished and confidential until Jayce Automata Research
publishes them or the Beta Period ends. Partners must not disclose the
Software, its capabilities, its performance, or its existence to any third
party (including in press releases, blogs, or social media) without prior
written permission from Jayce Automata Research.

**j. FEEDBACK LICENSE.** To the extent Partners provide feedback,
suggestions, improvements, or bug reports to Jayce Automata Research, Partners
grant Jayce Automata Research a perpetual, irrevocable, worldwide,
non-exclusive right to use that feedback in the Software and its derivatives
without compensation or attribution, subject to the no-weaponization clause.

---

## 4. FIELD TRIALS

During the Beta Period, a Partner may operate the Software, or systems
integrating it, on Field Trial devices that the Partner controls, provided:

- The number of Field Trial devices is reasonable and bounded, as stated in
  the Partner's program acceptance.
- Field Trial operation is for evaluation only and does not constitute
  Commercial Use.
- No Field Trial device or its data is made available to the public.
- On termination of this License, Field Trial deployments are discontinued and
  the Software is removed from those devices.

---

## 5. RESERVATION OF RIGHTS

All rights not expressly granted in this License are reserved by Jayce
Automata Research. This includes, without limitation:

- The right to publish or distribute the Software.
- The right to use the Jayce or Jayce Automata Research name or logo.
- The right to make commercial use of the Software.
- The right to embed the Software in products or services.
- The right to terminate or revoke any Partner's rights under this License.

---

## 6. UPDATES AND DISTRIBUTION

Official releases are published at:
<https://www.jayceautomata.com/releases/>

Only releases signed with the official Jayce Automata Research developer key
and distributed via the official site are authorized releases. Third-party
redistribution of release packages is prohibited without written permission.

---

## 7. DISCLAIMER OF WARRANTIES

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE, AND NONINFRINGEMENT. IN NO EVENT SHALL JAYCE
AUTOMATA BE LIABLE FOR ANY CLAIM, DAMAGES, OR OTHER LIABILITY, WHETHER IN AN
ACTION OF CONTRACT, TORT, OR OTHERWISE, ARISING FROM, OUT OF, OR IN CONNECTION
WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

---

## 8. TERMINATION

Your rights under this License terminate automatically and immediately if you
violate any condition of this License, including the weaponization
prohibition. The Beta Period itself may also be ended by Jayce Automata
Research at any time, with or without notice. Upon termination, you must:

- Cease all use of the Software.
- Discontinue all Field Trials.
- Destroy all copies of the Software and any derivative works in your
  possession, or return them to Jayce Automata Research if requested.

Sections 3(d), 3(f), 3(i), 3(j), 5, 7, and 8 survive termination.

---

## 9. GOVERNING LAW

This License shall be governed by the laws of the jurisdiction in which Jose
Almendarez resides, without regard to conflict of law provisions.

---

## 10. CONTACT

Jayce Automata Research
Website: <https://www.jayceautomata.com>
Licensing: <mailto:JayceAutomataResearch@proton.me>

---

## NOTE ON THIRD-PARTY CODE

The Software builds on third-party open-source components which remain under
their own licenses (MIT, Apache-2.0, BSD, ISC, Zlib, and others). Those
components are attributed in the relevant THIRD_PARTY_NOTICES.md files, with
their license texts under the corresponding licenses/ directories. This
License applies to the Software authored by Jayce Automata Research, not to
those third-party components, and nothing in this License grants or restricts
rights under third-party licenses.
