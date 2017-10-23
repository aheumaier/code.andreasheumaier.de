---
layout: post
title: "Azure Resource Manager Template Best Practices"
date: 2017-10-22 15:36:00 +0200
comments: true
categories: 
  - Azure 
  - Templates
  - ARM
  - Cloud
---
I was tasked with helping refactoring the Azure Resource Manager deployment templates. They wanted something that had some consistency, as well as increased flexibility. The week following this, I was on-site with one of my ISV partners where we had a similar need. Both these projects helped drive my understanding and skill with ARM templates to an entirely new level. Along the way I learned a few tips/tricks that I figured I’d pass along to you.
JSON is “object” notation<!--more-->

The first learning is to realize that an ARM template isn’t just a bunch of strings, its defining objects that represent resources you want the Azure providers to create for you. An ARM template is a JSON (javascript object notation) file consisting (for the most part) of key/value pairs, object declarations (stuff inside curly brackets) and arrays (stuff inside square brackets). Furthermore, ARM templates provide us with various functions that can be use to create, manipulate, and insert things in the template.
Now, if you look at something like a simple Windows VM’s ip configuration, we can see this.
``` json Resource ipConfigurations
"ipConfigurations": [
    {
        "name": "ipconfig1",
        "properties": {
             "privateIPAllocationMethod": "Dynamic",
             "publicIPAddress": {
                  "id": "[resourceId('Microsoft.Network/publicIPAddresses',variables('publicIPAddressName'))]"
             },
             "subnet": {
                 "id": "[variables('subnetRef')]"
             }
         }
  
```
This section is an array (square brackets) of objects (curly brackets). And this particular example is associating the VM (well, its NIC actually) with a public IP address and the subnet by setting the values for those particular properties of the IP configuration “object”. But… what about if you’re using a load balancer?
``` json Resources ipConfigurations
"ipConfigurations": [
    {
        "name": "ipconfig1",
        "properties": {
            "privateIPAllocationMethod": "Dynamic",
            "subnet": {
                 "id": "[variables('subnetRef')]"
            },
            "loadBalancerBackendAddressPools": [
                 {
                     "id": "[concat(variables('lbID'), '/backendAddressPools/BackendPool1')]"
                 }
            ],
            "loadBalancerInboundNatRules": [
                 {
                     "id": "[concat(variables('lbID'),'/inboundNatRules/RDP-VM', copyindex())]"
                 }
            ]
        }
    }
]
```
Now the same “object” has a different set of properties. Gone is the publicIPAddress setting, and added is the loadBalancerBackendAddressPools and loadBalancerInboundNatRules. Not a big deal, unless you’re trying to create a template for a VM that can be easily deployed in either configuration. But if we look at the sections I’ve selected above, we realize that our template can actually look more like this.
``` json Params ipConfigurations
"ipConfigurations": [
    {
        "name": "ipconfig1",
        "properties": "[parameters('ipConfig')]"
    }
]
```
In this example, we still have an array with one object, but rather then defining the individual properties, we’ve instead said that the properties are contained in a parameter that was passed into the template itself. A parameter that looks as follows:
``` json Discover if a number is prime
"ipConfig": {
    "value": {
        "privateIPAllocationMethod": "Dynamic",
        "privateIPAddress": "[parameters('privateIP')]",
        "subnet": {
            "id": "[parameters('subnetResourceId')]"
        },
        "publicIPAddress": {
            "id": "[resourceId('Microsoft.Network/publicIPAddresses', variables('publicIPName'))]"
        }
    }
}
```
We could also just as easily construct the object in a variable. Which can be really helpful if we have a common set of settings we want to share across multiple objects in the same template.
This realization also opens up a whole new world of possibilities as we can now pass objects as parameters into a template,
``` json Params ipConfigurations
"ipConfig": {
    "type": "object",
    "metadata": {
        "description": "The IP configuration for the VM"
    }
}
```
and receive as output from a template
``` json Output ipConfigurations
"outputs": {
    "subnetIDs" : {
        "type" : "object",
        "value": {
            "frontEnd" : "[variables('subnetFrontEndRef')]",
            "backEnd" : "[variables('subnetBackEndRef')]",
            "management" : "[variables('subnetManagementRef')]"
        }
    }
}
```
By using objects and not just simple data types (strings, integers, etc…), we make it a big easier to group values together and pass them around.
Using variables to transform

I mentioned declaring objects in the variable section for reuse. But we can also use variables to transform things. Lets say you’re creating a template for a SQL database. The database needs an edition and a requestedserviceObjectiveName (tier). You could have both values passed into template and then set the properties using those values. But perhaps you want to simplify that for the template’s end user to avoid something like a request for a “Standard” edition with a “P4” service tier. So the template declares an input parameter that looks something like the following.
``` json Discover if a number is prime
"databaseSKU": {
    "type": "string",
    "defaultValue": "Basic",
    "allowedValues": [
        "Basic",
        "Standard",
        "Standard S1",
        "Standard S2",
        "Standard S3",
        "Premium P1",
        "Premium P2",
        "Premium P4",
        "Premium P6",
        "Premium P11",
        "Premium P15"
    ],
    "metadata": {
        "description": "Specifies the database pricing/performance."
    }
}
```
The user just declares that they want a “Basic”, or “Standard S2”… and the template transforms that into the appropriate settings. In the variables section of the template, we then create a a collection of objects that we can access using the parameter value as a key. Each object in the collection sets the values that can be used to set the properties of the database.
``` json Discover if a number is prime
"databasePricingTiers" : {
    "Basic" : {
        "edition": "Basic",
        "requestedServiceObjectiveName": "Basic"
    },
    "Standard" : {
        "edition": "Standard",
        "requestedServiceObjectiveName": "S0"
    },
    …
}
```
Since each item in the collection is an object, we can even use it set an entire section of the database configuration just like we did the IP configuration earlier.  Something like..

``` json
"properties”: “[variables(‘databasePricingTiers’)[parameters(‘databaseSKU’)]]"
```
We can even take this a step further, and have more complex templates use simplified sizings such as “small”, “medium”, and “large”, which are used to control all kinds of individual settings across different resources.
Linked Templates
IMHO, there are two advantages to the techniques I just mentioned. Used properly, I feel they can make a template easier to maintain. But just as importantly, these allow for reuse. And reuse is most evident when we start talking linked templates.
A linked template is one that’s called from another template. Its accomplished by providing the URL for where the template is located. This means that the template has to be somewhere that it can be linked to. A web site, or the raw github source link works well.  But sometimes you don’t want to expose your templates publicly.
This is where the powershell script I have in my repo comes in. Among other things, it creates a storage account and uploads all the templates to it.

``` powershell Get-ChildItem
Get-ChildItem -File $scriptRoot/* -Exclude *params.json -filter deploy-*.json | Set-AzureStorageBlobContent `
    -Context $storageAccount.Context `
    -Container $containerName `
    -Force
```
This snippet has been designed to go with the naming conventions I’m using. So it will only get files that start with “deploy-“ and end in “.json”. It also ignores any files that end in “params.json”, so I can include parameter files locally for testing purposes and not have to worry about uploading them accidentally. My GitHub repo has taken this a step further and ignores any files that end in privateparams.json so I don’t accidentally check them in.
I’d like to call out the work of Stuart Leeks on this. He did the up front work as part of the Nether project I mentioned earlier. I just adapted it for my needs and added a few minor enhancements in. I’ve worked with Stuart on a few things over and years and its always been a pleasure and great learning experience. So I really appreciate what I learned from him and for him as a result of some of the work on the Nether project. Back to the task at hand.
With the files uploaded, we then have to link to them. This is why some of my templates have you pass in a templateBaseURL and templateSaaSToken.  I’ve parameterized these values allow me to construct the full URI for where the files will be located. Thus I could pass in the following for templateBaseURL if I just wanted to access them from my GitHub repository:
https://raw.githubusercontent.com/brentstineman/PersonalStuff/master/ARM%20Templates/LinkedTemplateExample/
the templateSaaSToken is there in case you want to use a shared access signature for a blob container to access the files.
Any ARM template can pass values out. But when combined with linked templates, we can now take those outputs and pass them into subsequent templates.  Something like…

``` json
"sqlServerFQDN": { "value": "[<strong>reference('SQLDatabaseTemplate').outputs.</strong>databaseServerFQDN.value]" }
```
In this case “reference” says we are referencing the runtime values of an object in the current template (in this case of a linked template). From there we want its outputs and specifically the one named databaseServerFQDN, and finally its value property. In the template that outputs these values they are declared like…
``` json Template Outputs

"outputs": {
    "databaseServerFQDN" : {
        "type" : "string",
        "value": "[reference(variables('sqlDBServerName')).fullyQualifiedDomainName]"
    },
    "databaseName" : {
        "type" : "string",
        "value": "[parameters('databaseName')]"
    }
}
```

Outputs is an object, that contains a collection of other objects. Note the property outputs.databaseServerFQDN.value. We could also get databaseServerFQDN.type if we wanted. Or access the database name properties.
What’s also important here is the reference function. You may have seen this used in other places and thought it was interchangeable with the resourceID function. But its when you work with linked templates that it really shines. The reference function is really telling the resource provider to wait until the item I’m getting a reference to has completed, then give me access to its run time properties. This means that you don’t even need a “dependson” for the other template as the resource function will wait for that template to complete already. But me, I like putting it in anyways. Just to be safe.
The other big item here is that when we call a linked template, we have to give it a name. And here’s why… Each template is essentially run independently by Azure’s resource manager. So if you have a master template, that’s using 4 linked templates and then you check the resource group’s deployment history, you’d actually see 5 deployments.

multiple deployments resulting from a single master template with multiple linked templates.
Now the reason these deployment names are important is because the resource manager will track them and won’t allow two deployments with the same name to run at the same time. This isn’t a big deal most of the time. But earlier in this post, I described creating a reusable virtual machine template. That template is used by a parent template to create a resource. And if I have 2-3 of those parents running, I need to make sure that names don’t collide.
Now the handy way to avoid this is to reference a run-time value with the ARM template… deployment(). This exposes properties about the deployment such as the name. So when calling a linked template, we can actually craft a unique name by doing something like…
``` json 
concat(deployment().name, ‘-vm’) 
```

This allows each deployment template to take the parent’s name and add its own unique suffix on. Thus (hopefully) helping avoid having to deal with non-unique nested names. If you look at the image to the right, you’ll see deployments like jumpboxTemplate and jumpboxTemplate-vm. The later deployment is a reusable template that is linked from the former. And I’m using the value of deployment to set name of the vm template deployment. The same is also present in loadbalancedvmTemplate-lbvms000 and 0001. In that case, this is two VMs being deployed using the same linked template, but in this case being done multiple times as part of a copy loop in the parent template.
Other Misc Learnings

As if all this wasn’t enough, there were a couple other tips I wanted to pass along.
When I was working with Stuart on the Nether project, we wanted a template that would add a consumer group to an existing Service Bus event hub. Unfortunately, all the Service Bus templates we could find only showed the creation of the consumer group as part of creating the event hub via an approach called nested resources. I was able to quickly figure out how to create the consumer itself, but the challenge was how to then reference it.
When working with nested resources it is important to understand the paths present in both the resource type and its name. In the case of our consumer group, we were quickly able to determine that the proper resource type would be Microsoft.EventHub/Namespaces/EventHubs/ConsumerGroups.
You might assume that now that you have the type path, you’d just specify the resource name as something like “myconsumer”. But with nested resources, its gets more complicated The above type represents 3 nested tiers. As such, the name needs to follow suit and have the same number of tiers. So I had to actually name to something more like //.
Stuart pointed me to a tip he learned on another project. Namely that these two values are combined like the teeth on a zipper to create the path to the resource:
Microsoft.EventHub/Namespaces/NamespaceName/EventHubs/HubName/ConsumerGroups/GroupName

Once I realized this, a light bulb went off. This full name actually reflects the same type of value you usually get back from a call to the resourceId function. This function accepts two parameters, a type, and a name, and essentially zips them together while also adding on the leading value based on the current resource group (subscription and the like). You can even see this full path when you look at the properties for an existing object in the Azure portal.
Now the second tip is about the provider API versions. I often asked why I put these values into a variable and what should be the right value. Well, I put them in a variable because it means there’s less I have to accidentally mess up when creating a template. It also means that if/when I want to update the version of an API I’m using, I only have to change it once.
But as for the big question about how do we know what versions of the API exist… I got that tip from Michael Collier (former Azure MVP and currently one of my colleagues) who in turn got it from another old friend, Neil MacKenzie. They pointed out that you can get these pretty easily via Powershell.
``` powershell
(Get-AzureRmResourceProvider -ProviderNamespace Microsoft.Compute).ResourceTypes | where {$_.ResourceTypeName -eq 'virtualMachines'} | select -ExpandProperty ApiVersions
```
This powershell command will spit out the available API versions for Microsoft.Compute/VirtualMachines.
New versions are shipped all the time and its great to know I can be aware of them without having wait for someone to publish a sample template with those values in them.
My last item comes from another colleague, Greg Oliver. Greg has found what when you’re working on templates, you really get slowed down waiting for each deployment to finish, then get deleted, then start the deployment over again. So he’s taken to adding an ‘index’ parameter to his templates. Then, when he runs them, he simply increments the value (index++). Then, while the new deployment is running, you can go ahead and start deleting the old one. There can be several “old” iterations in the process of deleting while you continue to work on your template. Something like this could also be used  as part of my suffix approach, but Greg has gone the extra mile to make the iteration its own parameter. Awesome time saving tip!
All in all, I think these are some great, if little known, ARM tips.

## Deployment Complete
I wish I could say that these tips and tricks will make building ARM templates easier. Unfortunately, they won’t. Building templates requires lots of hands on practice, patience, and time. But I hope the tips I’ve discussed here might help you craft templates that are easier to maintain and reuse.
To help illustrate all these tricks (and a few less impressive ones), I’ve created a series of linked templates and put them in a single folder on GitHub. These include a PowerShell script to run the deployment as well as sample parameter files. I hope to continue to tweak these as I learn more including adding into the PowerShell script some options to help prevent issues with dns name collisions.