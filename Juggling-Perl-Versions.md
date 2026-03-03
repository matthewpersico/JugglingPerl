Jan03: Juggling Perl Versions

# Juggling Perl Versions

_The Perl Journal_ January 2003

### By Matthew O. Persico

_This article was published in The Perl Journal in January of 2003. The first releases of [plenv]() and [perlbrew]() appear to be in 2013 and 2010 respectively. Keep that in mind as you read._

- - -

Managing change when you maintain a sizable Perl installation isn't easy. Inevitably, one of your colleagues will come to you and say, "I was looking at CPAN and you know, I could save myself two weeks of work if I could install the Foo::Bar module for use in my Yadayadayada project. Can we get it installed?"

"Sure," you reply. "I'll get right on it." You download the tarball from CPAN, execute:

> gtar zxvf Foo-Bar-1.68.tar.gz
>
> cd Foo-Bar-1.68
>
> perl Makefile.PL

and then it hits you:

Perl v5.8.0 required—this is only v5.6.1, stopped at Bar.pm line 2.

Okay, now what? Well, you can try to find a version of Foo::Bar that works with 5.6.1, but the odds are there isn't one. Even if there is such a version, it is probably old enough that it is missing needed functionality or has bugs that are fixed in later versions.

You could say, "Sorry, we can't support it," and you might even get away with that for just one module or just one project.

Eventually, however, the modules you support are going to be upgraded, new modules will be required, and your version of Perl is not going to support them.

That's the point where you start to consider an upgrade of Perl.

However, by this time, you've discovered that there are hundreds of Perl scripts and modules used in production. Every one of them will have to be regression tested before upgrading. Regression testing takes time away from development. Try selling _that_ to your end users.

The cost of upgrading any programming language environment is not trivial. You must make sure that every program works in the new environment in the same manner as it did in the old environment before you switch over to the new one. In addition, as more and more operating systems (at least in the \*NIX and Linux worlds) come with default installations of Perl that are critical to the smooth operation of the system, upgrading Perl or its default modules can negatively impact the operating system's performance.

It seems nearly impossible to safely upgrade Perl without a massive amount of testing. But there is a solution. Instead of upgrading Perl, you can "side-grade" it. This article will describe how to install and maintain multiple versions of Perl on a machine in a transparent and flexible manner. In this manner, you can port scripts on an as-needed basis and maintain multiple versions for testing on your development machines.

### Principles

In order to maintain multiple versions of Perl in an environment, we set down the following principles:

*   **Do no harm.** A user or process that knows nothing about our  Perl-swapping environment should be presented with the default Perl environment that comes "out-of-the-box." This means that all paths should have default values that allow a process to use Perl without having to explicitly call any special functions or explicitly set any special environment variables.
*   **No pain, all gain.** The method for changing Perl versions should be as painless as possible. Ideally, one command should do it.
*   **You can go back.** There is no reason why the default version of Perl shouldn't be considered just another version of Perl to be swapped. You should be able to switch back to the default version of Perl as easily as to any other version.

To this end, we will create a shell script called "perl.sh," to be placed in the /etc/profile.d directory and called from /etc/profile upon login. It will create all functions needed to set a particular version of Perl in the environment, and it will set the default environment when first invoked.

I'll describe how to build perl.sh step by step. This is the easiest way to show you how to build the environment yourself, which will probably be necessary—your Perl may be in a different place, your modules may be scattered over multiple directories, and you may have a different version of Perl than I am describing here. Use this article and its code as a template—do not expect to drop this version of perl.sh onto your system without alteration.

### Standardizing the Default Perl Version

The first step is to get the default version of Perl properly set up using perl.sh. This will take a bit of work because the default version of Perl is already set up, so we do not want to clobber it when perl.sh is invoked the first time around. However, we must make sure that we can get back to the default version from another installed version if needed.

### Determine Locations

We need to get the locations of the Perl binary, modules, and manpages. One could use _perldoc_ instead of manpages, but having the Perl manpages available is convenient if you are using GUI _man_ tools.

First, execute the command _which perl_. In this case, the result is /usr/bin/perl, which tells us two things:

1\. Where the Perl binary lives.

2\. It is on a directory in the PATH.

Next, execute _perl -V_. We ignore the build information for the moment and focus on the tail end of the output, which is shown in <a name="re1"></a>[Example 1](Juggling-Perl-Versions-ex1.md).

What this tells us is that there are no other known locations for Perl modules other than the default locations. Practically, that means that PERL5LIB is not set and that all modules added to the system must be installed into /usr/lib/perl5.

My personal philosophy on the subject of installation locations is simple: If I didn't build it, I won't write to it. I install all modules added to the system into another directory, even those that are upgrades of modules that are part of the out-of-the-box installation. Upgrading the default version of Perl or its modules in place could cause operating-system components to fail (as previously discussed). Furthermore, if you need to rebuild your system, you will have yet another item to attend to—rebuilding all the modules you added to or updated in the default location. If these modules are installed elsewhere, then when you restore the same version of Perl under which the modules were originally installed to the default location, the add-on modules should be immediately available (assuming you unmounted and did not reformat that partition in the rebuild). Finally, one can, if necessary, cut off access to the upgraded versions by removing the directory from PERL5LIB and allow usage of the default modules since they still exist in the original Perl tree.

I am going to establish /opt/perl/lib as the location for my new modules. I am doing this because I consider Perl to be a product unto itself. /usr and /usr/local are good places to stash shared objects like libraries or executables. Perl, however, has a pretty intricate directory structure of its own. Isolating the structure makes it easier to do our environment modifications to swap versions later.

The exact setting of PERL5LIB that is needed to produce a valid @INC seems to have changed from release 5.004\_04 through 5.005, 5.6.0, 5.6.1, and 5.8.0. Furthermore, the end result is highly dependent on where and how Perl is configured and installed in the first place. Rather than try to determine the settings via the documentation, I suggest you determine it empirically with these steps:

1\. Create /opt/perl.

2\. Grab any module from CPAN.

3\. Unwind the tarball and perform the magic incantations:

perl Makefile.PL
make test install PREFIX=/opt/perl

4\. Take note of where the module is installed.

You'll notice that I set PREFIX at installation time. This is a result of the environment in which I was trained. By setting PREFIX at install time instead of Makefile at creation time, I can use the same tarball tree to install to a local testing area, a global testing area, and a production area without having to reexecute _perl Makefile.PL_ each time. You may feel differently. Feel free to specify PREFIX at _perl Makefile.PL_ time. The only caveat is that you must be consistent. Pick one method and stick with it. I vaguely recall that in one version of Perl, the end location of the module was different depending on which method was used. This is another example of why I am demonstrating the empirical method of building this framework.

Finally, test _man_ capabilities. When I try _man perl_, the manpage appears. This indicates that we can get to the manpages in Section 1 for Perl commands. When I try _man ExtUtils::MakeMaker_, the manpage does not appear. A quick check of the environment reveals that there is no MANPATH environment variable set. Therefore, _man_ is using its own heuristics to determine where manpages are located. By reexamining the top of the _perl -V_ output, we see the information in <a name="re2"></a>[Example 2](Juggling-Perl-Versions-ex2.md).

The flag _\-Dman3dir_ defines a location for module manpages that is not going to appear in a default manpath due to its location, which is not "near" a PATH entry. Even if this path did appear in the default manpath, it would not help us. Without a MANPATH variable to manipulate, we are not going to be able to swap in manpages that correspond to the version of Perl that we want to change. So, the first order of business is to fix this.

In most versions of _man_, setting MANPATH completely overrides _man_'s heuristics; it does not add new paths to the default ones. This is not a problem, however. There is usually a program _manpath_ that will print the default paths used by _man_ in the absence of a MANPATH variable. So, we create man.sh and put it in /etc/profile.d. This ensures that the variable is set for each login process. The contents of man.sh are simply:

MANPATH='manpath'
export MANPATH

This is strict Bourne Shell syntax. Korn or bash users can try:

export MANPATH=$(manpath)

I will use bash shell syntax from now on. Bourne Shell users can make the necessary adjustments.

Now we will be able to alter the _man_ environment to match our Perl environment. It might be argued that setting MANPATH to override _man_'s heuristics defeats the purpose of having them. In truth, the only time it matters is when installing new software. Without a MANAPTH set, _man_ will automatically find the new _man_ directory if it is "close" to a directory on the PATH. But a new PATH entry is only available to processes that log in after installation, unless a process (or more likely a user) resets PATH. Logging out and in again should rerun the /etc/profile.d files, including our new manpath.sh, which will reexecute the _manpath_ call and pick up any newly installed manpages if they follow the heuristics.

Create the set and _unset_ Functions

We will create a function named set\_perl\_5.6.1. It will perform three functions:

1\. Set PATH so that the 5.6.1 version of Perl is on the path.

2\. Set PERL5LIB so that any extra libraries are found.

3\. Set MANPATH so that the Perl manpages are located.

Here is the code:

set\_perl\_5.6.1()
{
  export PERL\_CURRENT\_VERSION=5.6.1
  addpath -f PATH /opt/perl/bin
  add\_perl5lib /opt/perl
  addpath -f MANPATH /usr/lib/perl5/man
  addpath -f MANPATH /opt/perl/lib/perl5/man
}

Take a look at the line _addpath -f MANPATH /usr/_lib/perl5/man. Given that this is the default version of Perl, you might be tempted to add it to /etc/man.config. It is even possible that the line is already in man.config, waiting to be uncommented. Do not do it. We need to get rid of this part of the path when swapping in a new version; making it part of the default will prevent its removal.

PERL\_CURRENT\_VERSION is defined in order to keep track of what version we are currently running. _addpath_ is a function that will add a directory to a PATH-like environment variable. It makes use of the PERL\_DEFAULT variable. <a name="rl1"></a>See [Listing 1](#listing-1) for the code. _addpath_ and _delpath_ are translations of shell scripts and are described in the following _Linux Journal_ articles: http://www .linuxjournal.com/article.php?sid=3645, http://www.linuxjournal .com/article.php?sid=3768, and http://www.linuxjournal.com/ article.php?sid=3935.

In this case, we add /opt/perl/bin to PATH because there are modules (such as DBI) that create binaries. This is where they will end up when installed. Why not /opt/perl/bin/5.6.1 or /opt/ perl/5.6.1/bin? Because when we execute _make install PREFIX=/opt/perl_, the remainder of the paths are taken from Perl's current configuration, and the configuration, in this case, is to put all binaries into $PREFIX/bin. _add\__perl5lib is another function, presented in <a name="re3"></a>[Example 3](Juggling-Perl-Versions-ex3.md) with its relatives.

We find the directories corresponding to the current version of Perl under the requested top-level directory and add or delete them from the PERL5LIB environment variable.

The corresponding _unset_ function is:

unset\_perl\_5.6.1()
{
  delpath -f MANPATH /opt/perl/lib/perl5/man
  delpath -f MANPATH /usr/lib/perl5/man
  del\_perl5lib /opt/perl
  delpath -f PATH /opt/perl/bin
  unset PERL\_CURRENT\_VERSION
}

_delpath_ is a function that will add a directory to a PATH-like environment variable; see <a name="rl2">[Listing 2](#listing-2). Detailed discussion of the function is beyond the scope of this article.

You will notice that the actions in the _unset_ function are performed in inverse order of the corresponding _set_ actions. Although this is not strictly required, it is imperative that the setting and unsetting of PERL\_CURRENT\_VERSION be the first and last actions of their respective functions. All of the other functions make use of the value of PERL\_CURRENT\_VERSION, so it must be available to execute the actions.

Add all of this code to perl.sh and all the defaults are now defined for your out-of-the-box version of Perl.

As a last note, notice that we do not add or delete the default path for Perl from PATH. There are a number of reasons. In most cases, the path for the default version of Perl is shared. In this case, removing /usr/bin from PATH would have dire consequences. A similar argument applies to the default MANPATH entry /usr/share/man where the section 1 manpages live.

You always want to leave a minimum version of Perl in the PATH because the _addpath_ and _delpath_ utilities use Perl to do their magic.

Up to this point, this all has been an interesting exercise, but without a second version of Perl, there is really no reason to have gone through all this work. Let's install another version of Perl.

### Build a New Version of Perl

I will demonstrate by installing version 5.8.0 into /opt/perl. During the configure phase (which I invoke using sh Configure), I take all the default values except for the prompts shown in <a name="re4"></a>[Example 4](Juggling-Perl-Versions-ex4.md). Subsequent questions will have their defaults altered based on these answers. (For example, we will not need to explicitly set the _man_ directory for modules \[section 3\]. Configure is smart enough to change the default to /opt/perl/man/5.8.0/man3 given the setting of man1.) I do not show those questions here since all you have to do is hit RETURN to take the now modified defaults. In order to make installation a bit smoother, create the directory

/opt/perl/bin/5.8.0

and unset PERL5LIB (if you've set it) before starting Configure. <a name="re4"></a>[Example 4](Juggling-Perl-Versions-ex4.md) shows the questions for which you must override the defaults.

Notice that we chose the location for the version number in /opt/perl/bin/5.8.0 and /opt/perl/man/5.8.0/man1 in order to mimic the structure of the lib directory that this version of Perl builds. This is not standard practice. It does, however, make our swapping code much easier to create.

Once configured, complete the installation with

make test install

### Determine Locations

We now need to get the location of the 5.8.0 Perl binary, modules and manpages.

The location of _perl_ is easy since we specified it: /opt/perl/ bin/5.8.0. Make sure that PERL5LIB is unset and execute _/opt/perl/ bin/5.8.0/perl -V_. Again, we are only interested in the tail; see <a name="re5"></a>[Example 5](Juggling-Perl-Versions-ex5.md).

Now, unlike our 5.6.1 installation, we can proceed without setting PERL5LIB and allow modules to be installed right into /opt/perl/lib/site\_perl. After all, we built this version and, therefore, have no default installation to preserve.

Create the _set_ and _unset_ Functions

We will create a function named set\_perl\_5.8.0. It will perform three functions:

1\. Set PATH so that the 5.8.0 version of Perl is on the path.

2\. Set PERL5LIB so that any extra libraries are found.

3\. Set MANPATH so that the Perl manpages are located.

Here is the code:

set\_perl\_5.8.0()
{
  export PERL\_CURRENT\_VERSION=5.8.0
  addpath -f PATH /opt/perl/bin/5.8.0
  addpath -f MANPATH /opt/perl/man/5.8.0
}
The corresponding _unset_ function is:
unset\_perl\_5.8.0()
{
  delpath -f MANPATH /opt/perl/man/5.8.0
  delpath -f PATH /opt/perl/bin/5.8.0
  unset PERL\_CURRENT\_VERSION
}

Add these two functions to perl.sh and we're almost finished.

### Create the _swap\_perl_ Function

The last function to consider is the one that actually does the swapping. You could always code something like

unset\_perl\_${PERL\_CURRENT\_VERSION}
swap\_perl\_5.8.0

before all calls to Perl, but that gets tedious after a while and is highly error prone. <a name="rl3">[Listing 3](#listing-33) shows the function _swap\__perl with commentary. In order to use the function, simply call it before executing any Perl script:

swap\_perl 5.8.0
perl foobar.pl

If you keep project-specific library locations, you can add them via arguments to _swap\__perl:

swap\_perl 5.8.0 /opt/myCompany/yadayadayada

The last lines in perl.sh should set the default Perl environment. <a name="re6"></a>[Example 6](Juggling-Perl-Versions-ex6.md) shows the specifics of our installation.

### Perl Version Selection and the Shebang Line

Perl scripts are usually executed by setting their protection mode to executable (+x in the \*NIX and Linux worlds) and invoking the script directly. By placing a shebang line at the top of the script, the shell will invoke the specified program in order to process the script. The usual shebang line is something like:

#!/usr/local/bin/perl

The problem with this is that it totally defeats the mechanism of putting the desired Perl executable in the PATH. One way around this is to modify the shebang line:

#!/usr/bin/env perl

From the manpage header:

env - run a program in a modified environment

In this usage, we do not modify the environment with the assignment of any variables; we simply use env's PATH-searching capabilities to find the version of Perl we desire.

### Shell Game

In order to swap versions of Perl on a case-by-case basis, you have to invoke _swap\_per_l before any call to Perl. For _at_ jobs, you must either call _swap\_perl_ in the job definition or in your own environment before the _at_ job is created. For _cron_ jobs or commercial job schedulers such as AutoSys, you may have to wrap the command in a shell that calls _swap\_perl_ before invoking the Perl script.

### Never Look Back

One of the potential disadvantages of having multiple versions of Perl is the need to install new modules in both places. I avoid that problem with the following policy: Once a new version of Perl has been tested enough to go into production, it is placed there with the latest version of all modules that have been installed with the last version. Any new module installations are performed on the new version only. If a developer wants/needs a new module or a module upgrade, he must migrate to the latest Perl version in order to access the module. An exception to this policy can be made for a show-stopping production bug. However, the developer will be highly encouraged to migrate immediately.

You should suggest that your developers put a _swap\__perl statement in their profiles (.bash\_profile in bash) so that they are always working in the latest version.

### Perl Versions Prior to 5.6.0

The directory layout of Perl has evolved over the years. If any of the versions of Perl that you are dealing with are earlier than 5.6.0, you may have to do some more experimentation in setting PERL5LIB in order to have @INC properly set. This may, in turn, require you to put version-specific code in the _add\__perl5lib function. Add\_perl5lib can be useful in its own right—I use _add\__perl5lib and _del\__perl5lib at the command line to temporarily choose between multiple versions of a module that I have installed in separate local trees in order to test the effect of upgrading a modules.

### Conclusion

Here is a summary of the steps needed to maintain multiple versions of Perl:

1\. Determine the default locations of the Perl binary, modules, and manpages.

2\. Create _set_ and _unset_ functions for this version of Perl. Place them in perl.sh.

3\. Install new versions of Perl, taking care to isolate them in highly version-numbered directory structures.

4\. Determine the locations of the Perl binary, modules, and manpages for the new version.

5\. Create _set_ and _unset_ functions for this version of Perl. Place them in perl.sh.

6\. Add _swap\__perl to perl.sh. Add a command to swap in the default version at the bottom of perl.sh. Place perl.sh in /etc/profile.d.

7\. Call _swap\__perl in your process before executing Perl scripts.

8\. Change the shebang line of Perl scripts to use the _env_ command, which will search the PATH for the desired version of Perl.

**TPJ**

#### Listing 1

```sh
\# -\*- ksh -\*-
# =====================================================================
# $Source: /home/cvs/repository/profiles/path\_funcs.sh,v $
# $Revision: 1.1 $
# $Date: 2002/07/20 02:46:53 $
# $Author: matthew $
# $Name:  $ - the cvs tag, if any
# $State: Exp $
# $Locker:  $
# =====================================================================

##
## Functions to manipulate paths safely
##
PD=${PERL\_DEFAULT:-/usr/bin/perl}

addpath()
{

  ## Get the path option. Figure out the pathvar and its value and
  ## pass both down to the Perl code. Why? Because if the pathvar is
  ## NOT exported, it does not end up in Perl %ENV and you will end
  ## up always redefining it instead of adding to it.

  ## Inits
  parg=''
  oarg=''

  ## Sharing OPTIND within a function call
  oldOPTIND=$OPTIND
  OPTIND=1

  while getopts ':hvp:fb' opt
  do
    case $opt in
      p )
	  pvar=$OPTARG
	  eval pval=\\$$pvar
	  parg="-p $pvar=$pval"
	  ;;

      h | v | f | b )
		      oarg="$oarg -$opt"
		      ;;
    esac
  done

  shift \`expr $OPTIND - 1\`

  if \[ "$parg" = "" \]
  then
    if \[ ! "$1" = "" \]
    then
      pvar=$1
      eval pval=\\$$pvar
      parg="-p $pvar=$pval"
      shift
    fi
  fi

  OPTIND=$oldOPTIND

  ## We hardcode perl just incase we are trying to run swap\_perl,
  ## which will take perl out of the PATH at some point before
  ## putting the new version in. The default in /usr/local/bin will
  ## be sufficient for this script.

  ## If you export PATH\_FUNCS\_DEBUG=-d, you get the debugger. -d:ptkdb
  ## works even better

  results=$($PD $PATH\_FUNCS\_DEBUG -we'

##
## Options/arg processing section. Straight-forward
##
use Getopt::Std;

my %opts = ();
my @verbose = ();
my $usage = <<EOUSAGE;
Usage: addpath \[-p\] <pathvar> \[-h\] \[-v\] \[-f|-b\] <dirspec> \[<dirspec> ...\]
       Idempotently adds <dirspec> to <pathvar>
       -p specifies <pathvar>. -p is optional
       -h prints usage
       -v prints messages about the status of the command
       -f adds <dirspec> to front of <pathvar>
       -b adds <dirspec> to back of <pathvar>
EOUSAGE
;

## Process options
if (!getopts(q{hvp:fb},\\%opts)) {
    die "$usage\\n";
}

if ($opts{h}) {
    print STDERR "$usage\\n";
    exit 0;
}

if(!defined($opts{p})) {
    die "No pathvar defined\\n$usage\\n";
}

if (defined($opts{f}) and defined($opts{b})) {
    die "-f and -b options are mutually exclusive\\n$usage";
}

## Process args
if (!scalar(@ARGV)) {
    die "\\nNo dirspec specified.\\n$usage\\n"
}

## Pull the pathvar and value out of the option
my $pathvar = undef;
my $pathval = undef;
($pathvar, $pathval) = split(/=/,$opts{p});

## $pathsep may be able to be pulled out of Config.pm on a per-platform
## basis. For now, default to UNIX.
my $pathsep=":";

## Hash-out the current sub-paths for easy comparison.
my %pathsubs=();
my $pathfront=0;
my $pathback=0;
if(!defined($pathval) ||
   !length($pathval)) {
    push @verbose, "$pathvar does not exist, initial assignment";
    $pathval = ""; ## Shuts up -w undef complaint later.
} else {
    %pathsubs = map { $\_ => $pathback++} split($pathsep,$pathval);
    $pathback--;
}

## Start checking each path arg
for my $argv (@ARGV) {

    ## This is a complete comparison, no need to take care
    ## of your path posssibly being a portion of an existing path.
    if(defined($pathsubs{$argv})) {
	push @verbose, "$argv already present in $pathvar";
    } else {
	## Stick it where it belongs
	if(defined($opts{f})) {
	    $pathsubs{$argv} = --$pathfront;
	    push @verbose, "Pre-pended path $argv";
	} else {
	    $pathsubs{$argv} = ++$pathback;
	    push @verbose, "Appended path $argv";
	}
    }
}

## Put humpty dumpty back together again
$pathval = join ($pathsep,
		 sort {$pathsubs{$a} <=> $pathsubs{$b}} keys %pathsubs);

print STDERR join("\\n",@verbose) if (defined($opts{v}));

## The shell will eval this:
print "$pathvar=$pathval";

' -- $parg $oarg $\*)
eval eval $results
}
```

[Back to Article](#rl1)

#### Listing 2

```sh
delpath()
{
  ## Get the path option. Figure out the pathvar and its value and
  ## pass both down to the Perl code. Why? Because if the pathvar is
  ## NOT exported, it does not end up in Perl %ENV and you will end
  ## up always redefining it instead of adding to it.

  ## Inits
  parg=''
  oarg=''

  ## Sharing OPTIND within a function call
  oldOPTIND=$OPTIND
  OPTIND=1

  while getopts ':hvp:en' opt
  do
    case $opt in
      p )
	  pvar=$OPTARG
	  eval pval=\\$$pvar
	  parg="-p $pvar=$pval"
	  ;;

      h | v | e | n )
		      oarg="$oarg -$opt"
		      ;;
    esac
  done

  shift \`expr $OPTIND - 1\`

  if \[ "$parg" = "" \]
  then
    if \[ ! "$1" = "" \]
    then
      pvar=$1
      eval pval=\\$$pvar
      parg="-p $pvar=$pval"
      shift
    fi
  fi

  OPTIND=$oldOPTIND

  ## We hardcode perl just incase we are trying to run swap\_perl,
  ## which will take perl out of the PATH at some point before
  ## putting the new version in. The default in /usr/local/bin will
  ## be sufficient for this script.

  ## If you export PATH\_FUNCS\_DEBUG=-d, you get the debugger. -d:ptkdb
  ## works even better

  results=$($PD $PATH\_FUNCS\_DEBUG -we'

##
## Options/arg processing section. Straight-forward
##
use Getopt::Std;

my %opts = ();
my @verbose = ();
my $usage = <<EOUSAGE;
Usage: delpath \[-p\] <pathvar> \[-h\] \[-v\] \[-e\] \[-n\] <dirspec> \[<dirspec> ...\]
       Removes <dirspec> from <pathvar>
       -p specifies <pathvar>. -p is optional
       -h prints usage
       -v prints messages about the status of the command
       -e <dirspec> is used as a regexp
       -n removed non-existent directories from <pathvar>
EOUSAGE
;

## Process options
if (!getopts(q{hvp:en},\\%opts)) {
    die "$usage\\n";
}

if ($opts{h}) {
    print STDERR "$usage\\n";
    exit 0;
}

if(!defined($opts{p})) {
    die "No pathvar defined.\\n$usage\\n";
}

## Process args
if (!scalar(@ARGV)) {
    die "\\nNo dirspec specified.\\n$usage\\n"
}

# Pull the pathvar and value out of the option
my $pathvar = undef;
my $pathval = undef;
($pathvar, $pathval) = split(/=/,$opts{p});

## $pathsep may be able to be pulled out of Config.pm on a per-platform
## basis. For now, default to UNIX.
my $pathsep=":";

## Hash-out the current sub-paths for easy comparison.
my %pathsubs=();
my $pathfront=0;
my $pathback=0;
if(!defined($pathval) ||
   !length($pathval)) {
    push @verbose, "$pathvar does not exist, nothing to do";
    $pathval = ""; ## Shuts up -w undef complaint later.
    goto NOTHING\_TO\_DO;
} else {
    %pathsubs = map { $\_ => $pathback++} split($pathsep,$pathval);
    $pathback--;
}

## Start checking each path arg
for my $argv (@ARGV) {

    my @matches = ();
    my $msg = undef;
    if(defined($opts{e})) {
	$msg = 'Regexp';
	@matches = grep {$\_ =~ /$argv/} keys %pathsubs;
    } else {
	$msg = 'Path';
	@matches = grep {$\_ eq $argv} keys %pathsubs;
    }
    if(scalar(@matches)) {
	delete @pathsubs{@matches};
	push @verbose, "Deleted paths ", join(",",@matches);
    } else {
	push @verbose, "$msg $argv not found";
    }
}

## Check for empties
if(defined($opts{n})) {
    @matches = grep {! -d $\_} keys %pathsubs;
    if(scalar(@matches)) {
	delete @pathsubs{@matches};
	push @verbose, "Deleted non-existent paths ", join(",",@matches);
    }
}

## Put humpty dumpty back together again
$pathval = join ($pathsep,
		 sort {$pathsubs{$a} <=> $pathsubs{$b}} keys %pathsubs);

NOTHING\_TO\_DO:
print STDERR join("\\n",@verbose) if (defined($opts{v}));

## The shell will eval this:
print "$pathvar=$pathval";

' -- $parg $oarg $\*)
eval eval $results
}
```

[Back to Article](#rl2)

#### Listing 3

```sh
swap\_perl()
{
  ## Validate the version argument
  to=$1
  case $to in
    5.6.1 | 5.8.0 )
	    ok=1
	    ;;
    \*)
       echo "Your choices are 5.6.1 and 5.8.0. Bye."
       return 1
       ;;
  esac
  shift

  ## Determine what version is to be swapped out and do so, if
  ## possible

  if \[ "$PERL\_CURRENT\_VERSION" = "" \]
  then
    echo "No PERL\_CURRENT\_VERSION defined to swap out. Continuing..."
  elif \[ "$PERL\_CURRENT\_VERSION" = "initial" \]
  then
    echo "Silently skip unsetting Perl" > /dev/null
  elif \[ "$PERL\_CURRENT\_VERSION" = "$to" \]
  then
    echo "Current Perl version is already $to. Exiting."
    return
  else
    echo "Swapping out $PERL\_CURRENT\_VERSION..."

    ## Multiple levels of string interpolation simplify our job here...
    unset\_perl\_$PERL\_CURRENT\_VERSION
  fi

  echo "Swapping in $to..."

  ## and here...
  set\_perl\_$to

  ## Any other argument given to the function is treated as a
  ## potential location for modules. You can add your own local
  ## libraries to the PERL5LIB variable by specifying the root
  ## directories as arguments after the version number.

  adds="$\*"
  if \[ ! "$adds" = "" \]
  then
    for i in $adds
    do
      add\_perl5lib $i
    done
  fi
}
```

[Back to Article](#rl3)
