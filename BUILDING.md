## Building this site

First thing is to setup.

```script
sudo apt-get install ruby-full build-essential zlib1g-dev
```

```
echo '# Install Ruby Gems to ~/gems' >> ~/.profile
echo 'export GEM_HOME="$HOME/gems"' >> ~/.profile
echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.profile
source ~/.profile
```

```
gem install jekyll bundler
```

### Update Dependencies

```shell
bundle update
```

### Refresh the build caches

```shell
bundle exec jekyll clean
```

