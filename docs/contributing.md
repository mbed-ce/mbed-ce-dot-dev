# Contributing to Mbed CE

Mbed CE is a community-driven project, and we would love your contributions! This page will provide some details on how you can contribute to Mbed CE/

## Useful Links

- [Mbed CE Main Repository](https://github.com/mbed-ce)
- [Mbed CE Dot Dev Repository](https://github.com/mbed-ce/mbed-ce-dot-dev) (this website, for docs)
- [Mbed CE Issues](https://github.com/mbed-ce/mbed-os/issues) (go here to report a bug)
- [Mbed CE Discussions](https://github.com/mbed-ce/mbed-os/discussions) (go here to ask a question or request a feature)

## Contribution Instructions

### Filing an issue
Click [this link](https://github.com/mbed-ce/mbed-os/issues/new/choose), then describe your issue. It's recommended to @mention a maintainer (see the Maintainers page for who maintains what) so that your issue is seen right away.

### Submitting a change

1. Fork the `mbed-ce/mbed-os` repository to your Github account
2. Create a branch for your changes
3. Make your changes and commit them to your fork.
4. [Submit](https://github.com/mbed-ce/mbed-os/compare/master...mbed-ce:mbed-os:master) a pull request (PR) to `mbed-ce/mbed-os` that compares your branch against the Mbed CE master branch. Be careful not to compare against the `ARMMbed/mbed-os` master branch, as sometimes github defaults to doing that!
5. Check the results of the Github Actions run after your code is submitted, and fix any issues detected.

It's best practice to ensure that each pull request only makes, at a high level, one "change" to Mbed, e.g. one bugfix or one new feature. This makes the git history clean as each change (and the discussion around it) can be viewed individually. It also makes life easier for reviewers by reducing the amount of code to review per PR.

Note that once your PR is approved, it will all be squashed together into one commit named according to the PR title. So, unlike with ARM Mbed, there's no need to squash your commits together or manage their individual titles/descriptions!

If you need to add additional changes to the PR after submitting it, please push them as additional commits rather than amending or squashing to the originally submitted commits. This enables us to see what changes were pushed since the PR was previously reviewed.

### Licensing

Contributions must be licensed under an accepted open source license, and should contain a license header and SPDX identifier. See the [Mbed Source Code Licensing](For Mbed Developers/mbed-source-code-licensing.md) page for more details on how to do this.

## Testing your Changes

## Major Additions

Mbed CE is happy to accept most types of external contributions. However, we do need to make some considerations for managing the scope and size of Mbed OS. Once something is committed to the Mbed CE repo, it's on us maintainers to maintain it if it breaks, so we do set some limits on what can be contributed.

For that reason, there are some limits on the types of major additions that we accept:

| Addition | Example | Criteria for Acceptance | Reason | Notes |
|-------|------|----------|---------|-------|
|New Target| Adding a new board (and/or MCU) in an existing MCU target family | You must test Mbed on the board and confirm that it works. We'd prefer a Greentea test run (see above) if possible. Also, the board being added must be purchaseable by the general public. <br> Note however that we do NOT require boards to have an onboard debugger to be officially supported.| We, the Mbed CE maintainers, need to be able to purchase the board so that we can test code on it. Additionally, people other than you likely won't be able to get value out of your port if the board isn't purchaseable. | If your board does not meet this requirement, we recommend that you independently maintain a [custom target](https://github.com/mbed-ce/mbed-ce-custom-targets) instead. |
|New MCU Family| Adding a new chip family that contains its own HAL driver implementations. | Greentea tests must pass, and we, the Mbed CE maintainers, must purchase or be sent a board to test with. | We like to be able to run tests on at least one board from every MCU family when testing changes. | It is encouraged to submit a target with only the barebones HAL drivers (e.g. us ticker and serial) implemented first, then add additional drivers in subsequent PRs. |
|New CPU Architecture| Porting Mbed CE to RISC-V or Xtensa | This is something we'd love to look into, but it would be a ton of work as there's quite a lot of ARM-specific code in Mbed. |||
|New External Chip Driver| Adding a driver for a new type of wireless PHY or memory chip | Chip must be present on a supported target board and required for basic functionality (storage or networking), OR the driver should be generic enough to support an entire class of widely used chips (e.g. all SD cards or all SPIF flash devices). | We prefer for drivers to be maintained out of the Mbed source tree if possible, as this makes the base OS simpler and easier to maintain. The exception is for drivers which are useful for a wide class of situations or which are needed to use Mbed boards out of the box. | We have the [Libraries &amp; Examples](https://github.com/mbed-ce-libraries-examples) organization which we'd be happy to add your library to! |