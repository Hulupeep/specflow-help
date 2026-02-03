source "https://rubygems.org"

# Jekyll version for GitHub Pages compatibility
gem "jekyll", "~> 3.9.5"

# Just the Docs theme
gem "just-the-docs", "0.8.2"

# Required plugins
group :jekyll_plugins do
  gem "jekyll-seo-tag", "~> 2.8"
  gem "jekyll-github-metadata", "~> 2.16"
  gem "jekyll-include-cache", "~> 0.2"
end

# Markdown parser for GitHub Flavored Markdown
gem "kramdown-parser-gfm", "~> 1.1"

# Windows and JRuby does not include zoneinfo files
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

# Performance-booster for watching directories
gem "wdm", "~> 0.1", :platforms => [:mingw, :x64_mingw, :mswin]

# Lock to prevent Bundler from upgrading
gem "webrick", "~> 1.8"
