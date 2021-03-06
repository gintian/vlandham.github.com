---
layout: post
title: Installing Galaxy Tools
categories:
  - bioinfo
---

If you followed along with [my galaxy server install guide](http://vallandingham.me/galaxy_install.html) you should have a basic but functional Galaxy server on your machine.

The next step is to get the tools that Galaxy depends on installed and working on this box. Galaxy’s wiki has an (incomplete) [listing of tool dependencies](http://wiki.g2.bx.psu.edu/Admin/Tools/Tool%20Dependencies?action=fullsearch&context=180&value=tool+dependency&titlesearch=Titles) . However, they don’t have a lot of information about exactly how Galaxy expects these dependencies to be installed / configured.

## Galaxy Documentation Deficiencies

From discussions on the mailing list, and poking around in the xml files in `galaxy-dist/tools`, it looks like the most straight forward way to satisfy these requirements is to have executables accessible in your galaxy user’s `PATH`.

However, with tools that are packaged as jars (like [picard](http://picard.sourceforge.net/) ), the jar files must be present in `galaxy-dist/tool-data/shared/jars` (or a sub-directory of this directory - depending on the package). This caveat is not mentioned anywhere that I can find. It is discussed a bit on the [Galaxy development forum](http://galaxy-development-list-archive.2308389.n4.nabble.com/How-and-where-to-install-tool-dependencies-td4179574.html) , but it really is unclear to me how people installing Galaxy are supposed to find out about these critical but obscure details.

Also, in the [Galaxy news from November 2010](http://wiki.g2.bx.psu.edu/News%20Briefs/2010_11_24?highlight=%28package%29%7C%28binary%29), there is mention of using an external `<tool_dependency_dir>` to hold `env.sh` scripts that will be sourced prior to running the command. However, there no documentation as to the best way to take advantage of these sourced files.

Because of the lack of documentation on how best to use this feature, I decided to take the simple route and get executables accessible from the `PATH` and symlink jar’s into their required directories.

## Bio.brew

I think [my fork version of bio.brew](https://github.com/vlandham/bio.brew) is one of the best ways to get started with the installation of these tools.

Bio.brew is a simple package manager, originally created by [David Rio Deiros](https://github.com/drio/bio.brew) with a focus on bioinformatic packages. It is similar to [homebrew](https://github.com/mxcl/homebrew) - but without many bells or whistles, and designed to work on both Mac and Linux.

[My branch of bio.brew](https://github.com/vlandham/bio.brew) simplifies some of the recipe creation process, and adds additional commands that I found useful (like ‘fake’ to pretend to install a prerequisite package). It also moves towards the idea of maintaining multiple versions of a particular tool, though currently this functionality is pretty basic.

So the idea is to get bio.brew setup, use it to install tools, and when necessary supplement the install with setup specific to Galaxy.

In the future, it might be nice to add Galaxy specific functionality to bio.brew to simplify the process even more.

## Bio.brew setup

I think the setup for bio.brew is pretty easy - largely due to it being just bash scripts. As the [readme](https://github.com/vlandham/bio.brew) states, simply clone bio.brew and add `BB_INSTALL` and `BB_PATH` variables to your environment and `PATH`.

Below are the commands I used to get bio.brew installed and configured. Tools will be installed in `/usr/local/galaxy/tools` (along with bio.brew itself).

{% highlight bash %}

# as galaxy user

cd /usr/local/galaxy
mkdir tools
cd tools
git clone git://github.com/vlandham/bio.brew.git
{% endhighlight %}

Then edit your `.bashrc` file to include:

{% highlight bash %}

# ...

export BB_INSTALL=~/tools
export BB_PATH=~/tools/bio.brew
PATH="$BB_INSTALL/bin:$BB_PATH/bin:\$PATH"
export PATH
{% endhighlight %}

This tells bio.brew to install packages into `~/tools` and adds the bin directory to your `PATH`.

## Bio.brew installing an executable

So, lets try out installing a required tool. [samtools](http://samtools.sourceforge.net/) is required by many Galaxy tools - so it provides a good test case for the bio.brew installation process. With bio.brew, you should be able to install samtools with two commands:

{% highlight bash %}
bb install samtools
bb activate samtools
{% endhighlight %}

`bb install` pulls down samtools and compiles it, getting everything ready. `bb activate` modifies the bio.brew symlinks used to point to the current version of a tool to point to this new version. The reason it is split up into two commands is to allow for testing of a tool prior to adding it to your path.

## Bio.brew installing jar’s

As mentioned above, we need to symlink jar files to a specific directory for Galaxy to know about them. Here we will install picard using bio.brew and add the .jar files to `galaxy-dist/tool-data/shared/jars/picard` so Galaxy knows about them. We will use the `fake` command to fake install java, as usually java is already installed on your machine and you don’t want to manage it through bio.brew.

{% highlight bash %}
bb fake java
bb install picard
bb activate picard

# now create symlinks

cd /usr/local/galaxy/galaxy-dist/tool-data/shared/jars
mkdir picard
cd picard
ln -s /usr/local/galaxy/tools/stage/picard/current/\*.jar ./
{% endhighlight %}

## Restart Galaxy and Test

Now its time to see if this actually worked. First restart Galaxy using the init script we setup before:

{% highlight bash %}
sudo /etc/init.d/galaxy restart
{% endhighlight %}

Now try out the NGS: Picard -\> FASTQ to BAM tool in your local Galaxy server on a test fastq file. If all goes well, you should see green all through your history. Enjoy!

## Limitations

bio.brew currently doesn’t have all the Galaxy prerequisites in it - but that is something that can be fixed easily. Simply create a new install recipe for the missing tool and contribute it back to bio.brew!
