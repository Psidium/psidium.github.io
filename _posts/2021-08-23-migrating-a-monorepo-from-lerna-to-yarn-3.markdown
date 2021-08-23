---
layout: post
title:  "Migrating a monorepo from Lerna to Yarn 3 with PnP and Zero Install"
date:   2021-08-23 11:00:23 -0300
author: Gabriel Borges
categories: lerna monorepo yarn nodejs pnp zero-install pnp zero-install
background: '/imgs/yarn3.jpg'
---

After sitting through a 15 minute `yarn install` I said "enough is enough" and I decided to migrate to the recently announced [Yarn 3][yarn-3] with PnP support to get rid of our install process. Yes, get rid of `yarn install` via [Zero Install](#fifth-step-enabling-zero-install).

This post is verbose on purpose. I've searched for the errors Yarn was outputting multiple times and didn't find much help online, so I am adding all the errors I had here so that I can save you some time [^thanks]. Also, I may refer to Yarn as Yarn Berry or Yarn 3, they all mean the same thing, as Berry is the name of the repository post-v1.

# First step: migrating to Yarn 3

 We've been using [lerna][lerna] for our JavaScript monorepo project [^monorepo]. I would glady stay with `lerna`, but after some tries I couldn't get lerna to work with `yarn 3`, and it seems it won't be [supporting the new yarn versions][lerna-upgrade-yarn] anytime soon. Well...


The [migration guide][yarn-migration] from the Yarn team is great, I'll take the liberty of copying it here for ease of access:

#### `<yarn-migration>`

1. Run `npm install -g yarn` to update the global yarn version to latest v1
2. Go into your project directory
3. Run `yarn set version berry` to enable v2 (cf [Install](https://yarnpkg.com/getting-started/install) for more details)
4. If you used `.npmrc` or `.yarnrc`, you'll need to turn them into the [new format](https://yarnpkg.com/configuration/yarnrc) (see also [1](https://yarnpkg.com/getting-started/migration#update-your-configuration-to-the-new-settings), [2](https://yarnpkg.com/getting-started/migration#dont-use-npmrc-files))
5. Add [`nodeLinker: node-modules`](https://yarnpkg.com/configuration/yarnrc#nodeLinker) in your `.yarnrc.yml` file (we will try to enable this later)
6. Commit the changes so far (`yarn-X.Y.Z.js`, `.yarnrc.yml`, ...)
7. Run `yarn install` to migrate the lockfile
8. Take a look at [this article](https://yarnpkg.com/getting-started/qa#which-files-should-be-gitignored) to see what should be gitignored
9. Commit everything remaining

#### `</yarn-migration>`

On step 8 they recommend modifying your `.gitignore` file. Here I recommend the [gitignore plugin][gitignore-plugin] from `oh-my-zsh`, which allows me to just run `gi yarn >> .gitignore` and forget about it. Also, if you plan on using Zero Install, [go check that section for more on gitignores](#fifth-step-enabling-zero-install)

Also go ahead and delete all `yarn.lock` files that *aren't in the root of your project*. This took me some time to figure out, but Yarn looks for the `yarn.lock`s outside the root folder to mark that dir as a possible [`node_modules`-compatible sub-package](https://yarnpkg.com/getting-started/recipes/#hybrid-pnp--node_modules-mono-repo).


### Moving from `.npmrc` and `.yarnrc` to `.yarnrc.yml`

Yarn 3 completely ignores the `.npmrc` and `.yarnrc` files on your project. I was actually using those to connect with my private npm repository.

My current `.npmrc` contains:

{% highlight yaml linenos %}
registry=https:/example.com/artifactory/api/npm/some.proxy/
always-auth=true
{% endhighlight %}

which becomes a `.yarnrc.yml` (keep reading and you'll know what issue does this excerpt hold here):

{% highlight yaml linenos %}
# danger here error here
yarnPath: ".yarn/releases/yarn-berry.cjs"
npmRegistryServer: "https:/example.com/artifactory/api/npm/some.proxy/"
npmAlwaysAuth: true
{% endhighlight %}

The `.yarnrc` contained a pointer to `.yarn/releases/yarn-1.22.11.cjs`, which I won't be needing anymore. So I deleted both the `.yarnrc` and this file.

### nohoist

We used the `nohoist` option in our `package.json`, which has a slightly different behavior in Yarn 3. Previously, you'd list in the `nohoist` option which packages should not be hoisted to the top-level `node_modules`  folder, and all sub-packages would follow that. Now, you'd go to the specific `package.json` and set the [installConfig][installConfig] option to avoid hoisiting that whole workspace. This is a great change if you ask me, specially if you are using Create React App.

Since I only had `"nohoist": []` there, I just deleted the property, but your case may be different.

### Authenticating with the npm repository

This was hard to figure out.

The first thing you need to do after setting up your `.yarnrc.yml` is run `yarn npm login`. After you have this login setup, you need to open your `~/.yarnrc.yml` (on your Home, not your project) and add `npmAlwaysAuth: true` under the login settings for the registry. 


<details> <summary>
I had way too many issues to configure this, only figuring out after debugging Yarn itself. Open here to see my issues and how to debug Yarn. 
</summary>

I tried to follow the documentation anyway I could but I kept getting either this error:

{% highlight shell %}
❯ yarn install
➤ YN0000: ┌ Resolution step
➤ YN0033: │ babel@npm:6.23.0: No authentication configured for request
➤ YN0000: └ Completed in 0s 364ms
➤ YN0000: Failed with errors in 0s 369ms
{% endhighlight %}

or this "Invalid authentication (as an anonymous user)" error:

{% highlight shell %}
❯ yarn install
➤ YN0000: ┌ Resolution step
➤ YN0041: │ eslint-plugin-promise@npm:^5.1.0: Invalid authentication (as an anonymous user)
➤ YN0000: └ Completed in 1s 775ms
➤ YN0000: Failed with errors in 1s 783ms

_e [Error]: convert-source-map@npm:^1.7.0: Invalid authentication (as an anonymous user)
    at xa (~/.yarn/releases/yarn-berry.cjs:558:15471)
    at runMicrotasks (<anonymous>)
    at processTicksAndRejections (node:internal/process/task_queues:96:5)
    at async jn (~/.yarn/releases/yarn-berry.cjs:558:16330)
    at async eR.getCandidates (~/.yarn/releases/yarn-berry.cjs:558:24344)
    at async Xc.getCandidates (~/.yarn/releases/yarn-berry.cjs:294:4907)
    at async Xc.getCandidates (~/.yarn/releases/yarn-berry.cjs:294:4907)
    at async ~/.yarn/releases/yarn-berry.cjs:303:7635
    at async jl (~/.yarn/releases/yarn-berry.cjs:242:3997)
    at async y (~/.yarn/releases/yarn-berry.cjs:303:7617) {
  reportExtra: undefined,
  reportCode: 41
}
{% endhighlight %}

I simply could not find anything online that would point me in the right direction here, it appeared as if Yarn simply gave up on trying to authenticate me with my registry. After trying for long I started inserting `debugger` statements inside the Yarn compiled source code, gave up on that, bit the bullet and downloaded the berry codebase to debug yarn from inside.

It turns out [Yarn can't merge your local registry config and your global's config][yarn-issue]. At least I learned how to debug yarn?


#### How to debug Yarn berry itself

If you ever need to debug Yarn itself, this is how you do it:

1. Clone [github.com/yarnpkg/berry/](https://github.com/yarnpkg/berry/);
1. Run `yarn install` on the resulting dir;
1. (Optional for macOS) run `realpath scripts/run-yarn.js | pbcopy` to copy this path to your clipboard
1. Go to your original folder (where the bad things are happening);
1. Change the `yarnPath` entry in `.yarnrc.yml` to `/path/to/your/clone/berry/scripts/run-yarn.js` (or just paste it if you used the command above)

Now whenever you run `yarn` inside this project folder you will use the development version. Props to the yarn team for using `node-ts` to load the typescript files on the go, that helped me a lot.
</details>

<br>

# Second Step: From `lerna` to `yarn`

In order to migrate from lerna we need to add a new plugin to our environment:

{% highlight shell linenos %}
yarn plugin import workspace-tools
{% endhighlight %}

## lerna.json

While it is installing the plugin, let's go ahead and delete our `lerna.json` file, remove all scripts that run `lerna`, and remove the dependendcy from our packages. If you only had your subpackages listed in `lerna.json`, it is time to migrate it to Yarn's ``package.json`:

{% highlight json linenos %}
// from lerna.json
{
  "packages": [
    "apps/*",
    "packages/*",
    "tools/*"
  ],
}

// to package.json
{
  "workspaces": {
    "packages": [
      "packages/*",
      "apps/*",
      "tools/*"
    ]
  }
}
{% endhighlight %}

## The [`workspace:` protocol][protocols]

Yarn berry now supports a few things that lerna didn't, one of those things is explicitly setting the package imports to always match the ones on your monorepo. This means that it doesn't matter when or where this package is run, it will only work when running inside the monorepo. If your CI/CD are ready to deal with monorepos, this can be a great thing, and you can go ahead and change the version of your packages to `workspace:*` (this will match the folder you're in). Unfortunatedly, that is not my case and I'll have to go without this.

## The `yarn start` command

In many React projects we have the `start` script ready to get things going. When using  `lerna` we used to have the following command to use it in our monorepo:

{% highlight shell linenos %}
lerna run --parallel --concurrency 2 start
{% endhighlight %}

This would simply execute all scripts named `start` in our subpackages. While okay, it did waste resources by running `start` on unnecessary packages.

Now, with Yarn berry we can run, directly in the package we need:

{% highlight shell linenos %}
yarn workspaces foreach -vpiR run start
{% endhighlight %}

This command come from the `workspace-tools` plugin I've mentioned. It is using these options:

* `v`: verbose, prefix the output with the package that printed that;
* `p`: run in parrarel;
* `i` print all outputs in realtime;
* `R`: recursive.
    * Recursive is the real beauty here. It will traverse your `dependencies`/`devDependencies` and only run `start` on the packages that actually need to be started! 

### Recursive explained

Here's an example. A workspace have these packages, all with a `start` script in their `package.json` file:

* `@my/a`, importing:
    * `@my/b`
    * `@my/c`
* `@my/d`, importing:
    * `@my/b`
    * `@my/d`
    * `@my/e`.

With lerna, all of these packages' `start` script would run, no matter if I am working only on `@my/a` right now. With the `recursive` option, if I run it inside `@my/a`, it will only run `start` on `@my/a`, `@my/b`, and `@my/c`, leaving `@my/d` and `@my/e` alone.

<details>
<summary>
More problems
</summary>

It turns out that when running the start command mentioned above, one can get the following error:

{% highlight shell linenos %}
Internal Error: @my-company/launchpad@workspace:.: This package doesn't seem to be present in your lockfile; run "yarn install" to update the lockfile
{% endhighlight %}

As mentioned in the beggining, you need to delete the yarn.lock files inside all subpackges on yarn 3, something that yarn 1 created.

</details>

If you are following along with your repo, I recommend you to commit your changes now, as this has been some work already!

# Third Step: Moving to PnP

You may not need PnP, you may need to evaluate if the gains from enabling PnP are really worth it for you, there are a lot stuff that breaks when migrating... But I am here for the Zero Install! Well, once again, the migration tutorial covers a lot of the use cases, lets copy it again:

#### `<yarn-migration>`

##### Enabling it

1. Look into your `.yarnrc.yml` file for the [`nodeLinker`](https://yarnpkg.com/configuration/yarnrc#nodeLinker) setting
2. If you don't find it, or if it's set to `pnp`, then it's all good: you're already using Plug'n'Play!
3. Otherwise, remove it from your configuration file
4. Run `yarn install`
5. Various files may have appeared; check [this article](https://yarnpkg.com/getting-started/qa#which-files-should-be-gitignored) to see what to put in your gitignore
6. Commit the changes

##### Editor support

We have a [dedicated documentation](https://yarnpkg.com/getting-started/editor-sdks), but if you're using VSCode (or some other IDE with Intellisense-like feature) the gist is:

1. Install the [ZipFS](https://marketplace.visualstudio.com/items?itemName=arcanis.vscode-zipfs) VSCode extension
2. Make sure that `typescript`, `eslint`, `prettier`, ... all dependencies typically used by your IDE extensions are listed at the *top level* of the project (rather than in a random workspace)
3. Run `yarn dlx @yarnpkg/sdks vscode`
4. Commit the changes - this way contributors won't have to follow the same procedure
5. For TypeScript, don't forget to select [Use Workspace Version](https://code.visualstudio.com/docs/typescript/typescript-compiling#_using-the-workspace-version-of-typescript) in VSCode

#### `</yarn-migration>`

I didn't know which deps should be on the root or not when I ran the step #2  for the editor support. So I did like any dev would and [looked at the code][sdk-install] to see which tools are supported by it. In my case I only needed to move `typescript`, `eslint` and `prettier` (my CSSs are written by another team).

Another caveat I bumped into was that after doing all that, my VSCode's TypeScript still didn't find my files. Turns out that if you have a `.code-workspace` file, VSCode will load the settings from there! So I added the settings that Yarn generated in that file and voila!

# Fourth Step: PnP Issues

If you have a `build` script in all your packages, you can run

{% highlight shell linenos %}
yarn workspaces foreach -vtR run build
{% endhighlight %}

and it will show you if your build is running or not. Turns out that mine wasn't well. 

One of the packages I depend on didn't declare their dependencies correctly. In order to fix that I was tempted to add their dependency in my root's `package.json`, well, this broke other stuff in my project because the mere instalation of that particular package interfered with the generation of the html on my compilation (yeah). 

The correct approach here is to use [Yarn's package extension](https://yarnpkg.com/configuration/yarnrc#packageExtensions) feature to correctly patch their `package.json` and load the package they are missing.

---

My next headache was with `eslint`. I first went with the dumb route and started adding my configurations in the root folder, but it wasn't cutting it. In my setup I have a package called `@mycompany/eslint-config-react` for my React apps, and this package is meant to be the only eslint config place for the project. To make `eslint` work you need to add [`@rushstack/eslint-patch`](https://yarnpkg.com/package/@rushstack/eslint-patch) to your configuration project, and on your `.eslintrc.js` (or `index.js`) you have to add:

{% highlight js linenos %}
require("@rushstack/eslint-patch/modern-module-resolution");
{% endhighlight %}

In my project this highlighted a bunch of plugins I was using but didn't declare on my package.json.

---

After adding all those plugins, I have the following error:

{% highlight shell linenos %}
 Plugin "@typescript-eslint" was conflicted between ".eslintrc.js » @my-company/eslint-config-react » plugin:@typescript-eslint/recommended-requiring-type-checking » ./configs/base" and ".eslintrc.js » @my-company/eslint-config-react » eslint-config-airbnb-typescript »
{% endhighlight %}

Running eslint directly yielded a better error log:

{% highlight shell linenos %}
$ yarn dlx eslint .  --ext 'js,jsx,ts,tsx'

➤ YN0000: ┌ Resolution step
➤ YN0000: └ Completed in 15s 691ms
➤ YN0000: ┌ Fetch step
➤ YN0000: └ Completed
➤ YN0000: ┌ Link step
➤ YN0000: └ Completed
➤ YN0000: Done in 15s 937ms


Oops! Something went wrong! :(

ESLint: 7.32.0

ESLint couldn't determine the plugin "@typescript-eslint" uniquely.

- ~/.yarn/__virtual__/@typescript-eslint-eslint-plugin-virtual-12afe7113d/0/cache/@typescript-eslint-eslint-plugin-npm-4.29.2-9a65d82faf-3d3646059d.zip/node_modules/@typescript-eslint/eslint-plugin/dist/index.js (loaded in ".eslintrc.js » @my-company/eslint-config-react » plugin:@typescript-eslint/recommended-requiring-type-checking » ./configs/base")
- ~/.yarn/__virtual__/@typescript-eslint-eslint-plugin-virtual-71978a2ec5/0/cache/@typescript-eslint-eslint-plugin-npm-4.29.2-9a65d82faf-3d3646059d.zip/node_modules/@typescript-eslint/eslint-plugin/dist/index.js (loaded in ".eslintrc.js » @my-company/eslint-config-react » eslint-config-airbnb-typescript » ~/.yarn/cache/eslint-config-airbnb-typescript-npm-12.3.1-9d9c799fa7-2aac9d736c.zip/node_modules/eslint-config-airbnb-typescript/lib/shared.js")

Please remove the "plugins" setting from either config or remove either plugin installation.
{% endhighlight %}


Fixing this was simple: upgrade this rowdy package (really they've fixed it yesterday 😂). 

---

On to the `eslint`/`jest` error:

{% highlight shell linenos %}
Error while loading rule 'jest/no-deprecated-functions': Unable to detect Jest version - please ensure jest package is installed, or otherwise set version explicitly
{% endhighlight %}

To fix this, set the `jest` version:


{% highlight js linenos %}
module.exports = { 
// ...
settings: {
  jest: {
    version: 27,
  },
};
{% endhighlight %}

---

`eslint-plugin-import` has some issues with PnP.

{% highlight shell linenos %}
Resolve error: unable to load resolver "node"  import/
{% endhighlight %}

`eslint-plugin-import` needs to know how to resolve the packages, it should do that internally, but it tries to load the resolver from the project that the linted files are in. This makes things weird here on PnP because we do not have all files dumped on the same dir. Usually the resolution is to add [eslint-import-resolver-node](https://github.com/airbnb/javascript/issues/1730#issuecomment-364796497) as a direct dev dependency. Since we are using workspaces it must be added on the root `package.json`.

# Fifth Step: Enabling zero install

Before enabling [zero installs](https://yarnpkg.com/features/zero-installs) to speed up your dev lifecycle, clone your project anew and make a clean install to see if you are on the green.

You can follow Yarn's guide [here](https://yarnpkg.com/getting-started/qa#which-files-should-be-gitignored) to un-ignore the cache files. Adding to your `.gitignore` the following:

{% highlight shell linenos %}
.yarn/*
!.yarn/cache
!.yarn/patches
!.yarn/plugins
!.yarn/releases
!.yarn/sdks
!.yarn/versions
{% endhighlight %}

Have you noticed there are some packages that compile some stuff after you install them? Think `fsevents`.  Well, after they compile, Yarn dumps their content unto `.yarn/unplugged`.

You have two ways to deal with this:

* The Yarn team recommends you to ignore this folder.
    * If you are going to ignore this folder, you need to add `enableScripts: false` to your `.yarnrc.yml` file so that no package can ever add something there. 
* If, however, you need the files generated, you'll have to add `!.yarn/unplugged` to your `.gitignore` folder, and then `git add -f .yarn/unplugged`. 

In my case, I went with the ignore/deletion route, but you should test your project to see what to do.

# Conclusion


After doing all that work (seriously it took me days). We need to check if it works! Clone your project and try to `build`/`start` it! If it doens't work, well, you'll have to dig deeper.


Was it worth my investment? In my context, the time saved by having Zero Install is surely worth it, people can focus more on their code and less on their tooling.

Seriously, [let me know](https://twitter.com/psidium_) if you have problems (post on StackOverflow and send me a link), or if I've helped you at all.


# Next Steps

Things I didn't cover, but you might need:

* versioning
* detecting changes for rebuilding only the necessary packages
* publishing the packages

[^monorepo]: Currently just a React application with multiple Create React App and microbundle libraries
[^thanks]: If this has helped you in any way please let me know on [Twitter](https://twitter.com/psidium_). I like to know when I help people.

[yarn-3]: https://github.com/yarnpkg/berry/blob/master/CHANGELOG.md#300
[lerna]: https://lerna.js.org
[lerna-upgrade-yarn]: https://github.com/lerna/lerna/issues/2449
[yarn-migration]: https://yarnpkg.com/getting-started/migration#step-by-step
[gitignore-plugin]: https://github.com/ohmyzsh/ohmyzsh/tree/master/plugins/gitignore
[installConfig]: https://yarnpkg.com/configuration/manifest#installConfig
[yarn-issue]: https://github.com/yarnpkg/berry/issues/2106
[protocols]: https://yarnpkg.com/features/protocols
[sdk-install]: https://github.com/yarnpkg/berry/blob/master/packages/yarnpkg-sdks/sources/generateSdk.ts#L177


