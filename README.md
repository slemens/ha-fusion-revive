# Fusion-Revive

A modern, easy-to-use and performant fork of the original **ha-fusion** [Home Assistant](https://www.home-assistant.io/) dashboard.

I created this fork because ha-fusion remains one of the most elegant and powerful dashboards for Home Assistant. Since the original maintainer (matt8707) no longer has time to actively continue development, **Fusion-Revive** aims to keep the project alive, resolve issues, modernize areas of the codebase, and introduce improvements that many of us have been missing.

I am not a full-time developer — just a passionate Home Assistant user — but I try my best in my free time to add meaningful features and improvements for myself and for everyone else who enjoys Fusion-Revive.

<https://www.youtube.com/watch?v=D8mWruSuPOM>

[![preview](/static/preview.png)](https://www.youtube.com/watch?v=D8mWruSuPOM)

If you find this project useful, be sure to ⭐ this repository!

[![Buy Me A Coffee](https://img.buymeacoffee.com/button-api/?text=Buy%20me%20a%20coffee&emoji=&slug=slemens&button_colour=FFDD00&font_colour=000000&font_family=Inter&outline_colour=000000&coffee_colour=ffffff)](https://buymeacoffee.com/slemens)

---

## 📣 Pre-beta

The current state of this project is **pre-beta**. This means that some functionality may be incomplete, missing, or unstable. Feedback, bug reports and feature requests are very welcome.

---

## Installation

### Add-on

The original ha-fusion add-on is **not continued** in this fork. Since I personally use Docker exclusively, I cannot test or maintain an add-on version at this time.

If contributors are interested in helping revive and maintain the add-on, it may be added later.

---

### Docker

If you're using the "Container" or "Core" installation methods, Fusion-Revive can be installed via Docker.

1. **Docker Compose File**: Place your edited copy of the `docker-compose.yml` from this repository in a suitable directory.

2. **Create Container**:

```bash
cd path/to/docker-compose.yml
docker-compose up -d fusion-revive
```

#### Update

```bash
docker-compose pull fusion-revive
docker-compose up -d fusion-revive
```

<details>
<summary><b>Other</b></summary>

Without docker-compose, updating requires stopping and removing the current container, pulling the new image, and running the container again.

```bash
docker run -d \
  --name fusion-revive \
  --network bridge \
  -p 5050:5050 \
  -v /path/to/fusion-revive:/app/data \
  -e TZ=Europe/Berlin \
  -e HASS_URL=http://192.168.1.X:8123 \
  --restart always \
  ghcr.io/<YOUR_USER>/fusion-revive
```

#### Kubernetes

A Helm chart may be added in the future.

</details>

---

## Query Strings

These will only function if Fusion-Revive is exposed via a direct port (Docker). When using Ingress, query strings cannot be read.

### View

To set a specific view when the page loads:

```
?view=Bedroom
```

### Menu

To disable the menu button:

```
?menu=false
```

---

## Keyboard Shortcuts

| Key                 | Description |
| ------------------- | ----------- |
| **f**               | filter      |
| **esc**             | exit        |
| **cmd + s**         | save        |
| **cmd + z**         | undo        |
| **cmd + shift + z** | redo        |

---

## Debug

To debug errors:
- Docker users: `docker logs fusion-revive`
- Inspect frontend issues via the browser's developer console

---

## Develop

To contribute to Fusion-Revive, install Node.js and pnpm. If you're unfamiliar with Svelte, consider the tutorial at <https://learn.svelte.dev>.

```bash
# prerequisites (macos)
brew install node pnpm

# install
git clone https://github.com/slemens/ha-fusion-revive.git
cd ha-fusion-revive
pnpm install

# environment
cp .env.example .env
code .env

# server
npm run dev -- --open

# dependencies
pnpm outdated
pnpm update

# lint
npm run check
npm run lint
npm run format
```

---

## Acknowledgement

Massive thanks to **matt8707**, the creator of the original ha-fusion project. Fusion-Revive exists to keep his excellent work alive and usable.

If you enjoy this project, please consider giving it a ⭐ on GitHub!
