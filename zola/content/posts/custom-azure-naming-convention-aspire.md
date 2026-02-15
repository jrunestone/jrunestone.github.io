+++
title = "How to configure a custom Azure naming convention for your Aspire resources"
template = "post.html"
date = 2026-02-15
path = "/custom-azure-naming-convention-aspire"
[taxonomies]
tags = ["Aspire", "Azure"]
+++

Give your resources predictable and standardized names based on the resource type, environment and region when deploying them to Azure instead of the default random names.

<!-- toc -->

## Link to the code
I'm only providing a quick overview of the system in this post. To see the full code and a sample app check out [this repo on GitHub](https://github.com/toxicinteractive/aspire-tools/tree/main/src/AspireTools/NamingConventions). It's part of a general nuget with some other utilities but the naming convention system is completely standalone and can be extracted from there.

## Results
The following code will produce a resource name like `ca-projectname-webapp-prod-swc` when publishing to Azure with environment name `Production`:

```csharp
builder
    .WithAzureNamingConvention("projectname");

builder
    .WithProject<MyProject>("aspire-name")
    .WithAzureWorkloadName("webapp")
    .PublishAsAzureContainerApp((app, infra) => 
    {
        var ca = app.GetProvisionableResources().OfType<ContainerApp>().Single();
        ca.Location = AzureLocation.SwedenCentral;
    });

// alternative way:
builder
    .WithProject<MyProject>("aspire-name")
    
    // short-circuit the naming convention system and hard-code the Azure resource name directly
    .WithAzureWorkloadName("my-completely-custom-azure-name", overrideResourceName: true)
    
    // use the default region with Azure:Location in appsettings
    .PublishAsAzureContainerApp((app, infra) => {});
```

If the `WithAzureWorkloadName` is omitted, the resource name would be `ca-projectname-prod-swc` (without a specific workload name, useful for shared resources that only has 1 instance such as a key vault).

## Overview
The key is the `InfrastructureResolver` class and how it's invoked for each resource being provisioned. By registering our own implementation we can set the name of the resource.

1. Register a custom `InfrastructureResolver`.
2. The `ResolveProperties` method is called for each resource being provisioned.
3. Construct a context with information about the resource region, workload name, environment name and naming separator etc.
3. Try and find a registered "name resolver" for this particular resource type. This is a custom class.
4. Invoke the name resolver for the resource with the context, letting it join the segments accordingly with possible resource-specific overrides.

## The InfrastructureResolver
Here's the _gist_ of the custom infrastructure resolver:

```csharp
public override void ResolveProperties(ProvisionableConstruct construct, ProvisioningBuildOptions options)
{
    // ...
    var nameResolver = GetResourceNameResolver(resource);

    if (nameResolver != null)
    {
        // set resource name with the resolver we found for this resource type
        SetResourceName(resource, nameResolver.ResolveName(resource, new NameResolutionContext
        {
            ProjectName = _projectName,
            AzureWorkloadName = workloadNameAssociation?.AzureWorkloadName,
            EnvironmentName = _environmentName,
            DefaultRegion = _defaultRegion,
            ResourceRegion = GetResourceLocation(resource),
            SupportsRegion = GetResourceLocationProperty(resource) != null,
            Separator = GetValidNameSeparator(nameRequirements),
            NameRequirements = nameRequirements
        }));

        return;
    }

    Console.WriteLine($"No name resolver found for {resource.GetType().Name}");
}
```

## The default name resolver
A name resolver is a generic implementation, the type argument being the type of a specific provisionable Azure resource.
The default resolver looks like this:

```csharp
public virtual string ResolveName(T resource, NameResolutionContext context)
{
    var parts = new List<string?> {
        ResourcePrefixes.GetResourcePrefix(resource),
        context.ProjectName,
        context.AzureWorkloadName,
        _environmentNameResolver.ResolveEnvironmentName(context.EnvironmentName),
        context.SupportsRegion ?
            RegionNames.GetRegionName(context.ResourceRegion ?? context.DefaultRegion) :
            null
    };

    return string.Join(context.Separator, parts.Where(x => !string.IsNullOrWhiteSpace(x)));
}
```

## Resource-specific name resolvers (overrides)
Certain resources have certain requirements. Here's the name resolver of an Azure SQL database resource that isn't bound to a region and therefore shouldn't include the region name in the resource name:

```csharp
public class SqlDatabaseNameResolver : DefaultResourceNameResolver<SqlDatabase>
{
    public SqlDatabaseNameResolver(IEnvironmentNameResolver environmentNameResolver)
        : base(environmentNameResolver)
    {

    }

    public override string ResolveName(SqlDatabase resource, NameResolutionContext context)
    {
        // don't include region name in database name
        context.SupportsRegion = false;

        return base.ResolveName(resource, context);
    }
}
```

To include (or override) a custom resource name resolver simply register it with the service provider after calling `WithAzureNamingConvention`.

## Putting it all together
Here's the code for the `WithAzureNamingConvention` extension that wires everything up:

```csharp
public IDistributedApplicationBuilder WithAzureNamingConvention(string projectName)
{
    // environment name resolver
    builder.Services.AddSingleton<IEnvironmentNameResolver, DefaultEnvironmentNameResolver>();

    // register our default resource name resolvers for some resources
    builder.Services.AddSingleton<IResourceNameResolver<ContainerApp>, DefaultResourceNameResolver<ContainerApp>>();
    builder.Services.AddSingleton<IResourceNameResolver<KeyVaultService>, DefaultResourceNameResolver<KeyVaultService>>();
    builder.Services.AddSingleton<IResourceNameResolver<ContainerRegistryService>, DefaultResourceNameResolver<ContainerRegistryService>>();
    builder.Services.AddSingleton<IResourceNameResolver<OperationalInsightsWorkspace>, DefaultResourceNameResolver<OperationalInsightsWorkspace>>();
    builder.Services.AddSingleton<IResourceNameResolver<SqlServer>, DefaultResourceNameResolver<SqlServer>>();
    builder.Services.AddSingleton<IResourceNameResolver<StorageAccount>, DefaultResourceNameResolver<StorageAccount>>();

    // specific resource name resolver overrides
    builder.Services.AddSingleton<IResourceNameResolver<ContainerAppManagedEnvironment>, ContainerAppManagedEnvironmentNameResolver>();
    builder.Services.AddSingleton<IResourceNameResolver<SqlDatabase>, SqlDatabaseNameResolver>();

    // register our custom InfrastructureResolver
    builder.Services.Configure<AzureProvisioningOptions>(options =>
    {
        options
            .ProvisioningBuildOptions
            .InfrastructureResolvers
            .Insert(0, new NamingInfrastructureResolver(
                projectName,
                builder.Environment.EnvironmentName,
                GetDefaultLocationFromConfig(builder),
                builder.Services));
    });

    return builder;
}
```

