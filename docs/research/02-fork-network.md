# Original branches and CAD fork network

**Snapshot:** 2026-10-26 in America/Toronto; checks completed 2026-10-27 03:32:54 UTC.  
**Scope:** the CAD repository `jyjblrd/single-propeller-drone`, its current public branches and releases, and the 91 discovered public forks whose repository metadata identifies this original as their source. Firmware repositories are separate projects and are reviewed in the adjacent firmware research.

## Finding

There is no committed alternative implementation to import from any currently exposed branch in the 91 verified CAD forks. Each fork exposes exactly one branch, `main`, and every branch points to the same commit as the original:

`efc9176bfc83777b870b967c1f62a686c099be2c`

This includes [your fork at the reviewed commit](https://github.com/k2jac9/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c). The classification comes from actual branch HEAD SHA equality, rather than repository names, sizes, recent push timestamps, or appearance. Equal commit identities mean equal committed trees and ancestry at these refs. A separate compare API call per identical fork would add no additional ancestry information. Had a different HEAD appeared, the next step would have been an ancestor/compare request and inspection of its actual commits and changed files.

The original itself exposes only `main`; the requested “other branches” do not exist in its current public branch listing. The original has no tag refs, no releases, and no open or closed pull requests in the queried collections. The filenames `prop_v1` and `prop_v2` identify two CAD variants within the same tree; they are not separate branches, releases, or evidence of tested performance progression. Sources: [branches](https://api.github.com/repos/jyjblrd/single-propeller-drone/branches?per_page=100&page=1), [tag refs](https://api.github.com/repos/jyjblrd/single-propeller-drone/git/matching-refs/tags/), [releases](https://api.github.com/repos/jyjblrd/single-propeller-drone/releases?per_page=100&page=1), [all-state PRs](https://api.github.com/repos/jyjblrd/single-propeller-drone/pulls?state=all&per_page=100&page=1), and the [pinned file tree](https://github.com/jyjblrd/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c).

## What was actually enumerated

The connector accepts repository, branch, Git-data, commit, and PR reads, but rejected the direct REST `/forks` and `/tags` collection URLs as outside its approved URL families. A public web attempt to read the forks page and REST forks collection was also unavailable. These were capability failures, not evidence that the collections were empty.

Discovery therefore used supported repository search with 100 results per page. All three searches were followed by a second page, which returned no additional results:

| Search | Role and observed result |
| --- | --- |
| `single-propeller-drone in:name fork:true` | 91 candidates: original, 89 actual forks, and an unrelated propulsion test rig |
| `"Under-actuated drone with only one motor" fork:only` | Recovered two renamed forks, `Hailmit/1-c-nh-drone` and `closerx/single-drone-flyeye`, missing from the name search |
| `single propeller drone fork:only` | Corroborated overlapping results and surfaced an unrelated fork of another project |

For each candidate, `GET /repos/{owner}/{repo}` was used to check `fork`, `parent.full_name`, and `source.full_name`. All 91 included forks identify `jyjblrd/single-propeller-drone` as both parent and source. The unrelated `TonsakTS/Single-Propeller-Drone-Propulsion-Test-Rig` is not a fork, and `danielmirand/Beak` belongs to the `UCSD-Whirly-Birdy/Beak` network; both were excluded from CAD fork conclusions.

Each included repository then received its own `GET /repos/{owner}/{repo}/branches?per_page=100&page=1` request. Every response had one entry, so no branch collection reached the 100-entry pagination boundary. This covers 92 branch refs across the original plus 91 forks. There were no failed membership or branch requests for the included set. The original [repository metadata](https://api.github.com/repos/jyjblrd/single-propeller-drone) reported both `forks_count` and `network_count` as 91 during this snapshot: the discovered and individually verified fork count matches both counters.

This is strong coverage of the publicly discoverable network at the stated time. Search results are not a permanent inventory API; matching a counter does not prove access to hidden, deleted, unindexed, future, detached, or unreachable refs. Branch reads are also sequential observations, not an atomic transaction over all repositories. The machine-readable companion records the exact SHA, branch-list source URL, comparison method, and check time for each included repository. It contains no raw owner profiles, email addresses, or unrelated repository payloads: [fork-network-2026-10-26.json](../../research-data/fork-network-2026-10-26.json).

## Original history: the work that does exist

The original's reachable `main` history contains six commits, all dated 2026-10-10. Its initial commit delivered the CAD artifacts, README, and demonstration media. The five later commits update documentation and presentation:

| Commit | Recorded change |
| --- | --- |
| [9f18de17](https://github.com/jyjblrd/single-propeller-drone/commit/9f18de17eacff3bb32a0796a703291f9e2aa5e18) | Add CAD files, README and demo animation |
| [2b7ad77b](https://github.com/jyjblrd/single-propeller-drone/commit/2b7ad77b7af9931b7a9392627e68834b35cef1eb) | Add test stand clip to README |
| [96a85626](https://github.com/jyjblrd/single-propeller-drone/commit/96a856266596dc06cc5842ee08352576d9947ffd) | Update README |
| [786801dd](https://github.com/jyjblrd/single-propeller-drone/commit/786801dd890897c5cc0b224377bfa4ce86cc64f4) | Show test stand clip as a looping GIF |
| [03f735be](https://github.com/jyjblrd/single-propeller-drone/commit/03f735befabae523bdb319e647e9fb493c1856ab) | Use a still render in the README |
| [efc9176b](https://github.com/jyjblrd/single-propeller-drone/commit/efc9176bfc83777b870b967c1f62a686c099be2c) | Link the ArduPilot and moteus forks |

The [initial-to-current comparison](https://github.com/jyjblrd/single-propeller-drone/compare/9f18de17eacff3bb32a0796a703291f9e2aa5e18...efc9176bfc83777b870b967c1f62a686c099be2c) reports five commits ahead and zero behind. Its changed-file list contains README and media changes only; none of the body, prop plate, or propeller CAD files changed. Consequently, this public history offers no later mechanical revision to cherry-pick beyond the initial CAD release.

The current [README](https://github.com/jyjblrd/single-propeller-drone/blob/efc9176bfc83777b870b967c1f62a686c099be2c/README.md) expressly points to [jyjblrd/ardupilot](https://github.com/jyjblrd/ardupilot) for the flight controller and [jyjblrd/moteus](https://github.com/jyjblrd/moteus) for the motor controller. Those links matter more for development than selecting another identical CAD fork. They also mean that this small CAD repository is not a complete flight-software source tree.

## Verified fork inventory

Every link below is pinned to the observed commit. “Identical” means zero unique commits relative to the original's reviewed `main` HEAD. All rows share the full SHA printed above; the compact JSON preserves it per row.

| Fork | Exposed branch | Comparison |
| --- | --- | --- |
| [19-front/single-propeller-drone](https://github.com/19-front/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [Abhishek-Kaudare/single-propeller-drone](https://github.com/Abhishek-Kaudare/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [Adam-Fresko/single-propeller-drone](https://github.com/Adam-Fresko/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [Akkuzin/single-propeller-drone](https://github.com/Akkuzin/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [alexseysua/single-propeller-drone](https://github.com/alexseysua/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [Allenmylath/single-propeller-drone](https://github.com/Allenmylath/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [Aniekanetim/single-propeller-drone](https://github.com/Aniekanetim/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [Aquahoodjd/single-propeller-drone](https://github.com/Aquahoodjd/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [ardritkrasniqi/single-propeller-drone](https://github.com/ardritkrasniqi/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [Aureum01/single-propeller-drone](https://github.com/Aureum01/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [b0rk901/single-propeller-drone](https://github.com/b0rk901/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [boriskra/single-propeller-drone](https://github.com/boriskra/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [BurhanUlTayyab/single-propeller-drone](https://github.com/BurhanUlTayyab/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [C3nsoreD/single-propeller-drone](https://github.com/C3nsoreD/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [caffedev-dev/single-propeller-drone](https://github.com/caffedev-dev/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [Carl-Thibault/single-propeller-drone](https://github.com/Carl-Thibault/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [celsopitta/single-propeller-drone](https://github.com/celsopitta/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [charanakreddy/single-propeller-drone](https://github.com/charanakreddy/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [chenguoming1/single-propeller-drone](https://github.com/chenguoming1/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [choyunsung/single-propeller-drone](https://github.com/choyunsung/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [closerx/single-drone-flyeye](https://github.com/closerx/single-drone-flyeye/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [cnr437/single-propeller-drone](https://github.com/cnr437/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [codesmith55/single-propeller-drone](https://github.com/codesmith55/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [delterio/single-propeller-drone](https://github.com/delterio/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [dmitryzenit/single-propeller-drone](https://github.com/dmitryzenit/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [Egortex/single-propeller-drone](https://github.com/Egortex/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [EricEisaman/single-propeller-drone](https://github.com/EricEisaman/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [f16v1per/single-propeller-drone](https://github.com/f16v1per/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [fiamor/single-propeller-drone](https://github.com/fiamor/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [Fridge2theface/single-propeller-drone](https://github.com/Fridge2theface/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [frontcover/single-propeller-drone](https://github.com/frontcover/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [giordanoshai/single-propeller-drone](https://github.com/giordanoshai/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [GusainovRasim/single-propeller-drone](https://github.com/GusainovRasim/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [gyillikci/single-propeller-drone](https://github.com/gyillikci/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [Hailmit/1-c-nh-drone](https://github.com/Hailmit/1-c-nh-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [HassanKanso/single-propeller-drone](https://github.com/HassanKanso/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [IntrovertInDisguise/single-propeller-drone](https://github.com/IntrovertInDisguise/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [ixobert/single-propeller-drone](https://github.com/ixobert/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [izzy-Davies04/single-propeller-drone](https://github.com/izzy-Davies04/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [j-a-chiang/single-propeller-drone](https://github.com/j-a-chiang/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [johnsonminzIdc/single-propeller-drone](https://github.com/johnsonminzIdc/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [juano327/single-propeller-drone](https://github.com/juano327/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [Just-Antonio/single-propeller-drone](https://github.com/Just-Antonio/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [k2jac9/single-propeller-drone](https://github.com/k2jac9/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [kalmancito/single-propeller-drone](https://github.com/kalmancito/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [kpbm-ai/single-propeller-drone](https://github.com/kpbm-ai/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [Krishna1408/single-propeller-drone](https://github.com/Krishna1408/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [kuzja111/single-propeller-drone](https://github.com/kuzja111/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [lilacmountain/single-propeller-drone](https://github.com/lilacmountain/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [llmra/single-propeller-drone](https://github.com/llmra/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [mackgerhadt/single-propeller-drone](https://github.com/mackgerhadt/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [marekbujko/single-propeller-drone](https://github.com/marekbujko/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [mcleber/single-propeller-drone](https://github.com/mcleber/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [medusa110/single-propeller-drone](https://github.com/medusa110/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [mertz3hack/single-propeller-drone](https://github.com/mertz3hack/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [mikendungu/single-propeller-drone](https://github.com/mikendungu/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [mkantowski/single-propeller-drone](https://github.com/mkantowski/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [MondayInRussian/single-propeller-drone](https://github.com/MondayInRussian/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [motsumi-makhene/single-propeller-drone](https://github.com/motsumi-makhene/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [mrmx/single-propeller-drone](https://github.com/mrmx/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [MrTypename/fork_single-propeller-drone](https://github.com/MrTypename/fork_single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [msitarzewski/single-propeller-drone](https://github.com/msitarzewski/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [Nonnyjoe/single-propeller-drone](https://github.com/Nonnyjoe/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [oilcrest/single-propeller-drone](https://github.com/oilcrest/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [oluwaseunmauwedo/single-propeller-drone](https://github.com/oluwaseunmauwedo/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [ondermogol/single-propeller-drone](https://github.com/ondermogol/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [pollstar/single-propeller-drone](https://github.com/pollstar/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [porom004/single-propeller-drone](https://github.com/porom004/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [Prasad-viduranga/single-propeller-drone](https://github.com/Prasad-viduranga/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [prolifixs/single-propeller-drone](https://github.com/prolifixs/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [rafal-rr/single-propeller-drone](https://github.com/rafal-rr/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [robinluojl/single-propeller-drone](https://github.com/robinluojl/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [rozgo/single-propeller-drone](https://github.com/rozgo/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [rubenbisso/single-propeller-drone](https://github.com/rubenbisso/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [rustyleforsite/single-propeller-drone](https://github.com/rustyleforsite/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [Sackstools/bom-single-propeller-drone](https://github.com/Sackstools/bom-single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [sadhu-sanjay/single-propeller-drone](https://github.com/sadhu-sanjay/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [sehb/single-propeller-drone](https://github.com/sehb/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [suryatmodulus/single-propeller-drone](https://github.com/suryatmodulus/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [tadryanom/jyjblrd_single-propeller-drone](https://github.com/tadryanom/jyjblrd_single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [TalkData2Me/single-propeller-drone](https://github.com/TalkData2Me/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [taufiqrahman/single-propeller-drone](https://github.com/taufiqrahman/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [theeofficialtj/single-propeller-drone](https://github.com/theeofficialtj/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [theproductstudiohqsecurity/single-propeller-drone](https://github.com/theproductstudiohqsecurity/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [tsai1278/single-propeller-drone](https://github.com/tsai1278/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [umuttalha/single-propeller-drone](https://github.com/umuttalha/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [vivek-nani/single-propeller-drone](https://github.com/vivek-nani/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [wolfpack-tech/single-propeller-drone](https://github.com/wolfpack-tech/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [xpersian/single-propeller-drone](https://github.com/xpersian/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [xtbanban/single-propeller-drone](https://github.com/xtbanban/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |
| [yunho-c/single-propeller-drone](https://github.com/yunho-c/single-propeller-drone/tree/efc9176bfc83777b870b967c1f62a686c099be2c) | main | Identical |

## What this means for evolving your fork

The useful baseline is your existing fork at the recorded commit. Switching to one of these 90 other forks provides no additional committed implementation at this snapshot. Repository renames such as “bom” or “flyeye” do not establish the presence of a bill of materials, revised airframe, or flight-tested changes: the verified commit is still identical.

Further development should establish new evidence in your fork: a reproducible hardware configuration, explicitly selected CAD revision, measured mass and inertia assumptions, pinned flight-controller and motor-controller commits, and bench-test results. Treat upstream CAD, flight firmware, motor firmware, parameters, and hardware versions as a matched set to be validated. The fact that separate repositories belong to the same author, or share a single-propeller description, does not establish compatibility between arbitrary branches or historical versions. The adjacent design and firmware reviews develop these engineering steps.

For future network checks, preserve the present SHA as a baseline, rerun both name and description searches, verify repository membership, and enumerate every exposed branch. For each changed HEAD, use the actual comparison status: `ahead` means new descendant commits, `behind` means missing baseline commits, and `diverged` means both sides have unique commits. Inspect the diff and relevant full files before deciding whether to port an idea. A recent push timestamp can prioritize discovery but should never serve as proof that a fork contains an improvement.

No divergent branch or unique CAD fork commit was found to inspect during this audit. That absence describes published commits at the queried refs; it says nothing about builders' private experiments, local files, issues elsewhere, or unreleased work.

## License and reuse observations

The original repository metadata reports `license: null`, and the complete [recursive tree at the reviewed SHA](https://api.github.com/repos/jyjblrd/single-propeller-drone/git/trees/efc9176bfc83777b870b967c1f62a686c099be2c?recursive=1) contains no license file. Every included fork's exposed HEAD is the same commit, so none supplies an additional committed license document in that tree. A fork operation or public visibility does not itself resolve the original CAD's reuse terms.

This observation is limited to the inspected repository and commit. It does not establish the absence of permissions granted elsewhere. Before redistribution, commercial reuse, or incorporating independently licensed firmware, identify the applicable permissions and each component's actual license. If a future fork adds a license declaration, its presence alone would not establish that the contributor can license all inherited third-party content.

## Reproducibility and limits

The core result can be checked with read-only GitHub requests: original repository metadata, the three paginated discovery searches, each candidate's source/parent metadata, and each included branch list. Empty original tag refs, releases, and all-state PR results were read separately. An ordinary `/tags` request failed, but `/git/matching-refs/tags/` successfully returned an empty array; these outcomes are distinguished deliberately.

The primary evidence is the pinned commit identity and its tree, with mutable API collection links retained for rerunning the inventory. The JSON file is a normalized snapshot rather than an archive of every API response. The network audit itself was read-only. Publication of this documentation subsequently adds a branch and draft PR to the development fork; those are outside the recorded snapshot and do not represent a CAD or flight-control change.
