# Multi-Environment Deployment

---

Hi everyone, my name is Craig Edwards and I'm the mobile lead for one of the big financial corporate.

Tonight I want to talk about a few simple things that we've been doing to help us streamline our build and deployment pipeline, with a particular focus on managing the application with multiple back-end environments.

Oh, a few months ago, I wrote a blog post along these lines so if you are one of the 4 people that read my blog, I apologise if you've already heard some of this stuff.

---

# Enterprise Development - Yay!

We all know that having multiple environments isn't unique to big corporates, however, it sure as hell is magnified in those types of companies.  To be honest, I'm not even sure how many we have now, but at last count, I think it was around 8 or 9!

Of course, having multiple environments doesn't just affect our development team; it also has a flow-on effect to our testing teams.  As an example, I want our testers to be able to have apps for different environments on their phone at the same time; I want them to easily be able to distinguish between them, and I want them to be able to quickly see how up-to-date their current app is.

While not strictly related to multiple environments, I'm also very conscious that we have a build system that delivers something that is supportable.  For example, does it provide the information that our call centre needs to assist the customer? Does it provide the appropriate controls and logs so that level 2 and 3 support can a) determine which build the user has, and b) be able to accurately determine how the problem occurred.

---

# Basic techniques

So, most of the info I'm going to go through depends on three key techniques. You've probably used one or more of them at various times, so I'll just quickly describe each one before I get into it.

---

# #1 - Configurations/Schemes

A Configuration is basically just a sack of compiler settings.  Xcode normally creates two configurations for you called Debug and Release when you create a project.

What we do is clone these to create a configuration for each of our environments.  In the screenshot, I have renamed Release as PROD, Debug as DEV, and cloned DEV as SYS.

A word of warning... as a general rule, I like my UAT builds to be as close to possible as production so we started by cloning our UAT configuration from the Release configuration. This ended up giving me grief because the Release configuration strips debug symbols by default, and I end up spending 20 minutes trying to figure out why the Xcode debugger couldn't match line numbers properly.

---

# #1 - Configurations/Schemes

Closely related to configurations are schemes.  A scheme is the way Xcode maps which configuration gets used when you are running, testing, archiving, etc.  We basically create as many schemes as we have environmnets (or configurations).

The scheme screenshot that you can see is for our DEV environment, so you'll note that we are using the DEV configuration everywhere.

---

# Schemes In Action

So, now that we have a scheme and configuration for each environment, it is really really easy for our developers to switch between environments.

---

# #2 Info.plist preprocessing

Obviously, we are all familiar with our old-friend, the Info.plist file - however there is a additional build setting you can use to tell Xcode to run a preprocessor over the Info.plist file.  

Additionally, you'll need to tell Xcode the name of a .h file that it will read the variables from, and then at build time Xcode will pick up whatever is in that file and substitute it into your Info.plist file.

---

# Modifying Info.plist

Once you've enabled the Info.plist preprocessing setting in your build setting, you can define your variable values.  As you can see I've defined two variables and set them to two values.

Note that the Info.plist already supports reading variables from your build settings using the Dollar/Curly brace notation.  However, if you need to dynamically generate values (eg. using a script) then you need to use preprocessing.

The great thing about this is that it facilitates updates to Info.plist without dirtying your workspace

---

# #3 Build Script

You're probably all familiar with being able to execute some script commands using the Run Script in your target's build phase. The problem with that is that it runs too late. Sometimes, to properly configure our app, we need to set things up a little earlier.

To achieve this, we can create an Aggregate Target (the little target icon) that just contains a single Run Script.  To get it to run first, we edit your target's dependencies (see screenshot) and add the build script as a dependency. This way, your Build Script code will always get run before your target starts to get built.

---

# Run Script

Writing your Build Script is pretty straightforward, but there are a couple of things that you should do to make things easier.  For example, you may need to know which environment your build script is running in, so we create a User-Defined variable called XXX_ENVIRONMENT.  I suggest that you create this at your Project level, not your target (and let the Xcode build setting inheritance do its magic). Otherwise, if you add target's later, you'll need to go and re-add this variable to each target.

So, as you can see in the screenshot, I've added a variable called XXX_ENVIRONMENT and set it to the appropriate value.  Then as one of the first things I do in the build script is to pull that out and set a shell variable called env.  We can use this value later.

Similarly, I set a variable to the name of the Info.plist preprocessing header file that we referenced earlier. This is so that we can write variables a bit later.

---

# Bundle Identifier

So, clearly, in order to support multiple apps, we'll need different bundle identifiers.  As you know, the bundle identifier is in the Info.plist so we'll need to modify that setting.

Now, you'll recall I mentioned that we can use the Info.plist preprocessing to substitute variables in.  But, we can also use Build Setting variable substitution.  In this particular case, though, we have no choice and must use the Build Setting substitution because Xcode does some validation (checking code signing certificates) very early on and it occurs even before our Build Script has had a chance to run.

As a quick sidebar, my general rule of thumb is to use the Build Settings where the values are simple and don't need to be derived or calculated in any way. 

I must admit I am torn as to whether the production bundle id should have the trailing .PROD. In one sense, I like the symmetry of having the .PROD (and if I am grepping logs, it is a bit easier) but on the other hand, it is sort of leaking meta info about our build process.  

---

# Bundle Identifier (cont)

It is literally as simple as setting the bundle identifier in the Info.plist.

---

# Setting App Icon

Now that we have multiple app identifiers, let's give them each a different icon.  The first thing you'll need to do is modify your Asset Catalog and create an AppIcon entry for each environment.

And then, it is as simple as modifying the App Icon Set Name field for each configuration to reference the appropriate entry.

If you aren't using Asset Catalogs, you just need to substitute the correct image name into your Info.plist file (similar to what we did for the bundle identifier).

---

# Setting App Name

At a minimum, we need to could just set the name using the build settings like we did with the bundle identifier. However, we wanted to be able to tell how current the app is. We toyed with a few other indicators, but settled on the day of the month so as you can see in the screenshot, this guy is running a SYS app built on the 23rd.

To do this, it is a pretty simple modification to the Build Script.

---

# Setting App Name (cont)

The first thing we need to do is change the Info.plist to reference the variable.  Note that it doesn't use the dollar/curly brace notation because it is substituted in using the preprocessor.

Then, in our build script, we default the appName variable to your real app name, and then if we are building for a non-prod environment, we change it to contain the environment name and the day of the month.

Lastly we write this out into the preprocess file that we defined a little bit earlier.  And we're done!

---

# Version Number

CFBundleShortVersionString is what the user sees

CFBundleVersion is supposed to be an ever increasing number that represents your build number. Why Apple couldn't choose slightly saner names beyond me.

So, obviously, CFBundleShortVersionString gets set to 1.2 (or 2.4 or whatever), but the question is what we put into CFBundleVersion.  We have two different approaches for non-production and production.  

In non-prod, we put the current git commit hash. This helps us in a couple of ways. 1/ Both TestFlight and Hockey conveniently display in their download page, and 2/ when we dump logs, we have a very easy way that a developer can determine which build a tester is using.

Production is a different matter though. We can't use the commit hash because it has to be numeric, so the question is what to use. I see that a few people on the web are proposing to use the git commit count.  The problem I have is that value doesn't really contain any significant meaning.  I mean, as a developer, I suppose I can use git to go back and retrieve the n'th commit but it strikes me as a bit icky.  Also, not that I expect it to occur very often, git history can be altered using rebase and squash which will affect the count.  In any event, it isn't as important in production because we will know what build the user is using because of the CFBundleShortVersionString.  We ended up just putting the same value as is in CFBundleShortVersionString. As far as I can tell, it isn't visible to the end user and it seems to work fine.  I'd be interested to hear if this is a flawed strategy, or about any other techniques.

---

# Version Number (cont)

Alright, so how do we make this happen? Again, we modify the info.plist to contain a reference to a variable name.

In our build script, we set the value to the same as the CFBundleShortVersionString and then overwrite it in the non-production environments with the commit hash.

---

# Application Settings

Up until now, we've mostly been dealing with meta data about the app. What about configuration settings that the app requires to operate? Some really basic ones are ...

---

# Application Setting Requirements

There were a couple of key things that, for me, are a must. Firstly, we should never let our non-prod settings get shipped into our production application. Secondly, a developer shouldn't have to go in an touch a file to switch, and neither should the build leave files lying around that causes git to think I have made changes.  And, speaking of git, all of our settings should be checked into source control so that everybody (developers and our build box) all use the same settings.

---

# Environment .plist files

So, the basic gist of this is that the app is going to read its settings from a file called environment.plist, which is injected into the bundle at build time by the build script.

How does this work in practice? We create a plist per environment, but the key is that your don't add them to your target. This stops these files getting shipped in your app.  Then you touch an empty file called environment.plist that you do add to your target, but you also add it to your gitignore file.  And lastly, your build script simply copies the relevant plist over the top of the empty environment.plist.

Really simple.

---

# Maintaining Environment

Now, reading the settings is really simple... you just read from the environment.plist just you would any other plist in your bundle.

A thought that will occur to you about 5 minutes after you're live with this code is that it would be really useful to be able to change these settings.  What we did is create a Configuration object that reads in the data from the environment.plist on startup, and then kicks off a background thread to go see if any of the settings have changed.  If so, the configuration object is updated with the new settings.

Note: There is a little bit of a race condition between when the app starts with the out-of-box settings and a short while later when it receives the updated settings.  A future enhancement we're looking at is to store the changed settings in NSUserDefaults so that the app always runs with the most recent settings.

---

# Continuous Integration

All of the things I've talked about work just fine with the normal CI build process. We just specify the scheme and configuration on our jenkins server, and it just builds in the same way it does on the developer's machine

Almost done... a couple of closing remarks.

---

# Logging

When things go wrong, dumping stuff out via NSLog is of little value to our production support... it isn't as though the user is going to send their phone in to hook up to Xcode.

We added a feature in the Contact Us section where the user can press a button, and it sends us through the logs, and gives them back a reference number that is generated by our server. This serves two purposes: 1) in production it gives our prod support team a chance to see what is happening, and 2) during testing, when a tester finds a bug they click on the same button, and add the reference number to the defect.

Logging levels are a bit of a double-edged sword. During development, it is fantastic to have the app logging at DEBUG level. This gives our developers everything they need.  However, in the production app, you won't (or at least, you shouldn't) have debug logs turned on. So, you need to make sure that the info that gets logged out at production log setting is sufficient for your prod support guys to figure out what happened.

Lastly, a quick plug for BDLogger. This is an open-source project that I wrote for my own stuff, and we are using it our app. It logs to a very small sqlite database, purges old records, and gives easy query access when you want to send the logs back.

---

# Going Live

So, you've submitted your app to Apple, it has been approved and you're ready to go live. It doesn't matter how much testing I've done prior, I'm always nervous at this point. A technique that we've been using to mitigate some of this nervousness is to use promo codes before the app is released.

For those of you who haven't used promo codes, they are a neat feature that Apple has that allow you to give a free copy of your app to journalists, friends, family, customers, whatever.  Apple documentation says that it is only available when the app is live on the store, but that isn't actually true.

When we submit the app for approval, we say that we will control the release date and set it to something about a month in the future. Once it is approved and waiting for us to release it, you see a screenshot that looks a bit like that.  You'll notice that the promo code button is enabled. We normally create a couple of promo codes, and give them to a very limited set of testers (usually it is me and our head tester) and we can give it a quick shakedown in prod before releasing to the public.

---

# Android

And I know this isn't AndroidHeads, but for those of you who support Android, all of the techniques here work almost exactly the same in the Android world.

The Android Manifest already supports variable substitution so that is easy. The environment.plist file is replaced by an environment.xml file that ends up being compiled into R.java values, and all of the build script functionality moves to being ANT targets.

The only real catch is supporting multiple app identifiers. For a long time we couldn't do it (all of the options I'd come up with were way too high risk) until one day I was complaining to our best Android guy about 
