# Reforge

A Go CLI that keeps GitHub-released software up to date: Windows installer apps (.msi/.exe), and mods or
tools shipped as archives (.zip, .7z, .tar.gz). Commands: `add`, `list`, `update`, `remove`.
[docs/design.md](docs/design.md) is the design note; read it for anything not covered here.

## Layout

- `cmd/reforge`: the commands (`add.go`, `list.go`, `commands.go`), help and banner (`main.go`,
  `banner.go`), and terminal output, colours and the results table (`ui.go`)
- `internal/core`: all the rules (what to update, which file to pick, when to ask). `core.go` has the
  `Engine`, `Status` and `Result`; `installer.go` handles installer apps; `folders.go` handles archives
  extracted into folders (`--dest`); `download.go` handles archives kept in a folder; `verify.go`
  handles checksums, signatures and questions; `find.go`, `lookup.go`, `add.go` and `names.go` find apps
  on the PC and on GitHub
- `internal/system`: the only code that touches the OS (installed apps from the registry, running
  installers, signatures). `system_windows.go` is the real Windows code; `system_other.go` is for
  macOS and Linux
- `internal/github`: the GitHub API client and its cache; `internal/applist`: `apps.json` (field names
  match the old PowerShell module); `internal/version`: parses and compares versions (up to four numbers;
  `-beta` and other suffixes are ignored); `internal/pick`: chooses the release asset;
  `internal/archive`: extracts and deploys archives; `internal/wildcard`: name patterns

## How it works

- App types (`applist.App.Type`): `""` is an installer app, matched to Windows' Installed apps by
  `Name`; `"archive"` is extracted into `Destination` folders; `"download"` is an archive kept in `Folder`.
- Installer apps: the installed version comes from the registry (`system.Program`), so it can differ from
  the tag. After Reforge installs one, it saves the tag in `InstalledVersion` and the version Windows
  showed in `InstalledAs`, and compares tags while Windows still shows that version (issue #19: Nuvio's
  0.1.33-alpha shows as 1.1.33).
- MSI updates run `msiexec /i … /qn /norestart`, plus `ALLUSERS=1` for apps installed for all users (or
  they fail with 1603). All-users apps need admin rights, so without them the status is `NeedsAdmin`.
- `Engine.DryRun` (used by `list`) checks without downloading or asking anything.
- `Engine.Check` asks GitHub about all apps at once, and Installed apps are read once per run
  (`installedPrograms`).

## Build and test

```
go build ./cmd/reforge
gofmt -l .        # must print nothing
go vet ./...
go test ./...
```

- Tests in `internal/core` use the fakes in `fakes_test.go` (a fake GitHub, PC and asker). Add a test
  next to similar ones (`update_test.go`, `install_find_test.go`) for every behaviour change.
- CI (`.github/workflows/test.yml`) runs on Windows, Linux and macOS, with a race check on Linux.
- To update the user's own copy, build with the installed version number (`reforge --version`), or it
  shows "dev": `go build -trimpath -ldflags "-s -w -X main.version=<version>"` into
  `%LOCALAPPDATA%\Programs\Reforge\reforge.exe`.

## Conventions

- Match the existing style: short comments that say why, plain wording in messages, and notes that end
  without a full stop (they're joined with ". ").
- Git: for each change, make a short branch (`fix/…`, `feature/…`, `docs/…`), merge it into `main` with
  `--no-ff`, and push `main`. No Claude attribution in commits or PRs.
- Releases: the user publishes them on GitHub (tag `vX.Y.Z`); `publish.yml` builds and attaches the files.
  Never reuse a published version number. Batch changes into few, larger releases.
- Don't poll the GitHub API in loops from the user's PC: it shares their 60-requests-an-hour limit.
