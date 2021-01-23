---
title: "How to compile Chef from source"
date: 2020-04-29
tags: [How-to]
---

`ARM` machines can be cheaper alternatives to `X86_64` instances either running
on bare metal (e.g. on Raspberyy Pi) or in cloud.

They are becoming more and more popular and they're making their way into the
consumer electronic.

Chef is a popular `infrastructure-as-a-code` tool, which you can install to
remotely or locally manage your servers.

In this post you can find how to build your favorite version of Chef for ARM.

> At the time of writing this article the most recent versions of Chef are
available to download for ARM but not for all Linux distributions, namely
Debian.

## Compiling Ruby

The
<a href="https://github.com/ruby/ruby#git" target="_blank">Ruby GitHub page</a>
is very helpful when it comes to compiling Ruby from source code.

> You can skip this step and go directly to [Compiling
Chef](#compiling-chef) if your distro already has a suitable ruby version
available. In my case, the version was too old for the Chef build to work.

The building process is as easy as:

```bash
./configure
make
```

### Installation in a destination directory

Now a `make install` will install Ruby in `/usr` directory, but that behavior
can be changed using `DESTDIR` option:

```bash
make install DESTDIR=~/ruby
```

The above command will install the build in `~/ruby`. This is useful if `root`
access is not available, or if the build is meant to be copied to a different
machine.

### Install Ruby globally

After building Ruby, you can install it globally by:

```bash
sudo cp -r ~/ruby/usr/* /usr
```

## Compiling Chef

You will need `Ruby` and `bundler` to be installed to build Chef. `gem` is also
needed which comes as a package with `Ruby`.

### Installing bundler

Make sure `bundler` is installed, if not install it:

```bash
gem install bundler
```

### Downloading the source

Download the specific version of Chef source from
<a href="https://github.com/chef/chef/" target="_blank">Chef on GitHub</a>
then extract the package.

```bash
wget https://github.com/chef/chef/archive/v15.6.10.tar.gz
tar xf v15.6.10.tar.gz
cd ~/chef-15.6.10/omnibus # yes, chef is built from the omnibus folder
```

### Installing dependencies

Then install the required dependencies for building chef locally:

```bash
bundle install --without development --path=.bundle
```

This will install all the dependencies in `.bundle` inside the source folder.

### Building using omnibus

After that start building Chef using `omnibus`:

```bash
bundle exec omnibus build chef -l internal
```

> <a href="https://github.com/chef/omnibus" target="_blank">Omnibus</a>
is a packaging solution that makes sure the packages and all its dependencies
are installed in a way that is easily manageable and will not conflict with the
existing packages installed on your system.

The result of the above build is a package specific to your OS, in this case,
I'm building Chef for Debian, so there will be a `.deb` file that I will able
to install and uninstall using `dpkg`.

```bash{outputLines: 2}
ls pkg/
chef_15.6.10*.deb
```

### Licensing issues

In my case the build failed because of a licensing error, you can try
ignoring licensing problems for the build, read more about that in
<a href="https://github.com/chef/omnibus/issues/696" target="_blank">issue #696</a>.

I fixed that by editing the `omnibus.rb` file (the omnibus config) and adding
these two lines:

```ruby:title=omnibus.rb
fatal_licensing_warnings false
fatal_transitive_dependency_licensing_warnings false
```

### Installation of the built deb package

Now we can easily install the package:

```bash
sudo dpkg -i pkg/chef_15.6.10*.deb
```

And verify the desired version is installed:

```bash{outputLines: 2}
chef-client -v
Chef Infra Client: 15.6.10
```
