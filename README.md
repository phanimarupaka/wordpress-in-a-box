Composite Package
==================================================

# NAME

  wordpress-in-a-box

# SYNOPSIS

Charlie is a platform admin in my-org and he wants to create an out-of-the-box
experience to build an application wordpress-in-a-box for teams in my-org.
He would need 2 components, mysql backend and wordpress for that. Charlie pulls
mysql and wordpress manifests from the official websites which may or may not be
kpt packages(let’s assume not). Mysql from the website uses storage and query-engine
as micro services. So Charlie felt it would be better to apply them as part of
separate resource-groups/inventory objects. Auth-module is specific to my-org and
is maintained by a different team and is a kpt package. Charlie wants to integrate
it with wordpress but also give a choice to the teams to upgrade it independently.
Since auth-module is not a separate service, he felt it’s better to maintain it in
the same resource group of wordpress.

Note: This is futuristic user experience proposal, few commands might not work
as mentioned in the steps below.

#### Steps to produce this package

To produce the package:

    $ mkdir wordpress-in-a-box; cd wordpress-in-a-box;

    # assume this is from official mysql website
    $ kpt pkg get git@github.com:phanimarupaka/mysql.git@v0.1.0 ./
    
    fetching package / from git@github.com:phanimarupaka/mysql to mysql
    sucessfully fetched the package
    
    # assume this is from official wordpress website
    $ kpt pkg get git@github.com:phanimarupaka/wordpress.git@v0.1.0 ./
    
    fetching package / from git@github.com:phanimarupaka/wordpress to wordpress
    sucessfully fetched the package

To customize the package:
    
    # initialize wordpress-in-a-box, storage and query-engine packages and their resource groups
    $ kpt pkg init .
    $ kpt pkg init mysql/storage --include-live --namespace=yourspace
    $ kpt pkg init mysql/query-engine --include-live --namespace=yourspace
    
    # set 2 packages as dependencies to wordpress-in-a-box package
    # this is the order in which the packages will be applied
    $ kpt pkg dependency set . ./mysql --strategy=resource-merge
    $ kpt pkg dependency set . ./wordpress --strategy=resource-merge
    
    # set 2 packages as dependencies to mysql package
    # this is the order in which the packages will be applied
    $ kpt pkg dependency set mysql ./storage --strategy=resource-merge
    $ kpt pkg dependency set mysql ./query-engine --strategy=resource-merge
    
    # auth-module is maintained by kpt, just add it as remote dependency to wordpress
    # it need not live in the wordpress-in-a-box package
    $ kpt pkg dependency set wordpress git@github.com:phanimarupaka/auth-module.git@v0.1.0 --strategy=resource-merge
    
    # create namespace-setter for entire package
    $ kpt cfg create-setter . namespace-setter yourspace
    
    # create replicas setters for each sub package
    $ kpt cfg create-setter mysql mysql-replicas 3
    $ kpt cfg create-setter mysql/storage storage-replicas 2
    $ kpt cfg create-setter mysql/query-engine query-replicas 1
    $ kpt cfg create-setter wordpress wordpress-replicas 5
    
    # auth-module has a different setter name for namespace: auth-namespace
    # you must group it with namespace-setter so that users can set it to
    # single value, if you wish entire application should be in same namespace
    $ kpt cfg group-setters wiab-namespace namespace-setter,auth-namespace
    
To publish the package:   
    
    $ git remote add origin git@github.com:<username>/wordpress-in-a-box.git
    $ git push origin master
    $ git tag v0.1.0
    $ git push origin v0.1.0

#### Steps to consume this package

To fetch the package:

    $ kpt pkg get git@github.com:phanimarupaka/wordpress-in-a-box.git@v0.1.0 ./
    
    fetching package / from git@github.com:phanimarupaka/wordpress-in-a-box to wordpress-in-a-box
    sucessfully fetched the package
    
    detected remote dependency auth-module in package wordpress
    fetching package / from git@github.com:phanimarupaka/authmodule to wordpress/auth-module

To edit the package:

    $ kpt cfg list-setters wordpress-in-a-box/
    
    # namespace-setter and auth-namespace are not listed as they are grouped 
    # and abstracted by wiab-namespace which makes namespace consistent
    listing setters in wordpress-in-a-box package
         NAME            VALUE     SET BY   DESCRIPTION   COUNT   REQUIRED      
    wiab-namespace     yourspace                            5       Yes        
    
    listing setters in wordpress package
         NAME            VALUE     SET BY   DESCRIPTION   COUNT   REQUIRED  
    wordpress-replicas   5                                  1       No        
    
    listing setters in auth-module package
         NAME            VALUE     SET BY   DESCRIPTION   COUNT   REQUIRED    
    auth-replicas        1                                  1       Yes        

    listing setters in mysql package
         NAME            VALUE     SET BY   DESCRIPTION   COUNT   REQUIRED      
    mysql-replicas       3                                  1       Yes        

    listing setters in storage package
         NAME            VALUE     SET BY   DESCRIPTION   COUNT   REQUIRED        
    storage-replicas     2                                  1       No          
    
    listing setters in query-engine package
         NAME            VALUE     SET BY   DESCRIPTION   COUNT   REQUIRED      
    query-replicas       1                                  1       No        
    
    
    $ kpt cfg set wordpress-in-a-box/ wiab-namespace myspace
    set 5 values in all the 5 packages combined
    
    # optional
    $ kpt cfg set wordpress-in-a-box storage-replicas myspace
    set 1 value in package mysql/storage
    
To apply the package:   
    
    $ kpt live apply wordpress-in-a-box/ --reconcile-timeout=10m
    
    packages will be applied in the following order 
        storage
        query-engine
        mysql
        wordpress (auth-module will be applied with this package as a single group)
        
    shall we proceed (y/n)? y
    
    successfully applied 4 packages