source "https://rubygems.org"

gem "jekyll", "~> 3.9.1"
gem "nokogiri", ">= 1.11.0"
gem "kramdown-parser-gfm"

group :jekyll_plugins do
  gem 'jekyll-sitemap'
  gem 'jekyll-feed'
  gem 'jekyll-seo-tag', "~> 2.7.1"
  gem 'jekyll-compose'
end

gem 'jekyll-gist'
gem 'jemoji'

# Windows does not include zoneinfo files, so bundle the tzinfo-data gem
install_if -> { RUBY_PLATFORM =~ %r!mingw|mswin|java! } do
  gem "tzinfo", "~> 1.2"
  gem "tzinfo-data"
end

# Performance-booster for watching directories on Windows
gem "wdm", "~> 0.1.1", :install_if => Gem.win_platform?