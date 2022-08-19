When using Azure DevOps there are situations where you need to use Personal Access Tokens (PAT). For example when interacting with the azure devops REST api to for example add comments to a work items from a schedules job on a VM.

Often you see PAT tokens being used in a azure devops pipeline to call the REST api too. When in a pipeline often you can also chose to use the alternative of the system.accesstoken predefined variable in the pipeline. Access tokens would mostly be used on process that run in other situations. For example in DSC of Azure Policy scripts, or runbooks from an Azure Automation Account. In this blog we will show how to use it from a Azure DevOps pipeline to keep it simple but this technique can be used in any situation.

A PAT is an easy way to authorize and authenticate. All information needed to authenticate is embedded in the token, also your permissions are embedded in the token. So when using the token you only need to pass on the token and you will get the permissions assigned to you. A disadvantage of them is that they can (at the moment) only be assigned to actual users. And due to how they work they are very vulnerable for leaking so the best practice is to only make PAT’s with a short lifetime.

Because we want PAT’s with a short lifetime incase they get leaked we need a system to renew them periodically. Since last year this can be done with a newly introduced API. In this post I will explain how to use this API. All files used will also be available on my github page.

Creating and Managing PAT’s
PAT’s are made by specific users. When using PAT’s in a automation workflow it’s advised to create these PAT tokens under a special service account. All tokens needed in the workflows can be created in this account and then used. To create a token login as this sevice account into Azure DevOps and in your settings menu select “Personal access tokens”.

View fullsize
Azure DevOps create PAT
In the new window which opens select the “New Token” option. Now you can create the token. A token has specific permissions and a lifetime. You can also specify if it’s only able to access a specific organization or if it can access all organizations it has access to. We create a token for all organizations

View fullsize
Azure DevOps create PAT
The PAT is now created and you get a new window showing the value. This value you need to store somewhere because you can only see this once. This value can be used in your automation to authenticate.

Renewing the PAT
There are two ways of renewing the PAT. In this blog I will look at both ways. The difference between the methods is that in the simple method it will only renew the PAT but not change the value of the token itself. This is useful if you have applications/scripts where the token is hardcoded and you can’t have it retrieve the token from a location like an Azure Key Vault. It is not advised to work with PAT’s this way.
The second advanced way will create a new PAT and so the token will also change. If you are using an Azure Key Vault you can then write the value created by it to the azure key vault to have your scripts/applications use this one afterwards.

To make sure the pipelines work for both you need to set up three variables in the pipeline. When editing the pipeline click variables and enter the variables.

View fullsize
Azure DevOps create variable
The three variables that need to be created are:
sa_password - this is the password of your serviceaccount
sa_username - this is the username of your serviceaccount
tenantid - this is tenantid of your azure ad directory

Simple method
The simple method uses a yaml pipeline to run a powershell script. I will explain what is happening in the different parts of the script.

      #Install the required modules
      Install-Module Az.Accounts -Force

      #Get the Azure Ad AccessToken
      $username = $($env:USER_NAME)
      $userpassword = ConvertTo-SecureString -String $($env:USER_PASSWORD) -AsPlainText -Force
      $Credential = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $username, $userpassword
      $null = Connect-AzAccount -Credential $Credential  -TenantId  $($env:TENANT_ID)
      $adtoken = Get-AzAccessToken

      #Create the authentication header for the DevOps API
      $Headers = @{
          "Content-Type" = "application/json"
          Authorization = "Bearer $($adtoken.Token)"
      }
      
      #Retrieve all tokens
      $Url = "https://vssps.dev.azure.com/<YOUR ORGANIZATION>/_apis/tokens/pats?api-version=7.1-preview.1"
      $tokens = Invoke-RestMethod -Uri $Url -Headers $Headers -Method GET
First the Az.Accounts module is installed, this is needed to connect to azure ad. After that a credential object is created with the username and password and then a connection to Azure is made.

After the connection is made the accesstoken is retrieved. This token is used to connect to the Azure DevOps API. Most API calls you can do with a PAT’s, but the lifecycle API is restricted to oauth authentication only.

After the accesstoken is retrieved a header is created where this token is entered as a Bearer token. And the api is called to retrieve all PAT’s for the service account. Note: to replace the <YOUR ORGANIZATION> with the name of your organization when you use it.

#Loop through all tokens
      foreach($token in $tokens.patTokens){
        #Create the new Token
        $body = @{
          AllOrgs = $true
          authorizationId = $token.authorizationId
          displayName = $token.displayName
          scope = $pat.scope
          validTo = (Get-Date ((Get-Date).AddDays(60)) -Format "MM/dd/yyyy HH:mm:ss")
        }

        $pat = Invoke-RestMethod -Uri $Url -Headers $Headers -Method PUT -Body (ConvertTo-Json $body)

        Write-Output "Renewed Pat $($pat.patToken.displayName)"
      }
In the second part a loop is created to loop through all the retrieved tokens. A body object is created which contains most of the values of the token and the new date. This date is now set to 60 days in the future (you can change this 60 to whatever you want). The AllOrgs attribute is set to true here so the token will work for all organizations. This can be changed to false if you want it to only work for your organization.

After that the REST api is called to submit the body. The token is now renewed.

Advanced method
The advanced method starts of the same but in the loop something different happens.

#Loop through all tokens
      foreach($token in $tokens.patTokens){
        #Delete the old Token
        $Url = "https://vssps.dev.azure.com/<YOUR ORGANIZATION>/_apis/tokens/pats?authorizationId=$($token.authorizationId)&api-version=7.1-preview.1"
        $pat = Invoke-RestMethod -Uri $Url -Headers $Headers -Method DELETE

        #Create the new Token
        $Url = "https://vssps.dev.azure.com/<YOUR ORGANIZATION>/_apis/tokens/pats?api-version=7.1-preview.1"
        $body = @{
          AllOrgs = $true
          displayName = $token.displayName
          scope = $pat.scope
          validTo = (Get-Date ((Get-Date).AddDays(60)) -Format "MM/dd/yyyy HH:mm:ss")
        }

        $pat = Invoke-RestMethod -Uri $Url -Headers $Headers -Method POST -Body (ConvertTo-Json $body)

        Write-Output "New Pat for $($pat.patToken.displayName) is $($pat.patToken.token)"
      }
Now the API is first called to delete/revoke the PAT. After that a new body is created, note that this time the authorizationid value is not present, this is because a new token will be created so a new id will be created.

After the body is created the API is called to create the token. In the output it will give the new value of the token. Normally you would store this value then into something like an azure key vault. For the purpose of simplicity it’s just written to the output here.

note: dont forget to replace the <YOUR ORGANIZATION> fields with the name of your organization.

Conclusion
Renewing the PAT’s is very straightforward and can be done easily. When you are forced to used PAT’s in your automation workflows I advice to create a pipeline like the advanced version shown in this blog which is schedules using a cron trigger to schedule the pipeline. Have it run every week and update the lifetime of your tokens again. If you are afraid for timeouts due to caching it’s possible to split the pipeline up in two parts where you first run the new one which creates a new token and stores this in your keyvault and have a pipeline which runs a day later to remove/revoke the old token.