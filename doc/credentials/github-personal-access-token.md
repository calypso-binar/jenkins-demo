# GitHub Personal Access Token

We will need a personal access token from GitHub to allow Jenkins to pull from our repositories.  

## Create personal access token

We can follow [GitHub Personal Access Token](https://github.com/cloudbees/intro-to-declarative-pipeline/blob/master/Github-Personal-Access-Token.md) to create one. We will have to copy the token somewhere since after we close the webpage we won't be able to see the token anymore.

## Add Personal Access Token to Jenkins

Now, that we have the personal access token, we will save it in Jenkins.  
Open https://localhost:8443/jenkins and log in with the admin user.  
Navigate to `Manage Jenkins` > `Manage Credentials  `
Here we will see `Credentials` and `Stores Scoped to Jenkins`. Click on Stores scoped to Jenkins, then on `Global credentials (unrestricted)`.  
On the left side of the browser click on Add Credentials.  
Set the Kind to `Username with password`.  
Set Scope to System. 
Set Username to our GitHub Username.
Set the Password to the Personal Access Token.
ID can be set to something descriptive, like github-personal-access-token  
Description can be something like "Personal Access Token from GitHub to enable Jenkins to clone Git repositories."  
Save it, and you are done.