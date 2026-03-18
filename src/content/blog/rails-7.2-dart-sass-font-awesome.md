---
title: 'Rails 7.2 + DartSass + Font Awesome'
description: 'Migrating Rails 7.2 to DartSass'
pubDate: '2024-09-17'
---

I spent the greater portion of my morning converting by Rails 7.1 application to use `dartsass-rails` instead of `sassc-rails`. Everything seemed to work as expected with a few small hiccups.

## Multiple CSS entry points

We host a fairly large Rails application divided into the multiple section. (Reasons for this belong in another article.) By default, `dartsass` will pull a single application.scss file. Adding the following allowed me to pick up the others:

```ruby
# config/initializers/dartsass.rb
Rails.application.config.dartsass.builds = {
  "admin.scss"       => "admin.css",
  "application.scss" => "application.css",
  "connect.scss"     => "connect.css",
  "signup.scss"      => "signup.css"
}
```

## Trix

Trix was being pulled in via sprockets using the CSS comment directive:

```css
/*
 * Provides a drop-in pointer for the default Trix stylesheet that will format the toolbar and
 * the trix-editor content (whether displayed or under editing). Feel free to incorporate this
 * inclusion directly in any other asset bundle and remove this file.
 *
 *= require trix
*/
```

Switching this to the following pulled in the required styles:

```css
@import "trix"
```

## Font Awesome

Font Awesome was imported into our application using the following:

```ruby
# Gemfile
source "https://token:00000000-0000-0000-0000-000000000000@dl.fontawesome.com/basic/fontawesome-pro/ruby/" do
  gem "font-awesome-pro-sass"
end
```

The styles were still bring brought in, however, the fonts were no longer present in the pipeline. Adding them to the vendor directory did not help since the asset fingerprinting was not supported in the gem. For development, the font files were added to the `public/webfonts/` directory. For production, we added them to our CDN origin server.

I'm not thrilled with this last step. Sure it works, but, yikes. Have a better way? Let me know.
