# Dynamic versioning of an Atlassian Connect app
Developing a production-ready app based on the Atlassian Connect framework is not something you'd find a lot of tutorials on, so I want to take a second to describe how we do versioning of our Connect app and why we chose to do it this way.

## What we started with
When we started, we were all in on the concept of shipping a SPA, without much server-side logic, or even rendering. You can easily get an user identity token (AP.context.getToken()) in the frontend, so it's really not necessary to do any backend validation. Therefore, for a very long time, our apps Connect manifest simply looked something like this

```json
{
    "name": "Microsoft 365 for Jira",
    "key": "com.yasoon.jira.cloud",
    "baseUrl": "https://atlassianconnect.yasoon.com",
    "authentication": {
        "type": "jwt"
    },
    "lifecycle": {
        "installed": "/v2/atlassian-connect/lifecycle-v2?type=installed&version=5.9.6",
        "uninstalled": "/v2/atlassian-connect/lifecycle-v2?type=uninstalled&version=5.9.6",
        "enabled": "/v2/atlassian-connect/lifecycle-v2?type=enabled&version=5.9.6",
        "disabled": "/v2/atlassian-connect/lifecycle-v2?type=disabled&version=5.9.6"
    },
    "enableLicensing": true,
    "modules": {
        "postInstallPage": {
            // Static HTML file below!
            "url": "/jira-bundle/v5.9.6/index.html?view=gettingStarted",
            "key": "postInstall",
            "name": {
                "value": "Getting started"
            }
        }
    }
}
```

You can see that we hard-coded the view-URL of the module to point to a fixed version of of our SPA. This has a few benefits:
1. You don't need to care about caching issues. We experienced lots of caching problems from browsers & AWS Cloudfront along the way, which we simply worked around by providing a fully-versioned URL slug for each deployment. This has some drawbacks, especially higher client load times if you do release very often, but with proper code-splitting/chunking, this is somewhat negible and we consider this an acceptable trade-off.
2. When you keep you atlassian-connect.json file in a fixed, non-versioned location (e.g. https://atlassianconnect.yasoon.com/jira-bundle/atlassian-connect.json), the Atlassian Marketplace will take care of auto-updating all the instances in a rolling manner. This takes around 24h-48h, after that, all instances will be updated to the new version automatically.
3. It makes it easy to install an older version of the app by simply using an older version of the manifest, which will point to the correct resources.

There are however a few downsides with this approach:
1. You are completely reliant on Atlassian for delivering version updates to customers. There is no simple possibility to deliver a critical bugfix earlier, besides having the customer enable development mode and manually uploading a custom manifest, which is somewhat clumsy and disruptive.
2. Additionally, shipping an important bugfix or security update for every customer will take at least 24h - this might not be an acceptable timeframe for some issues.
3. There was no way to already to a server-side validation of certain properties, e.g. issue access or admin permissions of the calling user. This adds an additional overhead in the SPA which might introduce a delay in the hot-path, making the UI slow to load.

After observing these issues for quite a while, we set out to built a better version for us, and achieve some additonal niceties along the way. 

## The "loader" based approach
Our new manifest now looks like this for most modules:
```json
{
    "name": "Microsoft 365 for Jira",
    "key": "com.yasoon.jira.cloud",
    "baseUrl": "https://atlassianconnect.yasoon.com",
    "authentication": {
        "type": "jwt"
    },
    "lifecycle": {
        "installed": "/v2/atlassian-connect/lifecycle-v2?type=installed&version=5.9.6",
        "uninstalled": "/v2/atlassian-connect/lifecycle-v2?type=uninstalled&version=5.9.6",
        "enabled": "/v2/atlassian-connect/lifecycle-v2?type=enabled&version=5.9.6",
        "disabled": "/v2/atlassian-connect/lifecycle-v2?type=disabled&version=5.9.6"
    },
    "enableLicensing": true,
    "modules": {
        "webItems": [ {
			"key": "teams-share-dialog",
                        // Backend endpoint instead of static HTML file!
			"url": "/v2/atlassian-connect/render?view=teams-share-dialog&version=5.9.6&issueId={issue.id}",
			"location": "jira.issue.tools",
			"context": "addon"
		} 
	}
}
```

The most important part is now, that the URL used by the module is not not versioned anymore, but points to a service in our backend, while still providing the Connect manifest "version" via a query parameter. This allows us to do a few neat things.

### Dynamic version determination
In the backend service, we now do a few different things. The most basic logic, when there are no exceptions defined, is rendering a HTML page with the version that was provided as a query parameter. So the happy path is basically the same as before, but introducing the possibility to do more. As Atlassian Connect provides an additional query parameter called jwt for each call, you can now do a simple version determination based on the user, the instance or any other criteria you like. We introduced the following options, by using a simple, single version database table, which is key-value only. As it's a single table, we can do a select with three or four different deployment keys at once, and then have a prioritized order in which they are evaluated. 

The last part of this is then updating the version entries in the database from your CD pipelines, to automate version releases.

#### 1. User based version
The first criteria that is being evaluated is the user + instance, which allows you to target certain users with a specific version of the app. We use this mostly internally, to let developers activate specific PR builds in a single PR test system. We have a button in our PRs that says "Activate PR in test system", which will write a database entry for the current reviewer. When opening the PR test Jira instance, all app entrypoints will point to the version from the PR. This allows us to quickly validate & review UI changes without checking out and building the code locally.

The database entries follow the following format
`clientKey_userAccountId`
| deploymentKey | version |
| --- | --- |
| 217f4fed-99c3-3aa7-9e13-ac6c4707d6b4_5e0df022b783d60db09fe120 | v_PR_655 |
| 217f4fed-99c3-3aa7-9e13-ac6c4707d6b4_3714821f-d783-3a2c-b468-02e86d627cb6 | v_PR_786 |

#### 2. Instance based version
The second criteria that is being evaluated is the instance itself - we now can have a certain frontend version that is depedent on a specific customer, e.g. to ship certain bugfixes earlier, for all users in that Jira system. The DB entries look something like this, where the guid represents the clientKey of the system. It works a bit different in our production environment but that is the basic idea behind it.

| deploymentKey | version |
| --- | --- |
| 217f4fed-99c3-3aa7-9e13-ac6c4707d6b4 | v6.0.0 |
| 49544f88-8709-354e-b152-a70a9fa7db94 | v5.9.10 |


#### 3. Ring based version
The database entries for our ring based approach look something like this. These will be taken into account for instances where no explicit version is set (see above).

| deploymentKey | version |
| --- | --- |
| ring_internal | v6.0.0 |
| ring_fast | v5.9.10 |
| ring_stable | v5.9.0 |

Ring-based deployment, currently has three "rings": Internal, fast & stable. For each installed app in a site, we store which ring the instance is on: Internal instances are our own "production" Jira sites, fast are non-paying customers and stable are all our paying customers. For each ring, we store which version should be served, and in each "/render" call, we determine the correct version on the fly. The versions for each ring are updated from our CD pipelines. Once a change/PR is merged to main, our internal systems immediately receive the new version, by updating the `ring_internal` entry. Each monday, the latest version would be set for the `ring_fast`. After another week, we will release the new version of the atlassian-connect.json to the Atlassian Marketplace, to update ring_stable.

### Side benefit: pre-supply data to the UI
Using a backend endpoint also comes with some additonal benefits. If you already know which view is being loaded, you might be able to either server-side render the view, or at least already bootstrap the UI with some required information, like settings from your database. This allows to render the initial SPA UI faster, because less backend calls need to be done in the hot path.

### Downsides
As always, it's not only sunshine and rainbows with this solution. We have identified the following trade-offs for this approach, which are acceptable to us, but might not be to you.

- Having a server-based loader means one more blocking call before anything can happen on the UI. Simply loading a HTML file directly from a Cloudfront cache will always be faster than to have a backend service process the request first

- In case of manifest changes (e.g. additional scopes), you'll need to make sure to hold dependent versions back, until all customer instances have updated to the latest manifest version (as in the "before" model) 

- Dependency between backend releases & frontend versions needs to be coordinated. You can't have a frontend version PR merged until the backend service supports the frontend change. In general, you'll need to think about a compatibility matrix more, if you muck around with dynamic versioning.

- Communication about new features to customers get’s trickier, if you release features / fixes not at the same time for everyone. We heavily rely on the Jira deployment tab in our Software project to allow everyone to check which issue change is live for which ring.

