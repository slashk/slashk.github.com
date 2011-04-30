--- 
layout: post
title: "Finding Your OpenStack Nova version"
category: openstack
---

_I just got back from the [OpenStack Developers Summit](http://summit.openstack.org/) and while I am still trying to compile my thoughts on the event, I thought I'd dash off this little tidbit:_

A slight annoyance in administering (and troubleshooting) OpenStack Nova is identifying your installed version. This information is actually readily (i.e. programmatically) available within the code (_and logged at the beginning of most logfiles_), but we hadn't exposed it to administrators on the command line until [Nova trunk revision #1036](http://bazaar.launchpad.net/~hudson-openstack/nova/trunk/revision/1036). With this change, you can now simply type <span class="code-inline">nova-manage version list</span> to find which OpenStack Nova version is installed. For example:

{% highlight bash %}
$ bin/nova-manage version list
2011.3-dev (2011.3-workspace:tarmac-20110428165803-elcz2wp2syfzvxm8)
{% endhighlight %}

As you can probably guess, this machine is installed with a post-Cactus / pre-Diablo version of Nova ("2011.3-dev") from the ppa:trunk packages ("tarmac-20110428165803"). Final released versions will report non-dev version numbers like `2011.1` (Bexar), `2011.2` (Cactus) or `2011.3` (upcoming Diablo). Releases from Launchpad's ppa/trunk or ppa/release repositories will report "tarmac" in the string.

For those wanting to understand this deeper: In [nova/version.py](http://bazaar.launchpad.net/~hudson-openstack/nova/trunk/view/head:/nova/version.py) there are two methods, `version_string()` and `version_string_with_vcs()`. You can see their use here in the python shell:

{% highlight python %}
$ nova-manage shell python
Python 2.6.6 (r266:84292, Sep 15 2010, 16:22:56) 
[GCC 4.4.5] on linux2
Type "help", "copyright", "credits" or "license" for more information.
(InteractiveConsole)
>>> from nova import version
>>> version.version_string()
'2011.3-dev'
>>> version.version_string_with_vcs()
u'2011.3-workspace:tarmac-20110428165803-elcz2wp2syfzvxm8'
>>> 
{% endhighlight %}

`version_string()` pulls info straight from constants in the source code while `version_string_with_vcs()` is generated at package creation time in the setup.py file (which basically writes the `nova/vcsversion.py` file with <span class="code-inline">bzr version-info --python</span> output). Unless you have run setup.py (such as when you are developing a local branch), you'll just see `2011.3-LOCALBRANCH:LOCALREVISION` as your `vcsversion` string. 

I am working on adding more diagnostic, operational support features for Diablo (service versions, flag state, etc.) and will post as they merge into OpenStack Nova trunk.
