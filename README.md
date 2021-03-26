# Tailwind CSS JIT + Rails without Webpacker

Me and Webpack(er) have never really clicked, so it has been exciting to see [DHH promote a modern approach with the traditional asset pipeline](https://github.com/hotwired/hotwire-rails-demo-chat). Me and Tailwind _have_  clicked however, and the [new just-in-time (JIT) compiler](https://blog.tailwindcss.com/just-in-time-the-next-generation-of-tailwind-css), looks very useful; but without Webpacker, how should it be integrated into a Rail project? [tailwindcss-rails](https://github.com/rails/tailwindcss-rails) looks promising, but I don't think it'll be possible to support the JIT compiler without getting stuck into Webpacker.

This project experiments with a vanilla Tailwind (JIT) build step to provide a CSS file for the Rails asset pipeline to consume. The following describes the setup.

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

Install Tailwind, the JIT compiler, PostCSS, and Autoprefixer:
```
npm install -D @tailwindcss/jit tailwindcss postcss postcss-cli autoprefixer
```

Setup Tailwind:
```
npx tailwindcss init
```

Create `postcss.config.js`and add the Tailwind JIT compiler as a PostCSS plugin:
```
module.exports = {
  plugins: {
    '@tailwindcss/jit': {},
    autoprefixer: {},
  }
}
```

In `app/assets/stylesheets/tailwind.css`:
```
@tailwind base;
@tailwind components;
@tailwind utilities;
```

## Build

Configure the files to scan for Tailwind class names. The Tailwind JIT compiler will use these to determine which CSS rules to generate. In `tailwind.config.js`:
```
…
purge: [
  './app/**/*.html.erb',
  './app/helpers/**/*.rb',
  './app/assets/javascripts/**/*.js'
],
…
```

Set up build scripts in `package.json` to generate a compiled `_tailwind.css` file:
```
…
"scripts": {
  "build": "postcss ./app/assets/stylesheets/tailwind.css -o ./app/assets/stylesheets/tailwind-build.css",
  "start": "postcss ./app/assets/stylesheets/tailwind.css -o ./app/assets/stylesheets/tailwind-build.css --watch",
},
…
```

Add the compiled `tailwind-build.css` to the `application.css` manifest:
```
/*
 *= require_tree .
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

Ignore the files we don't need to check in:
```
# .gitignore
…
/node_modules
npm-debug.log
/vendor/assets/stylesheets/_tailwind.css
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
On deploy, this will install our Node.js dependencies and then run our build script. It's important that this buildpack is added first so that `_tailwind.css` is compiled _before_ the assets are compiled by the Ruby buildpack.

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

Configure `postcss.config.js`:
```
module.exports = {
  plugins: {
    'postcss-easy-import': {},
    '@tailwindcss/jit': {},
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
