---
title: 'Rails Tailwind Components'
description: 'Building reusable view components in FutureFund with Rails and TailwindCSS.'
pubDate: '2024-11-02'
---

In this guide, we're showing how we build components at [FutureFund](https://www.futurefund.com). We're building a simple yet flexible avatar component in Rails using TailwindCSS, where user initials are displayed within a circular badge. We're keeping it straightforward, implementing the component as a Rails helper without any additional libraries or complex abstractions. We've never found a need to go beyond Rails view helpers.

## Setting Up the Basics

To start, create a dedicated subdirectory in your helpers folder for your design system. At FutureFund, we're sticking to our bee theme and using `Honey`. You can name yours anything that suits your project. If you're looking for a simple, functional name, `Components` works just fine.

The initial version of our helper is straightforward:

```ruby
module Honey::AvatarHelper
  def avatar(initials)
    tag.div initials, class: "relative inline-flex items-center justify-center border rounded-full bg-slate-600 font-semibold text-white shrink-0 w-10 h-10 text-xs bg-slate-600 text-white"
  end
end
```

To use the component, simply call it in a view (assuming you use Slim templates):

```ruby
= avatar user.initials
```

## Adding Custom Classes

To give users control over styling, let's enable additional classes to be passed in. This allows customization while keeping the base structure intact.

```ruby
def avatar(initials, options = {})
  options.symbolize_keys!

  base_classes = %w[
    relative
    justify-center
    items-center
    inline-flex
    border
    rounded-full
    font-semibold
    shrink-0
    w-10
    h-10
    text-xs
    bg-slate-600
    text-white
  ]
  tag.div initials, class: [ base_classes, options[:class] ]
end
```

With a high number of classes, which is typical when using Tailwind, I find it easier to put one class per line as we see in `base_classes`.

With this setup, users can add custom styles:

```ruby
= avatar user.initials, class: "bg-sky-600"
```

## Adding Custom HTML Attributes

To add flexibility for additional HTML attributes, we'll modify our method to include html_options:

```ruby
def avatar(initials, options = {}, html_options = {})
  options.symbolize_keys!

  base_classes = %w[
    ...
  ]
  tag.div initials, class: [ base_classes, options[:class] ], **html_options
end
```

## Variants

Variants add even more flexibility by allowing users to choose specific design options without manually setting classes. In our example, we introduce size and color variants:

- Size options: `sm`, `md`, and `lg`
- Color options: `primary` and `secondary`

```ruby
def avatar(initials, options = {}, html_options = {})
  options.symbolize_keys!

  base_classes = %w[
    relative
    justify-center
    items-center
    inline-flex
    border
    rounded-full
    font-semibold
    shrink-0
  ]

  variants = {
    size: {
      sm: %w[ w-10 h-10 text-xs ],
      md: %w[ w-12 h-12 text-base ],
      lg: %w[ w-16 h-16 text-lg ]
    },
    color: {
      primary: %w[ bg-slate-600 text-white ],
      secondary: %w[ bg-sky-600 text-white ]
    },
    defaults: {
      size: :sm,
      color: :primary
    }
  }

  tag.div(
    initials,
    class: [ base_classes, extract_variants(variants, options), options[:class] ],
    **html_options
  )
end
```

And here's the `extract_variants` helper:

```ruby
def extract_variants(variants, options = {})
  extracted_values = []

  variants[:defaults].each_key do |variant_type|
    key = options[variant_type] || variants[:defaults][variant_type]
    extracted_values << variants.dig(variant_type, key.to_sym)
  end

  extracted_values
end
```

Now, you can use the avatar helper with variants:

```ruby
= avatar user.initials, size: "sm", color: "primary"
```

## Wrapping Up

This avatar component is a simple example how to build your design toolkit, providing options for custom styling and configuration that allow your team to match it seamlessly with the rest of your application's UI. By using variants, you can maintain consistency while still providing enough flexibility for customization across different contexts.
