# Tailwind CSS JIT + Rails without Webpacker

**Update**: Rails 7 will support adding npm CSS-based packages via [rails/cssbundling-rails](https://github.com/rails/cssbundling-rails). It follows a similar approach to below. The **Deploy** part of this README is still pretty useful, but the rest will be outdated once Rails 7 is released.

## Install

Create a new Ruby on Rails app. We'll skip JavaScript and use PostgreSQL (for testing a deployment to Heroku):
```
rails new jitter -d postgresql --skip-javascript
cd jitter
rails db:create
```

Setup a `package.json`:
```
npm init -y
```

Install the latest versions of Tailwind, PostCSS, and Autoprefixer:
```
npm install -D tailwindcss@latest postcss@latest autoprefixer@latest
```

Setup Tailwind:
```
npx tailwindcss init
```

In `app/assets/stylesheets/tailwind.css`:
```
@tailwind base;
@tailwind components;
@tailwind utilities;
```
(This step isn't strictly necessary when using Tailwind CLI, but we'll include it here as it's pretty common to configure and add styles as your app grows.)

## Build

Configure the files to scan for Tailwind class names. The Tailwind JIT compiler will use these to determine which CSS rules to generate. In `tailwind.config.js`:
```
mode: 'jit',
purge: [
  './app/**/*.html.erb',
  './app/helpers/**/*.rb',
  './app/assets/javascripts/**/*.js'
],
```

Set up build scripts in `package.json` to generate a compiled `tailwind-build.css` file:
```
"scripts": {
  "build": "tailwindcss -i ./app/assets/stylesheets/tailwind.css -o ./app/assets/stylesheets/tailwind-build.css",
  "start": "tailwindcss -i ./app/assets/stylesheets/tailwind.css -o ./app/assets/stylesheets/tailwind-build.css --watch"
},
```

Add the compiled `tailwind-build.css` to the `application.css` manifest, and stub the `tailwind.css` manifest:
```
/*
 *= require_tree .
 *= stub tailwind
 *= require tailwind-build
 *= require_self
 */
```

## Try it out

Start the server and build watcher (in separate terminal tabs):
```
rails s
npm start
```

Set up a test view:
```
# config/routes.rb
root to: 'application#index'
```

```
# app/views/application/index.html.erb
<h1 class="text-red-600 text-4xl font-bold">Hello, World!</h1>
```

Make changes and try it out at `http://localhost:3000`

## Deploy

Rather than checking in the built `tailwind-build.css` file each time we make changes, we could make this part of a build process on deploy. Here's the steps for deploying to Heroku:

Ignore the files we don't need to check-in, in .gitignore:
```
/node_modules
npm-debug.log
/app/assets/stylesheets/tailwind-build.css
```

Commit:
```
git add . && git commit -m "Initial commit"
```

Create a Heroku app:
```
heroku create
```

Add the Node.js Heroku buildpack:
```
heroku buildpacks:add --index 1 heroku/nodejs
```
On deploy, this will install our Node.js dependencies and then run our build script. It's important that this buildpack is added first so that `tailwind-build.css` is compiled _before_ the assets are compiled by the Ruby buildpack.

Add the Ruby buildpack:
```
heroku buildpacks:add --index 2 heroku/ruby
```

Deploy:

```
git push heroku master
```

## `@apply`

I'd [avoid `@apply` where possible 😜](https://twitter.com/adamwathan/status/1308944904786268161), but if you have reasons…

As with any Tailwind project, you can add `@apply` before your `@tailwind utilities`. If this becomes unmanageable, and/or you need `@apply`directives in other files, use a PostCSS import plugin. The following uses [postcss-easy-import](https://github.com/TrySound/postcss-easy-import).

Install the plugin:
```
npm i -D postcss-easy-import
```

Create and configure `postcss.config.js`:
```
module.exports = {
  plugins: {
    'postcss-easy-import': {},
    tailwindcss: {},
    autoprefixer: {},
  }
}
```

Replace your `@tailwind` directives with `@import`s in `tailwind.css`:
```
@import 'tailwindcss/base';
@import 'tailwindcss/components';
@import 'tailwindcss/utilities';
```

Now you're ready to import, for example:
```
/* app/assets/stylesheets/cards.css */
.card {
  @apply m-8 p-8 text-center shadow-lg rounded-xl;
}
```

```
@import 'tailwindcss/base';
@import 'tailwindcss/components';
@import 'cards';
@import 'tailwindcss/utilities';
```
