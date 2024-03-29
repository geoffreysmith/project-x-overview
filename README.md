# Sitecore Project X Overview
> 12/21/2019

**The Project** currently runs on Sitecore 9.1, to run it simply run the [docker-compose.yaml](docker-compose.yaml) if you have the correct azure credentials. After an hour of downloading you will correctly have an environment running the bare minimum of Sitecore 9.1 XC taking 5GB a piece per server, and this is a development topology. There are many Sitecore variants and topologies, which change all the time. This is very important because the marketing terms change between versions. And if you read any documentation sometimes the names change or refer to an early version. Here's what the automated build accomplishes and a brief overview of which server differs by config:


```text
 sitecore91.azurecr.io/sitecore-xc-cm:9.1.0-windowsservercore-1903-v1.3
 \__________________/ \_______/ \___/ \____/\_______________/\___/\___/
           |              |       |     |            |         |    |
        ACR URI       sitecore server sc version core/nano  kernel custom tag based off latest release (latest git tag, will not push to ACR without a tag)
```


## ACR URI

A separate devops repository has each README for the creation of all services needed to bring the entire environment, anything linked in this document has been scrubbed of client information. To build the environment scratch entails the following:

1. Setup a a custom Azure build server (Self Hosted Build Agent)[SELF-HOSTED-AGENT.MD], this is in its own repository.The build exceeds the the standard Azure build pipeline as the limit is 10 GB per build. A ```--no-cache``` build takes around 3 hours.
2. To enable the build of the images from a hosted Azure build server, service connection between Azure Build Pipelimes and ACR needs to be created, this is documented. It runs off a service principal.
3. Azure git requires a similar setup and is documented in the devops repo, it does not need to be redone but is dodcumented.
4. A custom Azure build pipeline that builds off a push off master is in place for the SitecoreBuild repo. It encompasses rolling updates to the base Windows Server updates and is transparent. This is automatic and documented in a separate README with the build server repo.

**Sitecore**

> Convention is to put Sitecore in repository names, I simply followed conventions. [Nuke Build](https://nuke.build/) takes care of tagging of build images.

**Server (And Base Build)**

 ***Sitecore 9.1 XC XP0***

| Server             | Shared Config  | Separate Code  | Custom Code?    | Explanation															 |
| -------------------|----------------|----------------|-----------------|-----------------------------------------------------------------------|
| xc-cm              | N (CM/CD)      | Shared (CM/CD) | Shared (CM/CD)  | Authoring environment      											 |
| xc-cd              | N (CM/CD)      | Shared (CM/CD) | Shared (CM/CD)  | Delivery environments                                                 |
| xconnect           | N              | N              | N               | No idea what this does but it requires separate environment configs   |
| solr               | N/A            | N/A            | N/A             |																		 |
| mssql              | N/A            | N/A            | N/A             |                                                                       |
| identity server    | N              | Y              | Y               | IdentityServer4 this is broken OOTB                                   |
| xc-plumber         | N              | N              | N               | Commerce debugging site                                               |
| xc-authoring       | N              | Y              | Y               | Where users and plugins edit commerce "entities" that go to Sitecore  |
| xc-shops           | N              | N              | Y               | Where users create shops (we just have one)                           |
| xc-ops             | N              | Y              | Y               | Backplane for the commerce servers                                    |
| xc-minions         | N              | Y              | Y               | A daemon to run jobs like "update inventory"                          |
| xc-identity server | NO IDEA        | NO IDEA        | NO IDEA         | 99% sure this proxies dotcore objects to dotframework objects         |

1. Terminology: This is confusing and the documentation provided by Sitecore contradicts itself. There's multiple setups and topologies. In general for builds environments there is:
* XP (CMS only)
* XM (Standalone, no CD/CM)
* XC (Commerce)
> Ignore the term XM but this build is XM since we have just an authoring environment and not delivery node as the only difference is config and not code. For topology this gets even more complex as the same acronyms are used even if it applied to XC:
* XP0
* XP1
> Ignore the fact the term is "XP" as it applies to XC also. XP0 doesn't break apart the services as much as XP1 does. See [the Azure guide](https://doc.sitecore.com/developers/90/sitecore-experience-manager/en/sitecore-configurations-and-topology-for-azure.html). The project runs XC on an XP0 topology for dev, and for production will run XC with an XP1 topology. The difference is not in code but how configs are broken out and how services are on separate servers.

2. Everything needs certs to talk to each other over SSL and they each need thumbprints from each other to validate themselves. This all automated in the build, there are self signed certs.

3. Sitecore is installed in a non-Docker setup with a complex powershell framework called SIF (Sitecore Installation Framekwork), and it is buggy and doens't really work. It requires localhost to install all these IIS/servers services. The docker build process actually uses this to not deviate from conventions and eliminate discrepencies from Docker and SIF. This creates intermediate contrainers that are destroyed during the build process as docker compose is called. Sitecore 9.2 solves this and each service may be built without needing the more complex setup.

4. Docker containers on Windows cannot mount non-empty directories due to filesystem issues. To mitigate this a volume is mounted in ***C:\Workspace*** with a watch folder that copies things to ```inetpub/wwwroot```. On the host this appears as an empty directory. Once you pull down the CM/CD repo due to the complex nature of the watch directory on first startup it is easier to publish the site to empty folder, then bring up the containers. After this is complete publishing normally works fine for non-persistent container data. This is not Sitecore but a Docker/Windows filesystem driver overlay issue.

6. Similarly, if a file is in the base image you can't see it unless you ```docker exec -it sitecore_commerce_1 ps``` depending on the container whic needs debugging. VIM installed on all the servers and logs get exported. This typically is only a problem when I was building the base images or debugging. Volumes cannot be attached as SMB network shares as in Linux. If you have an idea on this one I'm all for it. I tried SFTP and just have it anonymous for dev, but giving anonymous root access is where I started fighting the system. Plus you have to figure out the IP for the server which changes, etc. At this point it is moot, as the base image is stable and should not need changing but it is awkward if you're not used to publishing to an empty folder on the host not knowing that it really is a non-empty folder.

7. All the servers have watch folders (empty volume you can mount), you publish in Visual Studio normally and that process is complete. I have a "CreateDirs.ps1" script to aid this. Similarly dbs and Solr indexes detach and are are exported so that you may reset Docker back to base image and get the base dbs. This is similar how MySql docker works on Linux.

**Servers**
1. The XC-CM part is important. The terminology is simplified but a TODO is to break the build up into servers that just need config changes to those that need code changes. CM is the authoring environment and CD is the delivery server. The code base is the same but the configs are different. This is a multistage build. Technically this is a "standalone" build so I should use CM but the term "standalone" or XM. Again, this sounds excessive to explain but the terminology in the documentation refers to these interchangeably. This is one of the many things that makes Sitecore documentation confusing. There's an important distinction between CM/CD/XM (aka Standalone). If in a config you specify it as CM vs. CD then other configs, such as allowing access to the authoring environments are disabled. This will be solved by a multi-stage build process that uses the same code base but targets different configsets. Again, not a Docker problem, any build system will need to deal with this.

2. IdentityServer4 doesn't run well and is propietary. To get it working there are several things that need to be done. First is to patch the identity server, which is complete. The second is to use Traefik (reverse proxy), and assign it a DNS address. The problem lies with the spec for IdentityServer4/Docker not Sitecore. As Docker sees calls in the cluster as "localhost" loopback issues occur. See a Sitecore free explanation of this from Stackoverlfow (How can I use IdentityServer4 from inside and outside a docker machine?)[https://forums.docker.com/t/how-can-i-use-identityserver4-from-inside-and-outside-a-docker-machine/32110], since we don't have access to Sitecore CMS or their IdentityServer4 implementation this can't be fied. This is a known enhancement in the Sitecore issue list [bug list](https://github.com/Sitecore/docker-images/issues/72). I omitted the Traefik portion of the config.

3. MSSQL/Solr I'm skipping those as you know what they are, also I turned off the need for Redis.

4. Plumber a debug utility for commerce server that makes more sense if you have multiple catalogs/virtual catalogs/multiple tenants/complex order processes that you need to debug. For reasons below we won't really be needing it for our purposes.

5. Commerce Authoring (which itself hosts BizFx/Ops/Shops/Minions is where it gets confusing. They're all separate servers and IIS instances but they have the exact same files. Prior to 9.1 (possibly 9.0, this is important), they were just one big server. Due to lack of scalability in this setup, Sitecore has really, really poor performance, that is not a concern for this project but unfortunately is needed to get Sitecore working. Authoring has a completely different interface from Sitecore and is where bundles, products in multiple categories along with creating catalogs. It makes more sense if you realize the instances are horizontally scalable. An example, L-Brands used DemandWare (Salesforce) and they are a large retailer, they owned Express, Victoria Secrets, were global and would have deals and share products between catalogs. This project does not need this but still needs to complex setup to work. A lot of work is done actually making the site simple.

> History: To continue on this, because it was all considered one big site and not split up into 4 sites. Documentation will use old terminology such as "publish to Authoring" but what it might mean is publish to Minions. Even though you have to have all four separate IIS servers running, and Authoring also hosts BizFX on port 4200 to add to complexity, that actually publishes to Sitecore via a "Sitecore Service Proxy" (this is just an assembly) that translates the dotcore objects into something Sitecore aka .NET Framework 7.2 can read. This doesn't make sense as Sitecore uses .NETStandard 2 which is supposed to be an interop between dotcore and .NET Framework and it the assumption is that Sitecore didn't adhere to .NET standards at some point and probably created their own classes that overwrote .NET behavior. In anycase the only time you'd manipulate code in these environments would be to change how default cart behavior works and create a plugin. Additional note, anything in a cart or order no longer becomes a Sitecore item, meaning you need to make calls in a different way.

6. With the attached simple docker-compose.yaml you'll have a complete site setup, it'll already have a sample catalog "Habitat_Master" ```bootstrapped```. Meaning if you installed the SIF way (keep in mind docker is running a SIF installation, which is why I have to do a docker-compose during the base build, because I cannot create the images by just building a Dockerfile).

[![habitat](Habitat.png)]

On the catalog itself there's a dropdown that lets the user select what template they want to use for categories and products (we will not be using things like product variants or bundles).

## Dynamics/AX7

Sitecore delivered a series of plugins to integrate into Dynamics AX7. They included the following to install:

**Sitecore Dynamics Package**

A "package" that consisted of a series of assemblies and configs. These override the pipeline calls to D365 on the CM/CD server. The call for the developer remains the same, it just intercepts the call "add to cart" and puts it in Dynamics. This is not in Docker or in a build script, but since I needed to reference it in the web project I just created a package in a private repo, it was stupid simple to do. Azure makes doing things like that really easy.

**Dynamics Identity Server**

An IdentityServer4 that I call the D365IdentityServer, to distinguish it from the others. It resides on the D365 instance and requires what's known as a Azure Active Directory Application Registration. That's scripted and part of the build now. This is a proxy between Sitecore, the Identity Server and Dynamics. The delivered solution had hard coded values for the client ID and secret (needed by D365 and Dynamics to communicate). The instructions were to modify the ```clients.cs``` file at build time, but this is now abstracted into the config file. Again, this is being built as a separate pipeline agent, and on a commit to master will uninstall the proxy, reinstall then use the Azure agent. Just for the Azure Active Directory Application Registration I included what I scripted (D365IdentityServer)[D365IdentityServer.md]. I keep it as a README in the Identity Server, it is all automated. Changing this would only happen in a new Dynamics install or if for some reason the D365IdentityServer is upgraded/client secret changed. Manual instructions in case the automated pipelines don't work are included in the repository/

**Sitecore Reference Storefront**

This is a problem. It is out of date and 4 years old right now. This is open source, and was estimated orginally as to just work OOTB. This obviously is not the case, even so here is a [controller](https://github.com/Sitecore/Reference-Storefront/blob/master/Storefront/CF/CSF/Controllers/CatalogController.cs), it is complicated as it deals with multi-tenant sites, il18n and other things that are not necessary. Furthermore, problems it solves are actually solved in Sitecore 9.x, making it more confusing. The plan originally was to assume front-end code delivery was to spec, and that it needed to be applied to this with minor modifications given the deadline. This did not work, but estimates coming from a previous architect relied on this. Compare this project to their [modern Habitat Commerce Example](https://github.com/Sitecore/Sitecore.HabitatHome.Commerce). The installation instructions for the deprecated Reference Project and the Dynamics depedencies were impossible to automate. It simply did not work against 9.1 and only worked on a local install where every service, including Dynamics, was set to localhost. Compare modern code I wrote to the example above:

```csharp        
public ActionResult Banner()
        {
            return View("~/Views/Shared/Banner.cshtml", _mvcContext.GetDataSourceItem<Banner>());
		}
```

There exists folder in Sitecore with banners, and on the page the Content Author may choose which of the banners they want or create a new one which binds to a POCO model (it is an interface to allow multiple inheritence):

```csharp
    public interface Banner
    {
        Image Image { get; set; }
    }
```

In the view this is rendered as:  ```@RenderImage(x => x.Image)```.

Complicating matters, Sitecore 9.x+ hosts the ability for the Content Author to extend templates. Templates are 1:1 to strongly typed models and up until 9.x only developers would be able to access them. See [this example](https://hachweb.wordpress.com/2018/07/15/sitecore-xc-9-0-2-walk-through-extending-product-definitions-with-custom-fields/) which explains conceptual problems I was having. Sitecore doesn't have very good documentation, but basically renderings are used to select items that are templated. A Content Author would usually select a banner to be put in a location then select which banner item should be displayed. The fields on the template predefined and this works. With composer it is dynamic so that a developer is not needed. The Storefront Reference project does not reflect this and increases complexity quite a bit. These are one of the featuers that work great if a poweruser is using Sitecore but giving the Content Author the ability to manipulate templates will cause a lot of false bugs.

Since data is being imported from Dynamics and the Commerce Server is simply a cache of the data, this feature will probably not work and is not a requirement as of now. Keep in mind there's an "Update Templates" in the UI that will need to be removed so that a Content Author does not try to regenerate templates based on strongly typed models.

**Sitecore Sync**

Sitecore Sync is a console app provided by Siteore that imports data in a JSON format from D365. This includes assets such as images. This works and has a dedicated build process. For multiple reasons this sync will not return the same results, integration testing is not possible. If an item is "synced" in D365 it is marked as published in D365, running the same call will not return a product set. Running jobs in Dynamics will "unpublish" this but Dynanmics does not have CLI support and is out of the scope. Keep in mind this is different then Sitecore publishing, this is not clear and not documented.

**Sitecore Import**

Once the "Sync" happens it creates JSON files and an image folders. What is known is as the import process runs as in the Authoring environment but technically is a Minion, which is a misnomer. As explained above the documentation was written by non-native English speakers and Sitecore marketing language changes. This is currently not working as there are clear differences in the plugin and what a Sitecore 9.1 environment contains. Sitecore was made aware of this issue and does not support it as it works on their local machine. The fix should be simple, but has not been implemented yet.

**HTML**

HTML is all static, no gulp or build process or framework. It will likely need to be rewritten to an extent as front end development was not in scope. The Reference Project and the new Habitat project rely heavily on close FED and BED interaction which is not currently possibly.

## RANT

Anyway If I ditch the Reference project and its complexity and do it from scratch we'd have to do what I'm doing now which is copy code selectively. 

- We'd also have to rewrite the URL resolution. Products exist outside of the Sitecore content tree, Sitecore also changes names from like " " to "-" and a bunch of other little things. The reference project is actually broken.
- Registration and any transactional elements custom
- Figure out how it converts guest users to non-guest users and cart still is there.
- Do wish lists
- All other things that come with commerce
- This isn't actually too bad because Wish Lists is OOTB with Sitecore so we'd just be like Commerce.Connnect.AddWishList(whateverusersubmitted).
- Same with add to cart but again we're just making calls to pass data around

We're not making a custom CMS or e-commerce it'll just be hooking up data to HTML which will need to be rewritten. The reference project is a mess and I hate to rewrite but I think we'll be running up against something not made for the version we're using. I could go into more detail but we've both done e-commerce projects and those were completely custom. We have a build system in place, integration is in place. I want to move to Sitecore 9.2 (I already have that building and since it is from Sitecore it has a lot of things mine doesn't since I didn't have tie like production and development stages where you can easily hook up VS debugger, mine is just development).

I can make a more coherent list, they won't like changing to 9.2, nor will they like ditching reference project but I don't think they knew the composer thing was now an option.

Besides the identity server, which I have a work around for, docker can't be the issue as I imported the catalog using docker. I'm just coming up against a lot of "should've used that"