- Feature Name: rust-lang_github_org_access_policy 
- Start Date: 2020-03-02 

# Summary
[summary]: #summary

This RFC proposes a policy for managing permissions to the [Rust-Lang GitHub Organization](https://www.github.com/rust-lang) and repositories within this organization.

This RFC was written in consultation with the Governance Working Group and the Infrastructure team. Most discussion took place on [this issue](https://github.com/rust-lang/wg-governance) and [this pull request](https://github.com/rust-lang/wg-governance/pull/42).

# Motivation
[motivation]: #motivation

Access control for the [Rust-Lang GitHub Organization](https://www.github.com/rust-lang) and repositories within that organization is currently managed ad-hoc. We need a policy that defines how these accesses are granted and managed. This will allow us to have greater security in permissions to our GitHub org and also allow the infra team to build appropriate tooling to automate access control when possible.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Rust-Lang GitHub Permissions Policy
This policy applies to both the [Rust-Lang GitHub Organization](https://github.com/rust-lang/) and all repositories within that organization.

### Rust-Lang Organization
Membership in the Rust-Lang GitHub organization is managed by the organization owners.

All members of the [Core Team](https://github.com/rust-lang/team/blob/master/teams/core.toml) have the right to be organization owners.

Selected members of the [Infrastructure Team](https://github.com/rust-lang/team/blob/master/teams/infra.toml) can also be organization owners if their work requires it.

Owners should use a separate account from their main GitHub account dedicated to managing the organization. This account may not be used to commit code and must have 2FA enabled.

### Rust-Lang Repositories

Access to and permissions for repositories within the Rust-Lang organization should be administered through GitHub teams. Rust-Lang GitHub teams are administered through the [Team repository](https://github.com/rust-lang/team).

GitHub provides several permission levels for access to a repository. Please refer to [GitHub's documentation](https://help.github.com/en/github/setting-up-and-managing-organizations-and-teams/repository-permission-levels-for-an-organization) for details on permission levels and what each level can do.

Repositories in the Rust-Lang organization should follow these permission guidelines:

Permissions should **only** be given to teams within the Rust-Lang organization, not to individuals.

* **Admin** - only Rust team or working group leads should have this permission level
* **Write** - contributors within GitHub teams may have this permission level at the discretion of the team leads
* **Triage** - contributors within GitHub teams involved in issue and pull request management (i.e. closing, re-opening, and assigning issues and pull requests) should be granted this access
* **Read** - by default, everyone should have access to read repositories

By default, repositories should be public and allow read access to all. When needed, some repositories can have limited read access (i.e. repositories related to security). 

Bot accounts controlled by the Infrastructure Team (such as the [Rust High Five Bot](https://github.com/rust-highfive)) can be granted any level of access required for them to work at the discretion of the Infrastructure Team.

# Drawbacks
[drawbacks]: #drawbacks

This policy would add more structure to managing GitHub permissions for both the [Rust-Lang GitHub Organization](https://github.com/rust-lang) and all repositories within it. Some might find this structure slows them down and alters their current workflow.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Should these rules applied to Rust-Lang affiliated repositories and organizations that are outside of the [Rust-Lang GitHub Org](https://www.github.com/rustlang)?
- Should we automate this process?
- How do we ensure that changes to the [Teams Repository](https://github.com/rust-lang/team) are reviewed and merged promptly?