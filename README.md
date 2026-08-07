# Morgan Stanley Résumé Projects: Build Plan and Knowledge Packets

Rebuild every project claimed in the Morgan Stanley GSF section of your résumés, from public data, in
public repos. Twenty-five distinct bullets across twenty tailored résumés map to seventeen buildable
projects.

## Start here

1. **[`BUILDABILITY.md`](BUILDABILITY.md)** which of the 25 bullets can actually be built, which need
   a public proxy, and which are not artifacts at all. Includes a recommended cut.
2. **[`PLAN.md`](PLAN.md)** the seventeen projects, build order, dependencies, data sources, and
   definition of done.
3. **[`prds/`](prds/)** agent-executable build specs, one per project, with numbered milestones and
   binary acceptance criteria. Start with [`prds/00-CONVENTIONS.md`](prds/00-CONVENTIONS.md), then
   hand a PRD to Claude Code one milestone at a time.
4. **[`knowledge-packets/`](knowledge-packets/)** one packet per project: domain background, data
   with gotchas, method, numbers to know cold, and the interview questions you will actually get.

## To start building

```
Read ~/ms-resume-projects/prds/00-CONVENTIONS.md and ~/ms-resume-projects/prds/06-utility-affordability-PRD.md.
Then read ~/ms-resume-projects/knowledge-packets/06-utility-affordability.md for domain background.
Execute milestone M0 only. Run its acceptance check. Commit. Then stop and report.
```

## The one-paragraph version

19 of 25 bullets are fully buildable from public data. 4 need an open substitute for a walled vendor
or licensed dataset, and you say so in the README rather than hiding it. 2 (`ambassador`, `audit`)
are role and process claims that no repo will ever substantiate. The four highest-leverage builds are
the utility affordability dataset (on 19 of 20 résumés), the nowcast pipeline (17), the parent
crosswalk (13), and the AI weather benchmark (12). Those four cover the most-used claims in about 92
hours across two repos.

## Non-negotiables

- No Morgan Stanley code, data, output, or screenshots enter these repos. Rebuild from public
  sources only.
- `~/Downloads/el_nino_research_handoff.md` is entitlement-gated MS Research. It stays local and
  private, and nothing derived from it is published.
- Route anything market-facing through the outside-activity and personal-publication policy before
  publishing. The ENSO basket (P1) is the one to clear first.
- Your public rebuilds will produce different numbers than the private originals. That is expected.
  Never retrofit a public result to match a private one.

## Reuse what exists

- `~/projects/ngfs-scenario-explorer` is most of P15.
- Your 23-company AI equity research platform is most of P13.
- `~/nvidia-earth2-learning` and your AI weather modeling book feed P4 and P5.
