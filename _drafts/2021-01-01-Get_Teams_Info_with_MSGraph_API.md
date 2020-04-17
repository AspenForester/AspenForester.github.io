---
layout: post
title: Get Teams Infor with MS Graph API
comments: true
---

I was asked to deliver a daily extract of information about our organization’s Microsoft Teams adoption.  It was to include the Team's settings, its owners, its members (separated by internal members and guests), and when it was to expire. Ideally, I'd be able to schedule this to run each morning and drop the resulting file in a Team's Channel document library.

### The first approach  
My first approach to the problem was to just see what I could do without thinking too hard.  So, with the `MicrosoftTeams` module, I knew I could retrieve info about all the teams.  The downside is that I would have to be elevated to something like "Team Services Administrator" in Azure AD.  
`$Teams = Get-Team`  
Getting the team members would be a little more difficult because the commands that get that info are in the Exchange Remote Admin module.  
```PowerShell
$GroupObject = Get-UnifiedGroup -Identity $team.GroupId 
$Members = $GroupObject | Get-UnifiedGroupLinks -LinkType Member -resultsize unlimited
```
The last requirement was to provide a list of the Teams currently in the recycle bin along with when they'd been deleted.  That took the AzureADPreview module.
```PowerShell
$DeletedTeams = Get-AzureADMSDeletedGroup
```
There are a few problems with these parts to a solution:
1. None of those modules are natively Core compatible.  
2. The "Classic" Exchange Module requires the use of basic authentication.  
3. Running as myself, I would have to elevate to something like "Team Services Administrator" in Azure AD to use the Teams module.  
4. The report, capturing info for about 2300 Teams, takes around 45 minutes to run.

### Solution:  
Could I use an App Registration in Azure AD to connect to Teams and AzureAD?  Yes! But... Neither the classic Exchange Admin module nor the new ExchangeOnline module support App Registration authentication.  What do all these modules really have in common?  The Microsoft Graph API.  
So, I started with the great guest post by Alex Asplund on the Adam The Automator blog.  His post walks through creating the App Registration.  I hadn't created an App Registration before, but I had the rights to elevate to Global Admin to create one and assign the right application permissions for Graph API:
+ Directory.Read.All
+ Group.Read.All
+ GroupMember.Read.All
+ Team.ReadBasic.All
+ TeamMember.Read.All
+ User.Read.All  

I chose to use the certificate method of authenticating for the OAuth token, and thus created a self-signed Certificate, then exported and uploaded it to the App Registration.
Again, I used the code snippets in the blog to create an OAuth token to use with Graph API requests.  
One of the requirements was to provide the expiration date of each group.  Researching the outputs of various teams- and groups-related graph, the creation date and the last-renewed date are returned.  We need to be able to calculate the expiration date, and for that we need to know what the lifecycle policy is for the groups.
```PowerShell
 $GLCPolicyURI = "https://graph.microsoft.com/v1.0/groupLifecyclePolicies"
$GLCPolicyRequest = Invoke-RestMethod -uri $GLCPolicyURI -Headers $header -Method Get 
# $GLCPolicyRequest.value[0].groupLifetimeInDays 
```
The request returns you a collection (of one, in my case) Group Lifecycle Policies with all of the policy's properties including the `groupLifetimeInDays` value that we're after.  
Now we want to get the groups.  We want to end up with the Teams, and Teams are "Unified" groups in Azure AD.  The following will get the Unified groups in your organization  
```PowerShell
$UnifiedGroupsURI = "https://graph.microsoft.com/v1.0/groups?`$filter=groupTypes/any(c:c+eq+'Unified')"
$AllUnifiedgroupsRequest = Invoke-RestMethod -uri $UnifiedGroupsURI -Headers $header -Method Get
```
>**Note:** When using the OData queries via PowerShell, you need to be careful to include the backtick that escapes the dollar sign, otherwise PowerShell is going to try and evaluate `$filter`. 

Another thing to be aware of is the Graph endpoints generally return just 100 items per page, but that response includes a link to get the **next** page.  So, to deal with that and get all of the groups, use a `while` loop
```PowerShell
while ($null -ne $AllUnifiedGroupsRequest.'@odata.nextLink')
    {
        $AllUnifiedGroupsRequest = Invoke-RestMethod -uri $AllUnifiedGroupsRequest.'@odata.nextLink' -Method Get -Headers $header 
        $AllGroups += $AllUnifiedGroupsRequest.value
    }
```

That gives us all of the Unified groups, but not all Unified groups are Teams!
```PowerShell
$Teams = $AllGroups | where-object resourceProvisioningOptions -eq "Team"
```
We know that going and getting the Team object, the members, and the owners is going to involve a lot of waiting for `Inovke-RestMethod` responses, let's start by leveraging some parallelism with:
```PowerShell
$TeamsReport = $teams | foreach-object -parallel {...}
```
Another point where we can achieve some efficiency is with making batch requests to the Graph API using the `'https://graph.microsoft.com/v1.0/$batch'` endpoint.  Again, be mindful of your single vs. double quotes here.  By using single quotes in this case, we avoid the dollar sign issue.  Using the batch endpoint is well documented at https://docs.microsoft.com/en-us/graph/json-batching?view=graph-rest-1.0 .  Create a hashtable key-value pair that is a collection of hashtables, one for each of the requests you want to submit in the batch.  Convert that to JSON, and submit it as the body of the request.
```PowerShell
$batch = @{requests = @(
                @{
                    id     = 1
                    method = "GET"
                    url    = "/teams/$($team.id)"
                },
                @{
                    id     = 2
                    method = "GET"
                    url    = "/groups/$($team.id)/members"
                },
                @{
                    id     = 3
                    method = "GET"
                    url    = "/groups/$($team.id)/owners"
                }
            )
        }
$batchjson = $batch | ConvertTo-Json 
```
You can see above that we're getting the Team, the members, and the owners, all keyed by the `ID` property.  Submit that block of JSON to the `$batch` endpoint and parse the results: 
```PowerShell
$BatchSplat = @{
            Method = "POST"
            URI = 'https://graph.microsoft.com/v1.0/$batch'
            ContentType = "application/json"
            Body = $batchjson
            Headers = $Using:Header
            ErrorAction = "Stop"
        } 
$batchRequest = Invoke-RestMethod @BatchSplat

$TeamRequest = ($batchRequest.responses | where-object id -eq 1).body
$memberRequest = ($batchRequest.responses | where-object id -eq 2).body
$OwnerRequest = ($batchRequest.responses | Where-Object id -eq 3).body.value # We'll assume for now that there are fewer than 100 owners
```
The advantage here is that we only make one request instead of three transactions.  For the members, we'll deal with pagination in the same way as with the Teams themselves.
```PowerShell
$allmembers = $memberRequest.value 
while ($null -ne $memberRequest.'@odata.nextlink')
{
    $memberRequest = Invoke-RestMethod -Uri $memberRequest.'@odata.nextlink' -Method Get -Headers $Using:Header
    $allmembers += $memberRequest.value
}
```
Distinguishing the "guests" from the internal "members" is just a matter of examining the User Principal Name (UPN), AzureAD guests will have "EXT" in middle of the UPN.
```PowerShell
$Members = $allmembers | Where-Object userPrincipalName -notlike "*EXT*"
$Guests = $allmembers | Where-Object userPrincipalName -like "*EXT*"
```
We've collected all the information we need for the team, we only need to calculate that expiration date before we generate some output:
```PowerShell
$ExpirationDate = $Team.renewedDateTime.addDays($($Using:GLCPolicyRequest).value[0].groupLifetimeInDays)
```
It's a matter of outputting a `[pscustomobject]` with all of the team properties.
```PowerShell
[pscustomobject]@{
    GroupID                           = $teamrequest.id
    DisplayName                       = $TeamRequest.displayName
    Description                       = $TeamRequest.description
    Visibility                        = $Team.visibility
    MailNickName                      = $Team.MailNickName
    Classification                    = $TeamRequest.classification
    Archived                          = $TeamRequest.isArchived
    AllowGiphy                        = $TeamRequest.funSettings.allowGiphy
    GiphyContentRating                = $TeamRequest.funSettings.giphyContentRating
    AllowStickersAndMemes             = $TeamRequest.funSettings.allowStickersAndMemes
    AllowCustomMemes                  = $TeamRequest.funSettings.allowCustomMemes
    AllowGuestCreateUpdateChannels    = $TeamRequest.guestSettings.allowCreateUpdateChannels
    AllowGuestDeleteChannels          = $TeamRequest.guestSettings.allowDeleteChannels
    AllowCreateUpdateChannels         = $TeamRequest.memberSettings.allowCreateUpdateChannels
    AllowDeleteChannels               = $TeamRequest.memberSettings.allowDeleteChannels
    AllowAddRemoveApps                = $TeamRequest.memberSettings.allowAddRemoveApps
    AllowCreateUpdateRemoveTabs       = $TeamRequest.memberSettings.allowCreateUpdateRemoveTabs
    AllowCreateUpdateRemoveConnectors = $TeamRequest.memberSettings.allowCreateUpdateRemoveConnectors
    AllowUserEditMessages             = $TeamRequest.messagingSettings.allowUserEditMessages
    AllowUserDeleteMessages           = $TeamRequest.messagingSettings.allowUserDeleteMessages
    AllowOwnerDeleteMessages          = $TeamRequest.messagingSettings.allowOwnerDeleteMessages
    AllowTeamMentions                 = $TeamRequest.messagingSettings.allowTeamMentions
    AllowChannelMentions              = $TeamRequest.messagingSettings.allowChannelMentions
    ShowInTeamsSearchAndSuggestions   = $TeamRequest.discoverySettings.showInTeamsSearchAndSuggestions
    Owner                             = $OwnerRequest.displayname -join "; "
    OwnerCount                        = $OwnerRequest.Count
    Member                            = $Members.displayName -join "; "
    Guests                            = $Guests.displayName -join "; "
    MemberCount                       = $Allmembers.Count
    WhenCreated                       = $Team.createdDateTime
    RenewedDateTime                   = $Team.renewedDateTime
    ExpirationDate                    = $ExpirationDate
}
```
The last requirement I was presented with was to provide a listing of all of the Groups that were in the recycle bin.  That's found the Graph endpoint `$DeletedGroupsURI = "https://graph.microsoft.com/v1.0/directory/deletedItems/microsoft.graph.group"` As with the Groups themselves, and the members, the results are paginated, and are handled the same ways as before.  My organization wanted an Excel spreadsheet so I used [Doug Finke's ImportExcel](https://github.com/dfinke/ImportExcel) module to export the collections.
```PowerShell
$TeamsReport | Export-Excel -Path $TodaysReport -WorksheetName 'Teams' -ClearSheet
$DeletedTeams | Export-Excel -Path $TodaysReport -WorksheetName 'RecycleBin' -ClearSheet 
```
### Conclusion
Working with the Graph API can allow you to customize your approach to AzureAD and Teams management.  Using the modern App Registration authentication acts not unlike JEA in that an unprivileged account can be enabled to perform specific limited tasks.  In this case we were able to efficiently gather all of the information we needed about the state of our Teams environment, while managing and maintaining control over access.  With no elevated access required, "who" the script runs as becomes less important, as long as that entity has access to the file location

The full code for this solution will be available at 