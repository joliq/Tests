# VF Portal application

Feel free to update this with notes and important items regarding the solution.

---

## VF.Portal.SPApp
This is the default SharePoint app project needed to deploy the application. 

### Versioning
Use semantic version scheme. (http://semver.org/)

Given a version number MAJOR.MINOR.PATCH, increment the:

1. MAJOR version when you make incompatible API changes,
2. MINOR version when you add functionality in a backwards-compatible manner, and
3. PATCH version when you make backwards-compatible bug fixes.
5. Additional labels for pre-release and build metadata are available as extensions to the MAJOR.MINOR.PATCH format.

### Permissions
The needed permissions are as of now FullControl of the sitecollection where the app is installed.
This is because the app will need to deploy files to  the site colelction as well as configure it.

### Prereqs
None at the moment.

### Events
The app is configured to use all three app events:
1. Installed 
2. Uninstalling 
3. Upgraded 

The events should be used to deploy functionality and clean up deployed artefacts.

The event receiver should be located at this url: 
~remoteAppUrl/Services/SP/AppEventReceiver.svc

### Deployment
Run Go.cmd to generate deploy scripts using the specified deploy configuration.
Then run ReDeploy.ps1 from Localbuild folder.
Alternatively you can run Redeploy.cmd which trigger both Go and Redeploy.
The ReDeploy command runs Provision-VFPortal from Deploy-Utility.ps1.
Provision-VFPortal enables site collection creation from CSOM on the Farm where the script is being run.
This means that Enable-OnPremSiteCollectionCreation needs to be run on the SP farm.

The script then goes on to create a new site collection and asks to remove a pre-existing site if it already exists.
Then the provider-hosted app (VF.Portal.WebApp) gets deployed to IIS. This requires the script to be run on the prodivder hosted app server.
Then the script creates a new navigation termset and termgroup and adds a sample term.

If you dont have the ability to run the deploy scripts then these are the manual steps needed.

1. Create a new site collection (Publishing portal template)
2. Register the app using _/layouts/15/appregnew.aspx
3. Deploy the WebApp project to IIS. 
Make sure anonymous authentication is enabled on the site.
Make sure the app-pool service account has write permission on a folder called logs (if not exists, create it) in the root folder.
4. Upload the app package to the app catalog.
5. Create a new navigation termset and term group. The termset ID must be equal to: 4a4d7657-7163-40b5-80b7-e392c14d1e4d
If you have permission to the termstore you can create the termset in the browser using JSOM:
```javascript
var context = SP.ClientContext.get_current();
var taxonomySession = SP.Taxonomy.TaxonomySession.getTaxonomySession(context);
var termStore = taxonomySession.getDefaultSiteCollectionTermStore();
// if the group does not exist, create it first by using this:
// var group = termStore.createGroup("VF Portal", "TERM-GROUP-GUID");
var group = termStore.getGroup("TERM-GROUP-GUID");
context.load(group);
context.executeQueryAsync(Function.createDelegate(this, function (sender, args) {
    group.createTermSet("Global Navigation", "4A4D7657-7163-40B5-80B7-E392C14D1E4D", 1033);
    group.createTermSet("Toolbox", "5e0e2342-e8e1-4385-b272-4c7a46e5acd8", 1033);
    context.executeQueryAsync(Function.createDelegate(this, function(sender, args) {
        alert('Succeded: ' + args.get_message());
    }), Function.createDelegate(this, function (sender, args) {
        alert('The error has occured: ' + args.get_message());
    }));
}), Function.createDelegate(this, function (sender, args) {
    alert('The error has occured: ' + args.get_message());
}));
```
6. Install the app in the site collection.
---

## VF.Portal.WebApp
This is the main web application responsible for deploying artefacts (scripts, css etc) to SharePoint.

The webapp has been set up as default empty ASP.Net web application configured to be used for Web API. This is because this solution is designed to work as a client-consuming and server-serving solution.
The client in this case are AngularJS scripts that will request data from our API and display the data in a certain manner.

The API will in turn request data from different sources such as external LOB systems, databases etc.

### .bin
Folder where cmdline utilities exist for node, npm and git exists. Used for the Gulp tasks.

### AppScriptParts
Use this folder to store .webpart-files used for deploying app-script-parts which is basically a scripteditorwebpart but you set the Content property to load your specified javascript.

### Branding
Use this folder to store CSS files or other branding artefacts.

### Scripts
Main folder for scripts.

### Models
Main folder for all POCOS and models.

### Repository
Keep data access classes here.

#### Landlord
The repository pattern here have been modeled to mimic the functionality in MSFTs entity framework. The reason for this to make it testable.


Create an object-url for the Landlord object in Models.Landlord.Constants.
Add a public property using RESTDataSet to LandlordDatacontext. 
Initialize the RESTDataSet in the constructor of the context using Set(url).

Usage:
```c#
using (var landlordContext = new LandLordDataContext("url to landlord api", true/false))
{
    var buildings = landLordDataContext.Buildings.Items;
    var building = landLordDataContext.Buildings.FindByGuid(b => b.guid.ToString().Equals(buildingGuid), buildingGuid);
    // Do something with the buildings here.
}
```

The repository api has been created with the principle to minimize calls to the api as much as possible. That is why there's an "innerList" of items that gets populated after the first call to get items from Landlord.
The "innerList" also gets populated with items if the LandLordDataContext is instanced with "preLoad = true". This is useful when you need to work with all items from Landlord but want to minimize the calls to Landlord. 
The "preLoad"-parameter makes calls to get all Building and Management items with as few calls as possible when the context is instanced.

### Services
Main folder for web services.

### Services/SP
Keep SharePoint specific services in here.The app event receiver resides here.

### Services/SP/Search
This folder has definitions for the services that expose a wcf endpoint for use with BCS and search.

### Utilities
Main folder for various utilities.

### gulpfile.js
This is the GulpJS file where we specify specific build tasks to be performed using gulp and node.js (http://gulpjs.com/).

Current configured tasks are clean, lint and browserify.

* lint - uses jshint to style check javascript files (jshint rules specified in .jshintrc file)
* clean - removes old files from build output
* browserify - uses browserify (http://browserify.org/) to bundle script files and dependencies following the require('module') standard. Allows for very cool stuff..
* less - builds css from less files
* build - runs clean, lint, browserify and less, with a command line option `mode` to set Release (this means js is minified)
* watch - automatically rebuilds when files change, useful when working with js or less

Currently scripts from the 'Scripts' folder are being linted and bundled to 'dist/js/_layouts'. Thats the output to be deployed.
Note: The "_layouts" folder needs to be present and called "_layouts" if SharePoint are to be able to register and add scripts using the JavaScript injection pattern from OfficePnP.

### package.json
Used by NPM to download gulp, scripts and libraries (such as angularJS).

### package.config
Used by NuGet to download packages such as JSON.Net, NodeJS, NPM etc.

### web.config
SharePoint specific app settings in web.config. These are replaced using publishingprofiles when building and packaging.
*ClientId
*ClientSecret
*ClientSigningCertificatePath
*ClientSigningCertificatePassword
*IssuerId

These are project specific app settings that are used for calling the correct API URL as well as creating sites in the correct site collection.
API URL for Landlord
*VF.LL3.APIUrl -  example value: https://weblord3.vgregion.se/api/v1/

URL for building site collection
*VF.Portal.BuildingSiteCollections -  example value: https://dev.booldevlocal.local/sites/vfp-buildings"

### VF.Portal.WebApp.MSBuild.targets
MSBuild file to include gulp and NPM build steps in the normal MSBuild procedure.
This is used to collect the output from the gulp tasks and include it in the MSBuild packaging process.

## VF.Portal.WebApp.Tests
This is the main project for tests related to the web app project.

The structure is set up in a similar order as the WebApp project. One Test class per Test__ed__ class.

### Fakes
Due to the fact that SharePoint is difficult to test with unit tests Microsoft Fakes are being used to fake the entire assemblies for:

* Microsoft.SharePoint.Client.fakes
* System.ServiceModel.fakes

This way we get Stubs and Shims to fake a ClientContext and remove the requirement of having an actual SP environment for unit testing.

### Utilities
Testing of all Utilities are located here.

### app.config
Use this if we need to store configuration variables for URLs, credentials etc.

### Resources
Sample files and other resources needed for tets are included here.

### Logging
Logging is being made with log4net. Configuration is located in web.config.
There are 2 loggers to use. "Server"-logger outputs to server_log.txt and is used for logging in .Net-code.
"Client"-logger outputs to client_log.txt and is used for logging errors in JavaScript on the server.
To log a message to the clinet log, use the web api method "Error" and post a message using the ClientError model.

## VF.Portal.ScheduledServices
This is a console application intended to run as a scheduled task 
to synchronize Building objects form external source LandLord to SharePoint sites.

### app.config
URL for building site collection
*VF.Portal.BuildingSiteCollections -  example value: https://dev.booldevlocal.local/sites/vfp-buildings

Name for the IIS site where the app is hosted.
VF.Portal.IISSiteName -  example value: VFPortalApp

API URL for Landlord
*VF.LL3.APIUrl -  example value: https://weblord3.vgregion.se/api/v1/


##Authentication
Add the oauth client in ADFS
Add-AdfsClient -ClientId "cd2c76fc-01c5-4c33-a287-46a53f4c57c3" -Name "VFPortal" -RedirectUri "https://vfportalapp.booldevlocal.local/home/authenticate"

Enable refresh tokens
Set-AdfsRelyingPartyTrust -TargetName "VF Portal" -IssueOAuthRefreshTokensTo AllDevices
Set-AdfsRelyingPartyTrust -TargetName "VF Portal" -TokenLifetime 10
#Set-AdfsProperties -SSOLifetime 480

###Warmup
Warmup can be performed using the supplied warmup script.
This script needs to be run with a service account that has READ-permission + Browse directories on site collection level for all site collections.
The warmup script works by opening up IE, atuhenticates against the ADFS service and then browsing to all sites through IE.
 