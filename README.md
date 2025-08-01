# ExVite


An example of a configuration to use `Vite` with `Phoenix LiveView`. [TODO] a mix task?


 __How?__ The documentation: <https://vite.dev/guide/backend-integration.html>


__Why?__ `Vite`does not bundle the code in development which means the dev server is fast to start, and your changes will be updated instantly. You can easily bring in plugins such as VitePWA with Workbox, or ZSTD compression, client-side SVG integration, React, Svelte, Solid... and [more](https://github.com/vitejs/awesome-vite#plugins). In production, `Vite` bundles the code, with tree-shaking...

__What?__ In DEV mode, you will be running a `Vite` dev server on port 5173 and `Phoenix` on port 4000. Changes in `.ex|.heex` files will be controlled by `Phoenix` code_reload whilst `.js |.css | .jsx..` changes will be controlled by `Vite`.


## Setup

- in "/", create `pnpm-workspace.yaml` (use "yaml", not "yml")
- go to "/assets", run `pnpm init` and set `"type": "module"`
- add/remove packages with `pnpm add -D xxx` or `pnpm remove xxx`
- go to "/" and run `pnpm install`.
- change the __app_name__ in "Vite.ex"


In DEV mode, `Phoenix` code reload will only listen to `ex|heex` files changes while `Vite` HMR will manage the `j(t)s(x)` files.

In PROD mode, `Vite` will build and bundle your assets if you structure your assets as shown above.

### Static assets

All your static assets should be organised in the "/assets" folder with the structure:

```
/assest/{js,css,seo, fonts, icons, images, wasm,...}
```

Do not add anything in the "/priv/static" folder as it will be pruned but instead in the "/assets" folder.

The __vite.config.js__ settings will populate the "/priv/static/asssets" folder with fingerprinted files, and copy the non-fingerprinted files into "/priv/static".

Note that this also means you do not need `mix phx.digest` anymore in the build stage.

For example, you have non-fingerprinted assets such as "robots.txt" and "sitemap.xml".
You place them in "/assets/seo" and these files will by copied in the "priv/static" folder and will be served by Phoenix.
For example, all your icons are placed in the folder "assets/icons" and will be copied in "priv/static/icons", and served by Phoenix.

The other fingerprinted static assets that are referenced in Elixir modules will use the helper `Vite.path/1`.

For example: in the __techs.ex__ module, we display a fingerprinted asset:

`<img src={Vite.path("images/my.svg"} alt="an-svg" loading="lazy" />`

These images are placed in the folder "assets/iamges" and are fingerprinted.

Static assets referenced in JS code will be referenced as "normal".


You let `Phoenix` serve the assets.

```elixir
def static_paths, do: ~w(
      assets
      icons
      robots.txt
      sw.js
      manifest.webmanifest
      sitemap.xml)
```

- [Phoenix dev.exs config](#phoenix-devexs-config)
- [Root layout](#root-layout)
- [Vite Config: server and build options](#vite-config-server-and-build-options)
  - [Build options](#build-options)
  - [Server options](#server-options)
  - [Run a separate Vite dev server in DEBUG mode](#run-a-separate-vite-dev-server-in-debug-mode)
- [Package.json](#packagejson)
  - [Using workspace with pnpm](#using-workspace-with-pnpm)
  - [without workspace](#without-workspace)
- [Tailwind, daisyui and heroicons](#tailwind-daisyui-and-heroicons)
- [An Elixir file path resolving module](#an-elixir-file-path-resolving-module)
- [Vite.config.js](#viteconfigjs)
- [Dockerfile](#dockerfile)



## Phoenix dev.exs config


Define a config "env" variable:

```elixir
# config.exs

config :my_app, :env, config_env()
```

Let `Phoenix` only  watch  `ex|heex` files, and run the `Vite` dev server (ie ignore `js`, `svg`,..., everything else)

```elixir
# dev.exs

config :my_app, MyAppWeb.Endpoint,
  http: [ip: {127, 0, 0, 1}, port: 4000],
  [...],
  code_reloader: true,
  live_reload: [
    web_console_logger: true,
    patterns: [
      ~r"lib/my_app_web/(controllers|live|components|router)/.*\.(ex|heex)$",
      ~r"lib/my_app_web/.*/.*\.heex$"
    ]
  ],
  watchers: [
    pnpm: [
      "vite",
      "serve",
      "--mode",
      "development",
      "--config",
      "vite.config.js",
      cd: Path.expand("../assets", __DIR__)
    ]
  ]
```

## Root layout

Pass the assign `@env` in the LiveView (or controller):

```elixir
|> assign(:env, Application.fetch_env!(:my_app, :env))
```


Add the following to "root.html.heex":

```elixir
# root.html.heex

<link
 :if={@env === :prod}
 rel="stylesheet"
 href={Vite.path("css/app.css")}
/>

<script 
  :if={@env === :dev}
  type="module"
  src="http://localhost:5173/@vite/client"
>
</script>

<script
 defer
 type="module"
 src={Vite.path("js/app.js")}
>
</script>
```

When you run the app, you can inspect the "network" tab and should get (at least) the two WebSocket connections: 

```
ws://localhost:4000/phoenix/live_reload/socket/websocket?vsn=2.0.0
ws://localhost:5173/?token=yFGCVgkhJxQg
```
 and

```
app.css -> http://localhost:5173/css/app.css
app.js  -> http://localhost:5173/js/app.js
```

## Vite Config, server and build options

### Build options

```js
const staticDir = "../priv/static";

const buildOps = (mode) => ({
  target: ["esnext"],
  // the directory to nest generated assets under (relative to build.outDir)
  outDir: staticDir,
  cssMinify: mode === 'production' && "lightningcss", // Use lightningcss for CSS minification
  rollupOptions: {
    input: mode == "production" ? getEntryPoints() : ["js/app.js"],
    output: mode === "production" && {
      assetFileNames: "assets/[name]-[hash][extname]",
      chunkFileNames: "assets/[name]-[hash].js",
      entryFileNames: "assets/[name]-[hash].js",
    },
  },
  // generate a manifest file that contains a mapping
  // of non-hashed asset filenames in PROD mode
  manifest: mode === "production",
  path: ".vite/manifest.json",
  minify: mode === "production",
  emptyOutDir: true, // Remove old assets
  sourcemap: mode === "development" ? "inline" : true,
  reportCompressedSize: true,
  assetsInlineLimit: 0,
});
```

The `getEntryPoints()` function is detailed in the __vite.config.js__ part. It is a list of all your files that will be fingerprinted.

The other static assets should by copied with the plugin `viteStaticCopy` to which we pass a list of objects (source, destination). These are eg SEO files (robots.txt, sitemap.xml), and your icons, fonts ...

### Server options

> Ignore `.ex|.heex` files changes.

```js
// vite.config.js

const devServer = {
  cors: { origin: "http://localhost:4000" },
  allowedHosts: ["localhost"],
  strictPort: true,
  origin: "http://localhost:5173", // Vite dev server origin
  port: 5173, // Vite dev server port
  host: "localhost", // Vite dev server host
  watch: {
    ignored: ["**/priv/static/**", "**/lib/**", "**/*.ex", "**/*.exs"],
  },
};
```

The __vite.config.js__ module will export:

```js
export default defineConfig = ({command, mode}) => {
  if (command == 'serve') {
    process.stdin.on('close', () => process.exit(0));
    process.stdin.resume();
  }

  return {
     server: mode === 'development' && devServer,
     build: buildOps(mode),
     publicDir: false,
     plugins: [tailwindcss(), viteCopy],
     [...]
  }
})
```

### Run a separate Vite dev server in DEBUG mode

You can also run the dev server in a separate terminal in DEBUG mode.

In this case, remove the watcher above, and run:

```sh
DEBUG=vite:* pnpm vite serve
```

The `DEBUG=vite:*` option gives extra informations that can be useful even if it may seem verbose.

## Package.json 

### Using workspace with pnpm

You can use `pnpm` with workspaces. In the root folder, define a "pnpm-workspace.yaml" file (❗️not "yml") and reference your "assets" folder and the "deps" folder (for `Phoenix.js`):

```yaml
# /pnpm-workspace.yaml

packages:
  - assets
  - deps/phoenix
  - deps/phoenix_html
  - deps/phoenix_live_view
```

In the "assets" folder, run:

```
/assets> pnpm init
```

and populate your newly created package.json with your favourite client dependencies:

```sh
/assets> pnpm add -D tailwindcss @tailwindcss/vite daisyui vite-plugin-static-copy fast-glob lightningcss
```

▶️ Set "type": "module" and use "workspace":

```json
# /assets/package.json

{
  "type": "module",
  "dependencies": {
    "fast-glob": "^3.3.3",
    "phoenix": "workspace:*",
    "phoenix_html": "workspace:*",
    "phoenix_live_view": "workspace:*",
    "topbar": "^3.0.0"
  },
  "devDependencies": {
    "@tailwindcss/vite": "^4.1.11",
    "daisyui": "^5.0.43",
    "tailwindcss": "^4.1.11",
    "vite": "^7.0.0",
    "vite-plugin-static-copy": "^2.3.1",
    "fast-glob": "^3.3.3",
    "lightningcss": "^1.30.1"
  }
}
```

In the root folder, install everything with:

```
/> pnpm install
```

### without workspace

Alternatively, if you don't use workspace, then reference directly the relative location for the Phoenix dependencies.

```json
{
  "type": "module",
  "dependencies": {
    "phoenix": "file:../deps/phoenix",
    "phoenix_html": "file:../deps/phoenix_html",
    "phoenix_live_view": "file:../deps/phoenix_live_view",
  }
  ...
}
```

## Tailwind, daisyui and heroicons

▶️ __cf `Phoenix 1.8`__: disable automatic source detection and instead specify sources explicitely.

```css
# /assets/css/app.css

@import 'tailwindcss' source(none);

@source "../**/.*{js, jsx}";
@source "../../lib/my_app_web/";

@plugin "daisyui";

@plugin "../vendor/heroicons.js";
```

where the "assets/vendor/heroicons.js" file is (from `phoenix` 1.8):

-------- heroicons.js --------

```js
// /assets/vendor/heroicons.js

const plugin = require("tailwindcss/plugin");
const fs = require("fs");
const path = require("path");

module.exports = plugin(function ({ matchComponents, theme }) {
  const iconsDir = path.join(__dirname, "../../deps/heroicons/optimized");
  const values = {};
  const icons = [
    ["", "/24/outline"],
    ["-solid", "/24/solid"],
    ["-mini", "/20/solid"],
    ["-micro", "/16/solid"],
  ];
  icons.forEach(([suffix, dir]) => {
    fs.readdirSync(path.join(iconsDir, dir)).forEach((file) => {
      const name = path.basename(file, ".svg") + suffix;
      values[name] = { name, fullPath: path.join(iconsDir, dir, file) };
    });
  });
  matchComponents(
    {
      hero: ({ name, fullPath }) => {
        let content = fs
          .readFileSync(fullPath)
          .toString()
          .replace(/\r?\n|\r/g, "");
        content = encodeURIComponent(content);
        let size = theme("spacing.6");
        if (name.endsWith("-mini")) {
          size = theme("spacing.5");
        } else if (name.endsWith("-micro")) {
          size = theme("spacing.4");
        }
        return {
          [`--hero-${name}`]: `url('data:image/svg+xml;utf8,${content}')`,
          "-webkit-mask": `var(--hero-${name})`,
          mask: `var(--hero-${name})`,
          "mask-repeat": "no-repeat",
          "background-color": "currentColor",
          "vertical-align": "middle",
          display: "inline-block",
          width: size,
          height: size,
        };
      },
    },
    { values }
  );
});
```

## An Elixir file path resolving module

This is needed to resolve the file path in dev or in prod mode.

❗️You need to change your application name in this module


-------- Vite.ex --------

```elixir
# lib/my_app_web/vite.ex

if Application.compile_env!(:my_app, :env) == :dev do
  defmodule Vite do
    @moduledoc """
    A helper module to manage Vite file discovery.

    It appends "http://localhost:5173" in DEV mode.

    It finds the fingerprinted name in PROD mode from the .vite/manifest.json file that Vite generates.
    """

    def path(asset) do
      "http://localhost:5173/#{asset}"
    end
  end
else
  defmodule Vite do
    # Ensure the manifest is loaded at compile time in production
    def path(asset) do
      app_name = :my_app
                  ^^^
      manifest = get_manifest(app_name)

      case Path.extname(asset) do
        ".css" ->
          get_main_css_in(manifest)

        _ ->
          get_name_in(manifest, asset)
      end
    end

    defp get_manifest(app_name) do
      manifest_path = Path.join(:code.priv_dir(app_name), "static/.vite/manifest.json")

      with {:ok, content} <- File.read(manifest_path),
           {:ok, decoded} <- Jason.decode(content) do
        decoded
      else
        _ -> raise "Could not read or decode Vite manifest at #{manifest_path}"
      end
    end

    defp get_main_css_in(manifest) do
      manifest
      |> Enum.flat_map(fn {_key, entry} ->
        Map.get(entry, "css", [])
      end)
      |> Enum.filter(fn file -> String.contains?(file, "app") end)
      |> List.first()
    end

    defp get_name_in(manifest, asset) do
      case manifest[asset] do
        %{"file" => file} -> "/#{file}"
        _ -> raise "Asset #{asset} not found in manifest"
      end
    end
  end
end
```

## Vite.config.js

Your "vite.config.js" file is placed in the "assets" folder.

You locate all your assets in this "assets" folder, with a structure like:

`assets/{js, css, icons, images, wasm, fonts, seo}`

This `Vite` file will copy and build the necessary files for you (given the structure above):


-------- vite.config.js --------

```js
// /assets/vite.config.js

import { defineConfig } from "vite";
import fs from "fs"; // for file system operations
import path from "path";
import fg from "fast-glob"; // for recursive file scanning
import tailwindcss from "@tailwindcss/vite";
import { viteStaticCopy } from "vite-plugin-static-copy";

const rootDir = path.resolve(import.meta.dirname);
const cssDir = path.resolve(rootDir, "css");
const jsDir = path.resolve(rootDir, "js");
const seoDir = path.resolve(rootDir, "seo");
const iconsDir = path.resolve(rootDir, "icons");
const srcImgDir = path.resolve(rootDir, "images");
const staticDir = path.resolve(rootDir, "../priv/static");

/* 
PROD mode: list of fingerprinted files to pass to RollUp(/Down)
*/
const getEntryPoints = () => {
  const entries = [];
  fg.sync([`${jsDir}/**/*.{js,jsx,ts,tsx}`]).forEach((file) => {
    if (/\.(js|jsx|ts|tsx)$/.test(file)) {
      entries.push(path.resolve(rootDir, file));
    }
  });

  fg.sync([`${srcImgDir}/**/*.*`]).forEach((file) => {
    if (/\.(jpg|png|svg|webp)$/.test(file)) {
      entries.push(path.resolve(rootDir, file));
    }
  });

  return entries;
};


const buildOps = (mode) => ({
  target: ["esnext"],
  outDir: staticDir,
  cssMinify: mode == 'production' && 'lightningcss', // already shipped by Tailwind v4?
  rollupOptions: {
    input:
      mode == "production" ? getEntryPoints() : ["js/app.js"],
    // hash only in production mode
    output: mode === "production" && {
      assetFileNames: "assets/[name]-[hash][extname]",
      chunkFileNames: "assets/[name]-[hash].js",
      entryFileNames: "assets/[name]-[hash].js",
    },
  },
  manifest: mode === 'production',
  path: ".vite/manifest.json",
  minify: mode === "production",
  emptyOutDir: true, // Remove old assets
  sourcemap: mode === "development" ? "inline" : true,
});


/* 
Static assets served by Phoenix via the plugin `viteStaticCopy`
=> add other folders like assets/fonts...if needed
*/
const getBuildTargets = () => {
  const baseTargets = [];

  // Only add targets if source directories exist
  if (fs.existsSync(seoDir)) {
    baseTargets.push({
      src: path.resolve(seoDir, "**", "*"),
      dest: path.resolve(staticDir),
    });
  }

  if (fs.existsSync(iconsDir)) {
    baseTargets.push({
      src: path.resolve(iconsDir, "**", "*"),
      dest: path.resolve(staticDir, "icons"),
    });
  }

  const devManifestPath = path.resolve(staticDir, "manifest.webmanifest");
  if (fs.existsSync(devManifestPath)) {
    fs.writeFileSync(devManifestPath, JSON.stringify(manifestOpts, null, 2));
  };
  return baseTargets;
}; 

const resolveConfig = {
  alias: {
    "@": rootDir,
    "@js": jsDir,
    "@jsx": jsDir,
    "@css": cssDir,
    "@static": staticDir,
    "@assets": srcImgDir,
  },
  extensions: [".js", ".jsx", "png", ".css", "webp", "jpg", "svg"],
};

const devServer = {
  cors: { origin: "http://localhost:4000" },
  allowedHosts: ["localhost"],
  strictPort: true,
  origin: "http://localhost:5173", // Vite dev server origin
  port: 5173, // Vite dev server port
  host: "localhost", // Vite dev server host
  watch: {
    ignored: ["**/priv/static/**", "**/lib/**", "**/*.ex", "**/*.exs"],
  },
};

export default defineConfig(({ command, mode }) => {
  if (command == "serve") {
    console.log("[vite.config] Running in development mode");
    process.stdin.on("close", () => process.exit(0));
    process.stdin.resume();
  }

  return {
    base: "/",
    plugins: [
      viteStaticCopy({ targets: getBuildTargets() }),
      tailwindcss(),
    ],
    resolve: resolveConfig,
    // Disable default public dir (using Phoenix's)
    publicDir: false,
    build: buildOps(mode),
    server: mode === "development" && devServer,
  };
});
```


Notice that we took advantage of the resolver "@". In your code, you can do:

```js
import { myHook} from "@js/hooks/myHook";
```

## Dockerfile

To build for production, you will run `pnpm vite build`.
In the Dockerfile below, we use the "workspace" version:

-------- Dockerfile --------

```dockerfile
# Stage 1: Build
ARG ELIXIR_VERSION=1.18.3
ARG OTP_VERSION=27.3.4
ARG DEBIAN_VERSION=bullseye-20250428-slim
ARG pnpm_VERSION=10.12.4


ARG BUILDER_IMAGE="hexpm/elixir:${ELIXIR_VERSION}-erlang-${OTP_VERSION}-debian-${DEBIAN_VERSION}"
ARG RUNNER_IMAGE="debian:${DEBIAN_VERSION}"

ARG MIX_ENV=prod
ARG NODE_ENV=production

FROM ${BUILDER_IMAGE} AS builder


RUN apt-get update -y && apt-get install -y \
  build-essential  git curl && \
  curl -sL https://deb.nodesource.com/setup_22.x | bash - && \
  apt-get install -y nodejs && \
  apt-get clean && rm -f /var/lib/apt/lists/*_*

ARG MIX_ENV
ARG NODE_ENV
ENV MIX_ENV=${MIX_ENV}
ENV NODE_ENV=${NODE_ENV}

# Install pnpm
RUN corepack enable && corepack prepare pnpm@${pnpm_VERSION} --activate

# Prepare build dir
WORKDIR /app

# Install Elixir deps
RUN mix local.hex --force && mix local.rebar --force

COPY mix.exs mix.lock pnpm-lock.yaml pnpm-workspace.yaml ./
RUN mix deps.get --only ${MIX_ENV}
RUN mkdir config

# compile Elxirr deps
COPY config/config.exs config/${MIX_ENV}.exs config/
RUN mix deps.compile

# compile Node deps
WORKDIR /app/assets
COPY assets/package.json  ./
WORKDIR /app
RUN pnpm install --frozen-lockfile

# Copy app server code before building the assets
# since the server code may contain Tailwind code.
COPY lib lib

# Copy, install & build assets--------
COPY priv priv

#  this will copy the assets/.env for the Maptiler api key loaded by Vite.loadenv
WORKDIR /app/assets
COPY assets ./ 
RUN pnpm vite build --mode ${NODE_ENV} --config vite.config.js

WORKDIR /app
# RUN mix phx.digest <-- used Vite to fingerprint assets instead
RUN mix compile

COPY config/runtime.exs config/

# Build the release-------
COPY rel rel
RUN mix release

# Stage 2: Runtime --------------------------------------------
FROM ${RUNNER_IMAGE}

RUN apt-get update -y && \
  apt-get install -y libstdc++6 openssl libncurses5 locales ca-certificates \
  && apt-get clean && rm -rf /var/lib/apt/lists/*

ENV MIX_ENV=prod

RUN sed -i '/en_US.UTF-8/s/^# //g' /etc/locale.gen && locale-gen
ENV LANG=en_US.UTF-8 LANGUAGE=en_US:en LC_ALL=en_US.UTF-8

WORKDIR /app

COPY --from=builder --chown=nobody:root /app/_build/${MIX_ENV}/rel/liveview_pwa ./

# <-- needed for local testing
RUN chown -R nobody:nogroup /mnt
RUN mkdir -p /app/db && \
  chown -R nobody:nogroup /app/db && \
  chmod -R 777 /app/db && \
  chown nobody /app

USER nobody

EXPOSE 4000
CMD ["/bin/sh", "-c", "/app/bin/server"]
```
