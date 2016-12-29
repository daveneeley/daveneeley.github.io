---
layout: post
title: Artifactory Prune Report
categories: code
date: 2016-12-28 16:40
---

## Pruning Unreferenced Data

My work uses Artifactory as our binary repository. If you're reading this, you probably use it too. It has this neat ability to 'prune unreferenced data', but it's one of those admin features you won't encounter in regular usages.

As the documentation points out:

> Unreferenced binary files may occur due to running with wrong file system permissions on storage folders, or running out of storage space.
> When you invoke this action, Artifactory removes unreferenced binary files and empty folders present in the filestore or cache folders.
> To avoid such errors, we recommend that you always allow Artifactory to shut down completely.

## How I Learned About It

Well, today I discovered that I made a mistake. First time, right? Over the Christmas Holiday, our Artifactory server got into a bad state. Ulimately it was out of disk space, but then I corrupted the database with some bad assumptions. A teammate moved Artifactory to a new server several months ago. Unfortunately, it appears that he migrated the database to internal storage. I never noticed. Enter the red herring. On Christmas Eve the same teammate did a live migration of the new Artifactory server between VMWare vCenters. Everything worked just fine, but it was a new process we'd never done before.

On the day after Christmas, while out of town and still on holiday, I saw an email come through from an overseas co-worker about the state of our Artifactory instance. It says Derby is throwing errors, and I'm like, 'Derby? That's the internal storage engine! We aren't even using that!'. And I set off to see why the live migration failed. I check the configuration files and see that indeed, we are not using the database I thought we should be using. I set off to resolve this and get back to my vacation. I update the configuration to use the external database, grabbing connection strings from an old backup. I even add the configuration changes to our Salt formula, so I never have to dig it up again. Our DevOps team is all stateside, so naturally no one is available to corroborate my course of action.

Over the next two days, symptoms are appearing right and left that I made a bad call. To make matters worse, once I had the server running against the external database, I had deleted the Derby files because we were out of space. And...no backups have been done since the migration, which was eight months ago. The backup storage location was never mounted on the new server. It's painful to write about this. But that's the point! Maybe, just maybe, I will never let my team do this again because I remember the pain of writing this. I'll just point them to the blog post and say, backups are important, and that blog post explains why!

## Managing the Damage

In summary, our filestore has eight months of uploads that our database does not have. If the 'prune reference data' command were to be run, all of these files would disappear. That's good and bad. We could get disk space back that we are not using. But we could lose files we can't easily re-create. What we need is a report! (Not really. That's a joke. We do need data though.)

### Grab the API Data

Artifactory has a good REST API. The data I'm after is the complete list of files that the database believes I should have, and the complete list of files that are on disk. I came up with this Powershell snippet for the former:

{% highlight Powershell %}
$repos = get-web -basicAuth -url "https://artifactory:8080/artifactory/api/repositories?type=local" | ConvertFrom-Json
foreach ($repo in $repos) {
        get-web -basicAuth -url "https://artifactory:8080/artifactory/api/storage/$($repo.key)/?list&deep=1" | ConvertFrom-Json
}
{% endhighlight %}

The Get-Web function comes from my [tcFunctions](https://github.com/SpillmanTech/tcFunctions) module. You can use Invoke-WebRequest if you manually [send the basic authentication headers](http://stackoverflow.com/a/27951845/9660). The end result is a stream of JSON objects like this:

``` Powershell
uri created                       files
--- -------                       -----
... 2016-12-28T18:24:22.215-07:00 {@{uri=/file/path.ext; folder=False; sha1=8b1d5e04a8c5727bff6041a395b629746...
```

Expanding the files element looks like this:

``` Powershell
uri          : /file/path.ext
size         : 5437440
lastModified : 2015-11-24T13:51:05.000-07:00
folder       : False
sha1         : 8b1d5e04a8c5727bff6041a395b629746
```

### Grab the Filesystem Data

With the database data in-hand, you can get the list of files on disk. Our Artifactory server runs on Linux, so I just used `find . -ls > filestore.out` from the root of the filestore to get that list. It looks like so:

``` Bash
61865988    4 drwxrwxr-x 259 artifactory artifactory     4096 May  7  2016 .
61866585   20 drwxrwxr-x   2 artifactory artifactory    20480 Dec 28 12:11 ./ac
61887700  192 -rw-rw-r--   1 artifactory artifactory   194962 May 19  2016 ./ac/ac05c6e5e153dbb8e4fa52bf07118a7584f4734c
61897157    4 -rw-rw-r--   1 artifactory artifactory     2305 Dec  8 15:35 ./ac/ac5fb2dfb7e8cab863b00eeaf309a5ae93839c05
61887674   12 -rw-rw-r--   1 artifactory artifactory    12218 Nov 22 05:12 ./ac/ac8a6c813399010b62165342763ebf79b26b25da
```

Artifactory stores each file by it's sha1 hash, sorted into subdirectories by the first two characters of the hash. There's no metadata or anything else useful here, as far as I can tell.

If you were running Artifactory on Windows, you could run `dir -recurse > filestore.out` to get a similar list.

### Merge

Now we have to merge the two lists. I copied my filestore.out file back over to my workstation and then iterated over the filestore list, which is unstructured and larger. We had 12,000 files reported from the api, but 33,000 from the filestore. Here's the whole script.

{% highlight Powershell %}
#cat is an alias to Get-Content
$alldiskfiles = cat filestore.out
#remove the directory listings from the result
$alldiskfiles = $alldiskfiles | ?{$_ -match "[a-z0-9]{40}$"}

#only collect from the local repositories
$repos = get-web -basicAuth -url "https://artifactory:8080/artifactory/api/repositories?type=local" | convertfrom-json
$alldbfiles = @()
foreach ($repo in $repos) {
    $alldbfiles += @(get-web -basicAuth -url "https://artifactory:8080/artifactory/api/storage/$($repo.key)/?list&deep=1" | convertfrom-json)
}
foreach ($diskfile in $alldiskfiles) {
    $sha = $diskfile.substring($diskfile.length - 40) 
    $results = $alldbfiles | ?{$_.sha1 -eq $sha};
    New-Object PSObject -Property @{sha1=$sha; results = $results; line = $diskfile}
}
{% endhighlight %}

This isn't very fast. I started the script just before I started the blog post, about two hours ago. It's still running. But the end result will look something like below.

``` Powershell
line                                                                                    results                                                        sha1
----                                                                                    -------                                                        ----
61887700  192 -rw-rw-r--   1 artifactory artifactory   194962 May 19  2016 ./ac/ac05...                                                                ac05c6e5e153dbb8e4fa52bf07118a7584f4734c
61897157    4 -rw-rw-r--   1 artifactory artifactory     2305 Dec  8 15:35 ./ac/ac5f...                                                                ac5fb2dfb7e8cab863b00eeaf309a5ae93839c05
61887674   12 -rw-rw-r--   1 artifactory artifactory    12218 Nov 22 05:12 ./ac/ac8a...                                                                ac8a6c813399010b62165342763ebf79b26b25da
61883621    4 -rw-rw-r--   1 artifactory artifactory      889 Dec  5 08:12 ./ac/ac92...                                                                ac92b212f1a5eca68bb744c304f9cac5349f9e3a
61893187 8412 -rw-rw-r--   1 artifactory artifactory  8613775 Dec 22 00:41 ./ac/ac1e...                                                                ac1ed1e1354c679bfbb2596168a84157cb1dcf09
61886239 3092 -rw-rw-r--   1 artifactory artifactory  3165605 Nov  9 13:12 ./ac/acae...                                                                acae28cd95b8c8fdf97f5fb5e90b8ce9a15781d8
61869107    4 -rw-rw-r--   1 artifactory artifactory     2570 May  7  2016 ./ac/ace6...                                                                ace6ff808c1ce66b57d1ebf97977acb02334cfc1
61897392 3232 -rw-rw-r--   1 artifactory artifactory  3307434 Dec  9 07:13 ./ac/ac80...                                                                ac80e0315f73b2d2d9bcf57114d4983ea78e1fe4
61884575    4 -rw-rw-r--   1 artifactory artifactory     2482 Nov 15 11:11 ./ac/ac1b...                                                                ac1b8fe6df3dfbc0131d3bcf8d8a6ff88a4e629b
61871121 1736 -rw-rw-r--   1 artifactory artifactory  1777454 May  7  2016 ./ac/acle1... {@{uri=/path/to/another/file.ext; size=1777454; lastModifi... ace11cab50fa77b10f833068c857e39245ec4f52
```

See that one entry in the results column? That's the only file in this list that exists in the database. Ouch. I'm not off to a very good start. Once my data is available, I will have to figure out how to classify the contents of each filestore file without a matching database entry. I'll save that for another post.

## Lessons Learned

First, always do backups. Always. Don't ever ever not do backups. If you have backups, don't turn them off. Don't disable them and promise to fix them later without a plan to actually do it. And of course, [test your backups](http://www.hanselman.com/blog/TheComputerBackupRuleOfThree.aspx). You can't test them if you don't have them! Second, communication with teammates is so important. Third, automate configuration management. While it may not help restore lost data, it can build confidence in a course of action that prevents that from occurring to begin with. And just as with backups, don't procrastinate this until later.