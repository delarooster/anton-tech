---
title: 'Taming Docker with Private NuGet Feeds'
type: blog
date: 2025-03-18
prev: blog/dont-bake-app-settings.md
next: blog/container_apps-private_networking.md
---

# Breaking the Local Build Dependency: A Docker Challenge

Let's be honest - we've all been there. You clone a repo, run the Docker build command, and... nothing works. That's exactly what happened to us recently at Anton Tech when we inherited a Docker setup that assumed all files were already built locally before creating the image. Talk about frustrating.

This approach flies in the face of what makes Docker so fantastic in the first place. Aren't containers supposed to free us from the "but it works on my machine" headache? We thought so too, which is why we rolled up our sleeves and fixed it properly.

If you've ever pulled your hair out trying to get a .NET solution to restore packages from a private Azure DevOps feed inside a Docker container, grab a coffee and read on - we've got you covered.

## Setting Up Your Project (The Right Way!)

### Solution-wide NuGet.config: Your New Best Friend

First up, let's talk about that all-important `NuGet.config` file. If you're not already placing one at the root of your directory (right next to your `.sln` file), you're missing out on some serious convenience.

Here are two key things to remember (I learned these the hard way, so you don't have to):

1. Make sure the `key` for your `packageSource` matches what you've set up in Azure DevOps
2. The `value` needs the full URI path - organization name, project name, feed name - the works!

Here's what ours looks like:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
    <add key="AntonTechPackages" value="https://pkgs.dev.azure.com/AntonTech/Internal/_packaging/AntonTechPackages/nuget/v3/index.json" />
  </packageSources>
</configuration>
```

This setup lets you juggle multiple package sources without losing your mind. Plus, when a teammate asks, "Where is this package coming from again?" you'll have a clear answer. Visibility makes everyone's life easier.

### Building a Dockerfile That Actually Works

Now for the Dockerfile. I'm a big fan of breaking out arguments for readability - it might seem like extra work now, but your future self (and colleagues) will thank you when debugging.

Here's the secret sauce that makes everything work with your private feed:

```dockerfile
ARG PERSONAL_ACCESS_TOKEN
ARG PRIVATE_FEED_URL="https://pkgs.dev.azure.com/AntonTech/Internal/_packaging/AntonTechPackages/nuget/v3/index.json"
ENV NUGET_CREDENTIALPROVIDER_SESSIONTOKENCACHE_ENABLED=true
ENV VSS_NUGET_EXTERNAL_FEED_ENDPOINTS="{\"endpointCredentials\": [{\"endpoint\":\"${PRIVATE_FEED_URL}\", \"password\":\"${PERSONAL_ACCESS_TOKEN}\"}]}"
RUN apt-get update && apt-get install -y wget
RUN wget https://raw.githubusercontent.com/Microsoft/artifacts-credprovider/master/helpers/installcredprovider.sh \
    && chmod +x installcredprovider.sh \
    && ./installcredprovider.sh

RUN dotnet restore AntonTechApi.sln --configfile NuGet.config
```

Let me break down what's happening here:

* **PERSONAL_ACCESS_TOKEN**: You'll need to generate an Azure DevOps PAT. Don't worry, it's easier than it sounds - just [follow Microsoft's tutorial](https://learn.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate).
* **PRIVATE_FEED_URL**: This is the same URL from your NuGet.config - consistency is key!
* **Credential Provider**: That script we're downloading? It's Microsoft's magic wand for handling authentication with Azure DevOps.
* **The Restore Command**: Notice how simple it is? That's the beauty of having a well-configured NuGet.config.

## The Full Monty: Our Complete Dockerfile

For those who want to see how it all fits together, here's our entire Dockerfile for a .NET 8.0 project:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
ENV ASPNETCORE_URLS=http://*:8080

WORKDIR /source
COPY ["NuGet.config", "./"]
COPY ["DemoSolution.sln", "./"]

# Configure Nuget to use the Azure Artifacts credential provider
ARG PERSONAL_ACCESS_TOKEN
ARG PRIVATE_FEED_URL="https://pkgs.dev.azure.com/AntonTech/Internal/_packaging/AntonTechPackages/nuget/v3/index.json"
ENV NUGET_CREDENTIALPROVIDER_SESSIONTOKENCACHE_ENABLED=true
ENV VSS_NUGET_EXTERNAL_FEED_ENDPOINTS="{\"endpointCredentials\": [{\"endpoint\":\"${PRIVATE_FEED_URL}\", \"password\":\"${PERSONAL_ACCESS_TOKEN}\"}]}"
RUN apt-get update && apt-get install -y wget
RUN wget https://raw.githubusercontent.com/Microsoft/artifacts-credprovider/master/helpers/installcredprovider.sh \
    && chmod +x installcredprovider.sh \
    && ./installcredprovider.sh

RUN dotnet restore AntonTechApi.sln --configfile NuGet.config
COPY . .

RUN dotnet publish DemoSolution/AntonTechApi.csproj -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
COPY --from=build /app .
EXPOSE 5000
HEALTHCHECK CMD curl --fail http://localhost || exit 1

CMD ["dotnet", "AntonTechApi.dll"]
```

## Why You'll Love This Approach

After implementing this solution, our team noticed several benefits that made our lives easier:

1. **True "works everywhere" portability**: No more "but it built fine on MY machine"
2. **Better security**: We're handling authentication properly without credentials showing up in logs or history
3. **Crystal-clear transparency**: Everyone can see exactly which packages come from where
4. **Easy updates**: When URLs change (they always do), we only need to update them in one place

## Wrapping Up

Docker and private NuGet feeds don't have to be mortal enemies. With a little setup work, you can create a clean, repeatable process that works for everyone on your team. No more mysterious build failures or dependency headaches.

Have you battled with similar Docker challenges? [Send us an email](https://anton-tech.com/contact/) - I'd love to hear your war stories and solutions. We're all in this containerized world together.

---

*Need a hand implementing these approaches in your own projects? Our team at [Anton Tech](https://anton-tech.com/contact) loves tackling these kinds of challenges. Reach out, and let's chat about how we can help!*