# mywant-gui

MyWant GUI server — serves the web frontend and proxies API requests to the MyWant backend.

> This is the **public distribution** repository. It hosts the downloadable release
> binaries (see [Releases](../../releases)) and the [documentation](./docs). The
> application source code lives in a separate private repository.

## TL;DR

```sh
brew install onelittlenightmusic/mywant/mywant-gui
mywant-gui start          # serves the GUI, default http://localhost:8081
```

Then open <http://localhost:8081>. `mywant-gui` proxies API calls to the `mywant`
backend, which Homebrew installs automatically as a dependency.

## Installation

### Homebrew (macOS / Linux) — recommended

```sh
brew tap onelittlenightmusic/mywant
brew install mywant-gui
```

This also installs the `mywant` backend it depends on. To upgrade later:

```sh
brew update && brew upgrade mywant-gui
```

### Manual

Download the archive for your OS/arch from the
[latest release](../../releases/latest), extract it, and put the `mywant-gui`
binary somewhere on your `PATH`:

```sh
tar -xzf mywant-gui_*_darwin_arm64.tar.gz
sudo mv mywant-gui /usr/local/bin/
```

## Usage

```sh
mywant-gui start [-D]     # start (-D = background)
mywant-gui stop           # stop
mywant-gui get            # show server status
```

See the [CLI reference](./docs/CLI.md) for the full command set.

## Documentation

Full docs live in [`docs/`](./docs):

- [CLI reference](./docs/CLI.md)
- [GUI concept](./docs/GUIconcept.md)
- [Design principles](./docs/DesignPrinciple.md)
- [Want card design](./docs/WantCardDesign.md)
- [Want card plugin system](./docs/WantCardPluginSystem.md)
- [Global param pane](./docs/GlobalParamPane.md)
- [Web Inspector on iPhone](./docs/WebInspectorIPhone.md)
