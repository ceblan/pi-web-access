This directory defines the high-level concepts, business logic, and architecture of this project using markdown. It is managed by [lat.md](https://www.npmjs.com/package/lat.md) — a tool that anchors source code to these definitions. Install the `lat` command with `npm i -g lat.md` and run `lat --help`.

# Pi Web Access

Web Access provides web search and content tools while its package metadata controls host integration and update notifications.

## Pi 1.5 changelog notification

The package disables Pi 1.5's duplicate automatic changelog so its existing curated upgrade notice remains the only update notification.

`package.json` sets `pi.changelogPath` to the empty string. Pi 1.4 ignores the metadata, while `CHANGELOG.md` and the extension's curated upgrade notice remain unchanged. `test/package-typebox-dependency.test.mjs` verifies the setting in both the source manifest and the packed installation so publishing cannot silently restore the duplicate notification.
