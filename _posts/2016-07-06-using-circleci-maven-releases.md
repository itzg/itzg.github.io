---
layout: post
title:  "Using CircleCI to perform Maven Releases"

---


## Your Project
The following operations are performed after you have at least initialized your project for git source control with `git init`.

### Add build-support module
A helper script is needed to execute the Maven release (and deploy) sequence, so first add the `build-support` sub-module to your project:

	git submodule add https://github.com/moorkop/build-support.git


### settings.xml
Add a `settings.xml` to your project's base directory to convey the server credentials needed to publish to Bintray. The following content can be used as is for that file:

```
<?xml version='1.0' encoding='UTF-8'?>
<settings xsi:schemaLocation='http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd'
          xmlns='http://maven.apache.org/SETTINGS/1.0.0' xmlns:xsi='http://www.w3.org/2001/XMLSchema-instance'>
    <servers>

        <server>
            <id>bintray</id>
            <username>${env.BINTRAY_USER}</username>
            <password>${env.BINTRAY_API_KEY}</password>
        </server>

    </servers>
</settings>
```

### pom.xml
Some specifics will need to be declared within your `pom.xml`.

#### SCM section
Add the following section to the top level of your `pom.xml` but replace the following placeholders with your GitHub specifics:

* `[[user]]`
* `[[repo]]`

```
<scm>
  <connection>scm:git:https://github.com/[[user]]/[[repo]].git</connection>
  <url>https://github.com/[[user]]/[[repo]]</url>
  <tag>HEAD</tag>
</scm>
```

#### Plugins section
Declare the latest deploy plugin to avoid a bug in older versions that would leave out the `git commit` step. Add the following text as is in the `build > plugins` section:

```
<plugin>
  <artifactId>maven-release-plugin</artifactId>
  <version>2.5.3</version>
</plugin>
```

#### Distribution management section
To complement the Bintray configuration in `settings.xml`, add the following text as is to the top level of your `pom.xml`:

```
<distributionManagement>
  <repository>
    <id>bintray</id>
    <name>bintray</name>
    <url>https://api.bintray.com/maven/${env.BINTRAY_REPO_OWNER}/${env.BINTRAY_REPO}/${project.artifactId}/;publish=1
    </url>
  </repository>
</distributionManagement>
```

### circle.yml
We'll hook the release process into the `deployment` stage of your CircleCI build configuration. Use the following snippet as an example for yours. You can specify your mainline branch instead of `master`, if needed.

```
deployment:
  releases:
    branch: master
    commands:
      - build-support/handle-mvn-release.sh
```

Since a git sub-module is being used you'll also need to add this:

```
checkout:
  post:
    - git submodule sync
    - git submodule update --init
```

## Bintray
Next, you will need to prepare a Bintray repository and project.

### API Key
Instead of your password, you will use your API key to deploy to Bintray. Obtain your API key from [your profile, in the edit section](https://bintray.com/profile/edit).

### Create project entry
Create a project entry with the same name as your project's `artifactId` and note the name of the containing repository since you'll need that for the `BINTRAY_REPO` variable, below.

## CircleCI project
You're almost done now. The last piece is configuring the project environment variables and push access for GitHub.

### Setup GitHub user key access

To enable git push access from within your builds, go to your project's settings and the section:

> Permissions > Checkout SSH keys

Click the action to "Create and add user key", as shown here:

![enter image description here](https://i.imgur.com/AK1BFHV.png)

### Setup project environment variables
Also in your project's settings, add these environment variables:

* BINTRAY_USER
* BINTRAY_API_KEY
* BINTRAY_REPO_OWNER
* BINTRAY_REPO

### Double-check Ubuntu Version

At the time of writing this, the `git push` works only with the choice of "Ubuntu 12.04" in the project's Build Environment settings:
![enter image description here](https://i.imgur.com/3dEJlUb.png)

## Perform a release
When you're ready to perform a release, invoke the [CircleCI parameterized build API](https://circleci.com/docs/parameterized-builds/) to trigger a build. 

In the POST body, replace these placeholders:

* `[[tag]]` : this will be the Git tag of the release and the default release name. GitHub recommends you prefix the tag with a "v" and use [semantic versioning](http://semver.org/).
* `[[version]]` : this will be the Maven project's version, which is typically the same as the `tag` but without the "v" prefix.
* `[[next]]` : this is the next snapshot version you'll want to use for continued development after the release. I recommend using a two-part shortening of the semantic version with the minor version bumped up to the next.
* `[[email]]` : your email address as configured in your GitHub account

In the POST URL, replace these parameters (or use the param editor in a tool like [Postman](https://www.getpostman.com/)):

* `:username` : GitHub username
* `:project` : GitHub project name
* `:branch` : the branch to build, which needs to match the `branch` configured in your `circle.yml`, above.
* `:token` : an API token allocated from the [API Tokens section in your account settings](https://circleci.com/account/api)

```
curl -X POST -H "Content-Type: application/json" -d '{
    "build_parameters": {
        "MVN_RELEASE_TAG": "[[tag]]",
        "MVN_RELEASE_VER": "[[version]]",
        "MVN_RELEASE_DEV_VER": "[[next]]-SNAPSHOT",
        "MVN_RELEASE_USER_EMAIL": "[[email]]",
        "MVN_RELEASE_USER_NAME": "Via CircleCI"
    }
}' "https://circleci.com/api/v1/project/:username/:project/tree/:branch?circle-token=:token
```

## Enjoy

Your API-triggered-build includes "Build Parameters" that you can inspect later to see what exact release was performed:
![enter image description here](https://i.imgur.com/UNlEuxm.png)

