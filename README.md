# Install
1. `brew install rbenv`
1. `rbenv init`
1. `curl -fsSL https://github.com/rbenv/rbenv-installer/raw/master/bin/rbenv-doctor | bash`
1. `rbenv install 2.6.3 && rbenv global 2.6.3`
1. `export PATH=$HOME/.gem/ruby/X.X.0/bin:$PATH`
1. `gem install bundler jekyll`
1. `bundle install`
1. `bundle exec jekyll serve`

# Dev server
`rbenv exec jekyll serve`
