---
owner: group:default/dto
last_reviewed: 2026-11-17
---

# 3CX ↔ Zammad Bridge

!!! note "Owner to confirm"
    TODO(owner): confirm that `group:default/dto` is the owning team for this component.

## What it does

`zammadbridge` is a small Go daemon. It polls AGM's 3CX phone system for active calls and sends each call's events (new call, answer, hangup) to Zammad's CTI webhook, so Zammad can show the call in its CTI caller log.

AGM added more on top of the open-source bridge:

- **Phone tickets on hangup.** When a call ends, the bridge creates a Zammad ticket in one group, with a "Phone Call" article. If the caller is not a Zammad user yet, it creates one first.
- **Filters.** Only calls in the chosen directions (inbound, outbound, both or none), answered or placed by the chosen 3CX extensions, create tickets. If no filter mode is set, nothing is created (fail closed).
- **Repeat-call de-duplication.** A repeat call from the same number is added to the caller's newest open ticket instead of opening a new one, if that ticket was created no more than a set number of minutes ago.
- **3CX call ID.** Each article shows the 3CX call ID. It can also be written into a Zammad ticket field so tickets can be joined to the 3CX call log.
- **Admin page.** A password-protected web page lets an operator change the filters and the de-duplication window. Changes apply at once, without a restart, but only when the bridge can write its config file (see [Known gaps](reference.md#known-gaps), item 3).

This repo is AGM's fork of [qmexnetworks/3cx-zammad-bridge](https://github.com/qmexnetworks/3cx-zammad-bridge). It was forked at upstream release v0.8.0 and carries AGM changes that were never sent upstream. The full list is in [Reference → AGM changes](reference.md#agm-changes).

## Who uses it

- **London Cab (LC) call-centre agents and supervisors.** Their calls become tickets in the Zammad group set by `Zammad.ticket_group`. The production filter limits ticket creation to LC extensions.
- **Operators who maintain the LC extension list.** They use the admin page, or ask DTO to edit the config file on the host if admin-page saves fail.
- **DTO engineers.** They build, deploy and support the bridge.

TODO(owner): confirm the list of business users, and whether any business line other than London Cab now relies on the bridge.

## Business owner

London Cab (`group:default/london-cab`), per the DevHub card audit of 2026-11-09. The card still flags this ("London Cab or company-wide?"). TODO(owner): confirm the business owner, and name the person who decides which calls should create tickets.

## PM

None, per the DevHub card audit of 2026-11-09. TODO(owner): confirm there is no PM.

## Links

| What | Link |
|---|---|
| Production: Zammad (where tickets land) | <https://helpdesk.agmegypt.com> |
| Production: bridge admin page (password-protected) | <https://helpdesk.agmegypt.com/bridge-admin/> |
| Other environments | None. There is no dev or staging bridge. |
| Repo (AGM fork) | <https://github.com/AGM-One-Vision/3cx-zammad-bridge> |
| Upstream | <https://github.com/qmexnetworks/3cx-zammad-bridge> |
| DevHub card | <https://devhub.agmegypt.com/catalog/default/component/3cx-zammad-bridge> |
| DevHub system | <https://devhub.agmegypt.com/catalog/default/system/zammad> |
| Zammad source (AGM fork) | <https://github.com/AGM-One-Vision/zammad> |
