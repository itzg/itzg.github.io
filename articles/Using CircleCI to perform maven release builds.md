<h2 id="your-project">Your Project</h2>
<p>The following operations are performed after you have at least initialized your project for git source control with <code>git init</code>.</p>
<h3 id="add-build-support-module">Add build-support module</h3>
<p>A helper script is needed to execute the Maven release (and deploy) sequence, so first add the <code>build-support</code> sub-module to your project:</p>
<pre><code>git submodule add https://github.com/moorkop/build-support.git
</code></pre>
<h3 id="settings.xml">settings.xml</h3>
<p>Add a <code>settings.xml</code> to your project’s base directory to convey the server credentials needed to publish to Bintray. The following content can be used as is for that file:</p>
<pre><code>&lt;?xml version='1.0' encoding='UTF-8'?&gt;
&lt;settings xsi:schemaLocation='http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd'
          xmlns='http://maven.apache.org/SETTINGS/1.0.0' xmlns:xsi='http://www.w3.org/2001/XMLSchema-instance'&gt;
    &lt;servers&gt;

        &lt;server&gt;
            &lt;id&gt;bintray&lt;/id&gt;
            &lt;username&gt;${env.BINTRAY_USER}&lt;/username&gt;
            &lt;password&gt;${env.BINTRAY_API_KEY}&lt;/password&gt;
        &lt;/server&gt;

    &lt;/servers&gt;
&lt;/settings&gt;
</code></pre>
<h3 id="pom.xml">pom.xml</h3>
<p>Some specifics will need to be declared within your <code>pom.xml</code>.</p>
<h4 id="scm-section">SCM section</h4>
<p>Add the following section to the top level of your <code>pom.xml</code> but replace the following placeholders with your GitHub specifics:</p>
<ul>
<li><code>[[user]]</code></li>
<li><code>[[repo]]</code></li>
</ul>
<pre><code>&lt;scm&gt;
  &lt;connection&gt;scm:git:https://github.com/[[user]]/[[repo]].git&lt;/connection&gt;
  &lt;url&gt;https://github.com/[[user]]/[[repo]]&lt;/url&gt;
  &lt;tag&gt;HEAD&lt;/tag&gt;
&lt;/scm&gt;
</code></pre>
<h4 id="plugins-section">Plugins section</h4>
<p>Declare the latest deploy plugin to avoid a bug in older versions that would leave out the <code>git commit</code> step. Add the following text as is in the <code>build &gt; plugins</code> section:</p>
<pre><code>&lt;plugin&gt;
  &lt;artifactId&gt;maven-release-plugin&lt;/artifactId&gt;
  &lt;version&gt;2.5.3&lt;/version&gt;
&lt;/plugin&gt;
</code></pre>
<h4 id="distribution-management-section">Distribution management section</h4>
<p>To complement the Bintray configuration in <code>settings.xml</code>, add the following text as is to the top level of your <code>pom.xml</code>:</p>
<pre><code>&lt;distributionManagement&gt;
  &lt;repository&gt;
    &lt;id&gt;bintray&lt;/id&gt;
    &lt;name&gt;bintray&lt;/name&gt;
    &lt;url&gt;https://api.bintray.com/maven/${env.BINTRAY_REPO_OWNER}/${env.BINTRAY_REPO}/${project.artifactId}/;publish=1
    &lt;/url&gt;
  &lt;/repository&gt;
&lt;/distributionManagement&gt;
</code></pre>
<h3 id="circle.yml">circle.yml</h3>
<p>We’ll hook the release process into the <code>deployment</code> stage of your CircleCI build configuration. Use the following snippet as an example for yours. You can specify your mainline branch instead of <code>master</code>, if needed.</p>
<pre><code>deployment:
  releases:
    branch: master
    commands:
      - build-support/handle-mvn-release.sh
</code></pre>
<p>Since a git sub-module is being used you’ll also need to add this:</p>
<pre><code>checkout:
  post:
    - git submodule sync
    - git submodule update --init
</code></pre>
<h2 id="bintray">Bintray</h2>
<p>Next, you will need to prepare a Bintray repository and project.</p>
<h3 id="api-key">API Key</h3>
<p>Instead of your password, you will use your API key to deploy to Bintray. Obtain your API key from <a href="https://bintray.com/profile/edit">your profile, in the edit section</a>.</p>
<h3 id="create-project-entry">Create project entry</h3>
<p>Create a project entry with the same name as your project’s <code>artifactId</code> and note the name of the containing repository since you’ll need that for the <code>BINTRAY_REPO</code> variable, below.</p>
<h2 id="circleci-project">CircleCI project</h2>
<p>You’re almost done now. The last piece is configuring the project environment variables and push access for GitHub.</p>
<h3 id="setup-github-user-key-access">Setup GitHub user key access</h3>
<p>To enable git push access from within your builds, go to your project’s settings and the section:</p>
<blockquote>
<p>Permissions &gt; Checkout SSH keys</p>
</blockquote>
<p>Click the action to “Create and add user key”, as shown here:</p>
<p><img src="https://i.imgur.com/AK1BFHV.png" alt="enter image description here"></p>
<h3 id="setup-project-environment-variables">Setup project environment variables</h3>
<p>Also in your project’s settings, add these environment variables:</p>
<ul>
<li>BINTRAY_USER</li>
<li>BINTRAY_API_KEY</li>
<li>BINTRAY_REPO_OWNER</li>
<li>BINTRAY_REPO</li>
</ul>
<h3 id="double-check-ubuntu-version">Double-check Ubuntu Version</h3>
<p>At the time of writing this, the <code>git push</code> works only with the choice of “Ubuntu 12.04” in the project’s Build Environment settings:<br>
<img src="https://i.imgur.com/3dEJlUb.png" alt="enter image description here"></p>
<h2 id="perform-a-release">Perform a release</h2>
<p>When you’re ready to perform a release, invoke the <a href="https://circleci.com/docs/parameterized-builds/">CircleCI parameterized build API</a> to trigger a build.</p>
<p>In the POST body, replace these placeholders:</p>
<ul>
<li><code>[[tag]]</code> : this will be the Git tag of the release and the default release name. GitHub recommends you prefix the tag with a “v” and use <a href="http://semver.org/">semantic versioning</a>.</li>
<li><code>[[version]]</code> : this will be the Maven project’s version, which is typically the same as the <code>tag</code> but without the “v” prefix.</li>
<li><code>[[next]]</code> : this is the next snapshot version you’ll want to use for continued development after the release. I recommend using a two-part shortening of the semantic version with the minor version bumped up to the next.</li>
<li><code>[[email]]</code> : your email address as configured in your GitHub account</li>
</ul>
<p>In the POST URL, replace these parameters (or use the param editor in a tool like <a href="https://www.getpostman.com/">Postman</a>):</p>
<ul>
<li><code>:username</code> : GitHub username</li>
<li><code>:project</code> : GitHub project name</li>
<li><code>:branch</code> : the branch to build, which needs to match the <code>branch</code> configured in your <code>circle.yml</code>, above.</li>
<li><code>:token</code> : an API token allocated from the <a href="https://circleci.com/account/api">API Tokens section in your account settings</a></li>
</ul>
<pre><code>curl -X POST -H "Content-Type: application/json" -d '{
    "build_parameters": {
        "MVN_RELEASE_TAG": "[[tag]]",
        "MVN_RELEASE_VER": "[[version]]",
        "MVN_RELEASE_DEV_VER": "[[next]]-SNAPSHOT",
        "MVN_RELEASE_USER_EMAIL": "[[email]]",
        "MVN_RELEASE_USER_NAME": "Via CircleCI"
    }
}' "https://circleci.com/api/v1/project/:username/:project/tree/:branch?circle-token=:token
</code></pre>
<h2 id="enjoy">Enjoy</h2>
<p>Your API-triggered-build includes “Build Parameters” that you can inspect later to see what exact release was performed:<br>
<img src="https://i.imgur.com/UNlEuxm.png" alt="enter image description here"></p>
