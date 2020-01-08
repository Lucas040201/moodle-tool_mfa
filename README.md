<a href="https://travis-ci.org/catalyst/moodle-tool_mfa">
<img src="https://travis-ci.org/catalyst/moodle-tool_mfa.svg?branch=master">
</a>

# moodle-tool_mfa

* [What is this?](#what-is-this)
* [Why another MFA plugin for Moodle?](#why-another-mfa-plugin-for-moodle)
* [Branches](#branches)
* [Installation](#installation)
* [Configuration](#configuration)
  * [Factors](#factors)
    * [IP range](#ip-range)
    * [TOTP](#totp)
    * [Auth type](#auth-type)
    * [Non-admin](#non-admin)
    * [Email](#email)
    * [Other factors](#other-factors)
* [Points and examples](#points-and-examples)
* [Debugging](#debugging)
* [Support](#support)

## What is this?

This is a Moodle plugin which adds Multi-Factor authentication (MFA), also known as Two-factor authentication (2FA) on top of your existing chosen authentication plugins.

https://en.wikipedia.org/wiki/Multi-factor_authentication

## Why another MFA plugin for Moodle?

There are other 2FA plugins for moodle such as:

https://moodle.org/plugins/auth_a2fa

This one is different because it is NOT a Moodle authentication plugin. It leverages new API's that Catalyst specifically implemented in Moodle Core to enable plugins to *augment* the login process instead of replacing it. This means that this MFA plugin can be added on top of any other authentication plugin resulting in a much cleaner architecture, and it means you can compose a solution that does everything you need instead of compromising by swapping out the entire login flow.

See this tracker and the dev docs for more info:

https://tracker.moodle.org/browse/MDL-66173

https://docs.moodle.org/dev/Login_callbacks

The other major difference is that we support multiple authentication factor **types** as sub plugins, eg IP Range, Email, TOPT and in future others such as SMS or hardware tokens or anything else as new sub-plugins. They can be flexible configured so that different combinations of factors are considered enough.

## Branches

`master` is considered stable and supports these versions, with the caveat of backporting the API's needed.

## Installation

Step 1: Install the local module
--------------------------------

Using git submodule:

```
git submodule add git@github.com:catalyst/moodle-tool_mfa.git admin/tool/mfa
```

OR you can download as a zip from github

https://github.com/catalyst/moodle-tool_mfa/archive/master.zip

Extract this into /var/www/yourmoodle/admin/tool/mfa/

Then run the moodle upgrade as normal.

https://docs.moodle.org/en/Installing_plugins


Step 2: Apply core patches
-------------------------------

This plugin requires [MDL-60470](https://tracker.moodle.org/browse/MDL-60470) which was only added 3.7, and [MDL-66340](https://tracker.moodle.org/browse/MDL-66340), which was added in 3.8.

You can easily backport these patches in one line for 3.5, 3.6 and 3.7:

For Moodle 3.5:

```
git apply --whitespace=nowarn admin/tool/mfa/patch/core35.diff
```

For Moodle 3.6:

```
git apply --whitespace=nowarn admin/tool/mfa/patch/core36.diff
```

For Moodle 3.7:

```
git apply --whitespace=nowarn admin/tool/mfa/patch/core37.diff
```

### Manual cherry-pick
In case the patches do not work due to an update to older Moodle branches (such as security updates), you can manually perform the cherry-picks.
For [MDL-60470](https://tracker.moodle.org/browse/MDL-60470):

```
git cherry-pick bf9f255523e5f8feb7cb39067475389ba260ff4e
```
If there are merge conflicts, ensure the lines that you are adding are consistent with the lines being added inside the patch files. Everything else can safely be ignored.

For [MDL-66340](https://tracker.moodle.org/browse/MDL-66340):

```
git cherry-pick 4ed105a9fd4c37e063d384ff155bd10c3bfbb303
```
As with above, if there are merge conflicts, ensure the lines that you are adding are consistent with the lines being added inside the patch files. Everything else can safely be ignored.

Once this has been performed, you can generate your own patch files using `git format-patch`. An example for Moodle 3.5 is below:
```
git format-patch MOODLE_35_STABLE --stdout > admin/tool/mfa/patch/new_core35.diff
```

## Configuration

WARNING: Do not try to fully configure this plugin in the web GUI immediately after installation, at this point during the upgrade process you are not actually logged in so it is easy to 'brick' your moodle and lock yourself out.


### General settings

* Grace period - This is important as some configurations of factors won't allow you to login. So you can optionally set a grace period where people can login and setup their other authentication factors


### Factors

The main concept to understand is the concept of factors. Each user must satisfy some combination of factors which sum to 100 points in order to login. By configuring multiple factors and weighting them you can easily have quite complex and flexible rules.

#### IP Range

Use this factor to say that if you are on a secure network then that counts for something. This is very useful because it requires no setup by the end user, so you can set it up so that you can login fully via a secure network, and once logged in they can setup other factors like TOTP, and then use those other factors for logging in when not on a secure network.

#### TOTP

This is standard TOTP using Google Authenticator or any other app which conforms to the open standard.

#### Auth Type

This is so you can specify that logging in via say SAML via ADFS which may have already done it's own MFA checks and is worth 100 points effectly make it exempt from additional checks.

#### Non-admin

This factor enables you to give points for free to a user who is not an admin. This makes it easy to require admin users to have 2 or more factors while not affecting normal users.

#### Email

*** Not recommended for production use ***

A simple factor which sends a short lived code to your email which you then need to enter to login. Generally speaking this is a low security factor because typically the same username and password which logs you into moodle is the same which logs you into your email so it doesn't add much value.

This factor was implemented as a proof of concept of a factor which can return a hard FAIL state, ie positive evidence that your account is compromised rather than NEUTRAL where we simply lack evidence of additional factors that the end user is who they say they are.

#### Grace Mode
The gracemode is a factor to allow users to log in without interacting with MFA for a set period of time. Users can only achieve the points for this factor if there are no other input factors for them to interact with during the login process. This factor should be placed last in the list, that way all other factors are interacted with during the login process first. On the first page after login, if a user is currently within their grace period, regardless of whether they used gracemode as a login factor, they are presented a notification informing them of the grace period length, and that they may need to setup other factors or risk being locked out once the grace period expires.

#### No Setup Factor
This factor is designed to allow people to pass only if they have not setup other factors for MFA already. Once another factor, such as TOTP is setup for a user, this factor no longer gives points, therefore the user must use TOTP to authenticate. This allows for an optional MFA rollout, where only users who wish to use MFA are affected by the MFA rollout.

#### Other factors

In theory you could impement almost anything as a factor, such as time of day, retina scans, or push notificatons. For a list of potential factor ideas see:

https://en.wikipedia.org/wiki/Multi-factor_authentication#Authentication_factors

### Points and examples

If a users cumulative points is high enough then they are able to login. Points can be weighted for different factors. Some factors do not require any input, such as checking their IP Address is inside secure subnet, while other factors require input such as entering a code like TOTP or SMS. Factors with no input are checked first and then the remaining factors are checked in from the largest points to the smaller until you either have a cumulative points high enough to login, or you run out of factors and you are denied login.

When you configure the points in the admin settings it will generate a list of valid factor permutations to easily check it's configured the way you want.

#### Example 1

If you have 3 factors configured, all factors default to 100 points effectiely making any of then enough to login:

```
auth=saml => 100
iprange => 100
totp => 100
```

Then it will show:

```
You must be:
* has an authentication type of saml
OR
* is on a secured network
OR
* using a TOTP app
```

#### Example 2

If you change all 3 points to 50 then it would say:

```
You must be:
* has an authentication type of saml AND is on a secured network
OR
* has an authentication type of saml AND using a TOTP app
OR
* is on a secured network AND using a TOTP app
```

#### Example 3

With a configuration of:

```
auth_saml => 100
iprange => 100
totp => 100
email => 50
security_quesstions => 50
```

Then these are valid conditions:

```
You must be:
* has an authentication type of saml
OR
* is on a secured network
OR
* using a TOTP app
OR
* has valid email setup AND answers security questions
```

## Debugging

While you are setting up MFA there are 2 things which help make it simple to see what is going on:

1) In the settings page is a 'Summary of good conditions for login' which does what it says on the box. If you have not setup any factors, or if they are configured in a way which would never all login then it will warn you.

2) You can turn on debug mode, when you are logging in and stepping through the MFA login flow if will show you the list of factors and how they have been resolved.

If you have inadvertantly messed things up and locked yourself out, you can disable the whole MFA plugin from the CLI:

```sh
php admin/cli/cfg.php --component=tool_mfa --name=enabled --set=0
```

## Support

If you have issues please log them in github here

https://github.com/catalyst/moodle-auth_saml2/issues

Please note our time is limited, so if you need urgent support or want to
sponsor a new feature then please contact Catalyst IT Australia:

https://www.catalyst-au.net/contact-us

This plugin was developed by Catalyst IT Australia:

https://www.catalyst-au.net/

<img alt="Catalyst IT" src="https://cdn.rawgit.com/CatalystIT-AU/moodle-auth_saml2/master/pix/catalyst-logo.svg" width="400">
