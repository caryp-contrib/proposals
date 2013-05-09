## Extending Chef Deploy for Object Storage Services


## OVERVIEW
Some cloud production environments require the deployment of application code to be stored in a highly-available object storage service (like S3 or CloudFiles) to avoid being dependent on github.com (or some other SCM server) availability when launching virtual servers.  To do this we propose a new SCM chef provider named ```Chef::Provider::ScmObject```.  This provider will be used to download application code to the filesystem much like the ```Chef::Provider::Git``` and ```Chef::Provider::Subversion``` providers do, the only difference is that it will do so from an archive file (i.e. tar) stored in a remote object storage service, instead of a traditional SCM server.  The reason for treating these  archive files as SCM artifacts is to leverage the all the migration, callback and rollback functionality offered by the existing [Deploy](http://docs.opscode.com/resource_deploy.html) chef resource.

The following sections go into the details of changes needed to support such a provider.

## SCOPE
Although, there may still be concerns with other sorts of external dependencies -- like mirroring package repos, mirroring rubygems, and making sure there are no git'ed gems in your application [Gemfiles](http://gembundler.com/v1.3/gemfile.html), this proposal only focuses on removing the external SCM dependency for our web application code.


## IMPLEMENTATION
These archives (tarballs) contain our web application code, and so, we will want to leverage all the migration, callback and rollback functionality offered by the existing [Deploy](http://docs.opscode.com/resource_deploy.html) chef resource for flexible application deployment.

The Deploy resource delegates to the [SCM](http://docs.opscode.com/resource_scm.html) chef resource for downloading the application code, so we will implement ```Chef::Provider::ScmObject``` as an SCM provider, which will allow it to 'plug-in' to the deploy resource.

For consistency with the Git and Subversion support, we will need to add some additional object storage specific attributes to both the SCM and Deploy resources. See [ATTRIBUTES](#attributes) for details.

Once supported by the above resources, it should be easy to extend ObjectSCM deployment as an option to the community [Application cookbook](http://community.opscode.com/cookbooks/application).

For some other tangential issues see the [OTHER STUFF](#other-stuff) section at the end of this doc.


## ATTRIBUTES
The following attributes are proposed for addition to the Deploy and SCM resources, with the intention that they will be used with ScmObject providers only:

* ```object_storage_account```  :  The account name (i.e. username, ID) that is required to access files in the specified location. This input is optional and may not be required if the files are publicly accessible.

* ```object_storage_credential``` : A valid credential (i.e. password, SSH key, account secret) to access files in the specified location. This input is optional and may not be required if the files are publicly accessible.

* ```object_storage_endpoint``` : The endpoint (URL) to the object storage service.  This is used for private object storage providers, like ```Chef::Provider::ScmObjectSwift```

* ```object_storage_container``` : The container, bucket or path that contains your application archives.

* ```object_storage_filename``` : The full or partial filename that will be used to locate the application RCHIVE. See [ARCHIVE NAMING CONVENTIONS](#archive-naming-conventions) section below for more details

## ARCHIVE NAMING CONVENTIONS

The initial version of the ScmObject providers will support .tar and .tgz archive extensions.

To support deploying the latest archive file in the container, there needs to be a natural sort order in the naming convention you choose for your archive files. For example: 
  
    mycoolapp-<epoch_timestamp>.tgz
  
In this case, to deploy the latest archive just use the prefix ```mycoolapp``` as the value for the ```object_storage_filename``` attribute.

To deploy a specific archive, the value for the ```object_storage_filename``` attribute should match the entire filename.  For example:  

    mycoolapp-1368041763.tgz

To use the ```Chef::Resource::DeployRevision``` or ```Chef::Resource::DeployBranch``` providers, your naming convention needs to supply a revision in the name.  The current thinking is that this will be the last dash delimited section in your naming convention, before the extension.  For example:

     mycoolapp-<timstamp>-<SHA>.tgz
     
But this needs a little more thought, before we commit to supporting these deploy providers.


## PROVIDERS
The providers would likely be built on top of the fog gem.  We currently have support for:

    S3
    CloudFiles
    Google
    Azure Blob Storage
    Openstack Swift
    HP
    SoftLayer

So I see these being possible deliverables in the first pass.


## OTHER STUFF

### Delivery of initial support
The beta version of this feature will be delivered in cookbook form.  To do this we will be monkey-patching both the ```Chef::Resource::Deploy``` and ```Chef::Resource::Scm``` resource classes.  Not a great solution, but will allow for easy test-driving of the feature and quick bugfix iterations.  Ideally, this code will eventually be folded into the Chef gem itself.

### SCM actions don't map to an object storage service
Since an object store is not a true SCM, the ```:sync``` and ```:checkout``` actions for this resource will simply be aliased in the provider code to execute the ```:export``` action.

### Stand-alone object_storage resources
Since downloading from object storage services has value by itself, we might also choose to add similar support to ```remote_file``` and ```remote_directory``` resources -- or, more likely, a new ```object_storage``` resource altogether.  (input welcome!)

The SCM providers should be able to inherit and build on ```object_storage``` providers for the ```:export``` and ```:force_export``` actions.

This will be a separate proposal -- even though it might be worked in conjunction with this feature.
