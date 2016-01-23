---
layout:     post
title:      Speeding up your CI builds
date:       2016-01-23
summary:    Using cache to speed up your builds.
tags:
 - deployment
 - drone
 - jenkins
 - continuous integration
 - docker
---

The whole idea of doing continuous integration can be summed up in this statement: "Integrate early, integrate often". For this to work, build must be fast. How fast depends on each project, but "as fast as possible" is a good starting point.

But integrating often is almost worthless if the result is not reliable. That is why each build should be self contained and reproducible: the same build under any circumstances should have the same result, no matter what happened before in the integration environment.

The trick is to juggle between both of the above ideas, given that the second one tends to attempt against the first.

In a perfect world, to achieve full reproducibility, one may have a brand new machine on every build. Of course this is not possible, but there are other still useful solutions:

1. Containers (id: docker).
2. Chroots.
3. Workspaces (as in jenkins using a plugin cleanear plugin maybe, or doing it by hand).
4. An empty directory at least!

Unfortunately, this means installing all of the application dependency each time which leads to very slow builds.

Just to have an approximation of what this means: a medium size single page web application we develop at @grandata used to take up to fifthteen minutes to download the whole package.json dependencies from npm. And that was just the first step in the pipeline.

Of course, there are palliative measures that can be taken. Using the same javascript application example, one could just 
to avoid hitting the public repository every time by installing a private one (even with fallback to the global). [Sinopia](https://github.com/rlidwka/sinopia) is a great option for this. In the case of scala project, we have a bunch of these too, one can use a [repository manager](https://maven.apache.org/repository-management.html). And for python a pypi cache proxy can be installed.

Those are great options which should be used whenever possible. If that cannot be done or if it is not enough, most continuous integrations systems allow some kind of cache.

In this case, a great candidate for cache is node_modules directory.

Let's see an example using [Drone](https://github.com/drone/drone). This is similar with other continuous integration systems, but one of the advantages of Drone is that each builds run in its own docker container.

```yaml
  # .drone.yml
  build:
    image: node
    commands:
      - npm -d install
      - npm test 

  cache:
    mount:
      - node_modules
```
The important part is the use of [cache](http://readme.drone.io/usage/caching/) directive which is actually a drone plugin. It says: keep the node_modules directory between builds. That would speed up the build, the second time it is run "npm install" wont download or install anything. 

But, wait a minute, shouldn't be each build self contained? Isn't this kind of the opposite? Yes, unless we know when to invalidate the cache.

### So when should the dependency cache be invalidated? 

That is application -and technology used- specific. In npm javascript application (using node, webpack, browserify), it is fairly safe to assume that dependencies changed only when package.json changes. In a scala application maybe it changes when build.sbt change.

So, in this example: npm modules should be downloaded and installed only if package.json has changed. Otherwise, the current build may use the libraries installed by the last build.

This one line will tell which was the last commit that modified this file:

```bash
git log -n 1 --pretty=format:%h  -- package.json
```

With that information, it is only a matter of store the commit hash somewhere in the cached directory. If the save one is different from the current one, the whole directory must be erased (or even just the modified parts if you like)


```bash
# ./.drone/clear_node_modules_cache.sh
#!/bin/bash
PACKAGES_VERSION_FILE="node_modules/.packages_version"

LAST_COMMIT=$(git log -n 1 --pretty=format:%h  -- package.json)

if [ -z "$LAST_COMMIT" ]; then
  LAST_COMMIT="default"
fi

echo "Last commit that changed 'package.json': $LAST_COMMIT"

if [[ ! -f $PACKAGES_VERSION_FILE ]]; then
  NPM_PREV_VERSION=0
else
  NPM_PREV_VERSION=`cat $PACKAGES_VERSION_FILE`
fi

if [[ $LAST_COMMIT != $NPM_PREV_VERSION ]]; then
    echo "package.json has changed. Removing node_modules";
    rm -rf node_modules/*
fi;


echo "Last build was: $NPM_PREV_VERSION. Next: $LAST_COMMIT"

mkdir -p node_modules/

echo $LAST_COMMIT > $PACKAGES_VERSION_FILE

```

And the modified .drone.yml file:

```yaml
  # .drone.yml
  build:
    image: node
    commands:
      - ./.drone/clear_node_modules_cache.sh
      - npm -d install
      - npm test 

  cache:
    mount:
      - node_modules
```

One may even not execute "npm install" if package.json was not modified. But it is still a good idea to run it, just in case that one module installation failed for external reasons. There maybe cases in which the cache gets corrupted. In those occasions, the cache may be deleted manually by erasing /var/lib/drone/cache/ or by changing package.json.

With this trick, our build time reduced from fifthteen minutes to less than one.


