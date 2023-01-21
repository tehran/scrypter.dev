---
layout: single
title: PowerShell - Retrieve a local group membership
excerpt: 
permalink: /2013/01/get-localgroupallmembers.html
tags: 
- active directory
- audit
- group membership
- local administrators
- nested groups
- powershell
published: true
comments: true
---
In my <a href="{{ site.url }}/2012/12/get-localgroupmembership-using-adsiwinnt.html" target="_blank">previous post</a>, I explain my PowerShell function to retrieve the local administrators group membership.
Today I will go a bit further and find the nested members from the Active Directory.

## Get-LocalGroupAllMembers

The following scripts query a localgroup on the localhost or a remote computer, and gather all the local and domain members (direct and nested).

By Default, if you do not specify any parameter, the function will query the Localhost and the Localgroup "Administrators".

## Dot Sourcing

Once your Powershell is launched you can load the function using the Dot Sourcing method:

`. ./Get-LocalGroupAllMembers.ps1`

## Usage

`Get-LocalGroupAllMembers -ComputerName SERVER01 -GroupName "Administrators"`

## The Code

```powershell
# ############################################################################# 
# NAME: FUNCTION-Get-LocalGroupAllMembers.ps1 
#  
# AUTHOR:	Francois-Xavier Cat 
# DATE:		2012/12/27 
# EMAIL:	fxcat@lazywinadmin.com
# WEBSITE:	LazyWinAdmin.com
# TWiTTER:	@lazywinadmin
#  
# COMMENT:	The following functions will gather all the local and domain members (direct or nested)
# 			By Default the function will query the Localhost and the Group Administrators.

# REQUIRE:	Quest Active Directory
# USAGE:	Get-LocalGroupAllMembers -ComputerName SERVER01 -GroupName "Administrators"
# 
# VERSION HISTORY 
# 1.0 2012.12.27 Initial Version. 
#
# ############################################################################# 

# Function to get LOCAL Group Member(s)
#	Members can be from the Local Machine or from the Domain.
#	Also, this function does not look for nested membership on local group(local group member of another local group)
#	since it is not supported by Microsoft (even if you can do it via net.exe, the permission won't work)
#	http://technet.microsoft.com/en-us/library/ee681621(v=ws.10).aspx
Function global:Get-LocalGroupMembership {
	[Cmdletbinding()]
	Param (
		[Parameter(ValueFromPipelineByPropertyName=$true,ValueFromPipeline=$true)]
		[string]$ComputerName = $env:COMPUTERNAME,
		
		[string]$GroupName = "Administrators"
		)
	
	# Create the array that will contains all the output of this function
	$Output = @()
	
	# Get the members for the group and computer specified
	$Group = [ADSI]"WinNT://$ComputerName/$GroupName" 
	$Members = @($group.psbase.Invoke("Members"))

	# Format the Output
	$Members | foreach {
		$name = $_.GetType().InvokeMember("Name", 'GetProperty', $null, $_, $null)
		$class = $_.GetType().InvokeMember("Class", 'GetProperty', $null, $_, $null)
		$path = $_.GetType().InvokeMember("ADsPath", 'GetProperty', $null, $_, $null)
		
		# Find out if this is a local or domain object
		if ($path -like "*/$ComputerName/*"){
			$Type = "Local"
			}
		else {$Type = "Domain"
		}
		
		$Details = "" | Select ComputerName,Account,Class,Group,Path,Type
		$Details.ComputerName = $ComputerName
		$Details.Account = $name
		$Details.Class = $class
        $Details.Group = $GroupName
		$details.Path = $path
		$details.Type = $type
		
		# Send the current result to the $output variable
		$output = $output + $Details
	}
	# Finally show the Output to the user.
	$output
}

# Function to get DOMAIN Group Member(s)
#	This function allow to dig into Active Directory to get all the members (direct or nested)
#	Members can only be from the Domain.
function global:Get-DomainGroupMembership {
	param ($GroupName,$ComputerName)
	#$ComputerName parameter here is only used for information purpose, to show in the output
	
	# Create the array that will contains all the output of this function
	$Output = @()
	
	# Check the current members of $GroupName
	$Members = $GroupName | Get-QADGroupMember
	
	# Check the Count of $members
	$MembersCount = $Members.count

	# If there is at least 1 member, do the following
	if ($MembersCount -gt 0){
		foreach ($member in $Members){
			switch ($Member.type){
				"user"{
					# Return the user information
					$Details = "" | Select ComputerName,Account,Class,Group,Domain,type,path
					$Details.ComputerName = $ComputerName
					$Details.Account = $member.name
					$Details.Class = $Member.type
					$Details.Group = $GroupName
					$details.domain = $member.domain.name
					$details.type = "Domain"
					$Details.path = $Member.CanonicalName
					$output = $output + $Details
                }#Switch user
				"group"{
					# Return the group object information
					$Details = "" | Select ComputerName,Account,Class,Group,Domain,type,path
					$Details.ComputerName = $ComputerName
					$Details.Account = $member.name
					$Details.Class = $Member.type
					$Details.Group = $GroupName
					$details.domain = $member.domain.name
					$details.type = "Domain"
					$Details.path = $Member.CanonicalName
					$output = $output + $Details
					# Return the members of the current group
					Get-DomainGroupMembership -GroupName $Member.name -ComputerName $ComputerName
				}#Switch group
			}#switch ($Member.type)
		}#foreach ($member in $Members)
	}#if ($MembersCount -gt 0)
	#Finally show the output
	$Output
}#end function Get-DomainGroupMembership

# Function to Get LOCAL and DOMAIN member(s) information
#	LOCAL Group Membership information is handled by the function Get-LocalGroupMembership
#	DOMAIN Group Membership information is handled by the function Get-DomainGroupMembership
function global:Get-LocalGroupAllMembers {
	param (
        [parameter(ValueFromPipeline=$true)]
	    [string]$ComputerName = "$env:computername",
        [string]$GroupName = "Administrators"
	)
# Create the array that will contains all the output of this function
$Output = @()

# Get the local administrators for the current ComputerName using the function declared 
#	earlier: Get-LocalGroupMembership
$LocalAdministrators = Get-LocalGroupMembership -ComputerName $ComputerName -GroupName $GroupName

# Let's now get information about each members, and members of members, etc...
foreach ($admin in $LocalAdministrators){

    # L O C A L #
    #	Local User
    if (($admin.Type -like "Local") -and ($admin.class -like "User")){
        $Details = "" | Select ComputerName,Account,Class,Group,Type,Path
        $Details.ComputerName = $admin.ComputerName
        $Details.Account = $admin.account
        $Details.Class = $admin.class
        $Details.Group = $admin.group
        $Details.Type = $admin.type
        $Details.Path = $admin.path
        $output = $output + $Details

    }
    #	Local Group
    if (($admin.type -like "Local") -and ($admin.class -like "group")){
        # Return the local group information before checking its members
        $Details = "" | Select ComputerName,Account,Group,Domain
        $Details.ComputerName = $admin.ComputerName
        $Details.Account = $admin.account
        $Details.Class = $admin.class
        $Details.Group = $admin.group
        $Details.type = $admin.type
        $Details.path = $admin.path
        $output = $output + $Details
		# Return the members of the current Local Group
        $localgroup = Get-LocalGroupMembership -GroupName $admin.account -ComputerName $ComputerName
        
        # If There is at least 1 member, do the following
        if ($localgroup.count -gt 0) {
            foreach ($localMember in $localgroup){
                $Details = "" | Select ComputerName,Account,Class,Group,Type,Path                       
                $Details.Account = $localMember.account
                $Details.Group = $admin.account # Here we are taking the name of the parent group
                $Details.ComputerName = $localMember.ComputerName
                $Details.Class = $localMember.class
                $Details.type = $localMember.type
                $Details.path = $localMember.path
                $output = $output + $Details
                }#foreach
            }#if
    }#if (Get-LocalGroupMember -group $admin.account -ComputerName $ComputerName)

    # D O M A I N #
    if ($admin.type -like "Domain"){
        # Get information about this object in the domain
        #	Here we just want to know if it is an User or Group.
        $ADObject = Get-QADObject $admin.account
        
        Switch ($ADObject.type) {
        
        	#	Domain User
            "user" {
                # Return the User information
                $Details = "" | Select ComputerName,Account,Class,Group,Domain,type,path
                $Details.ComputerName = $ComputerName
                $Details.Account = $ADObject.name
                $Details.Class = $ADObject.type
                $Details.Group = $admin.group
                $Details.domain = $ADObject.domain.name
                $Details.Type = $admin.type
                $Details.path = $ADObject.CanonicalName
                $output = $output + $Details

                }#user (switch)

            #	Domain Group
            "group"{
                # Return the Group information
                $Details = "" | Select ComputerName,Account,Class,Group,Domain,type,path
                $Details.ComputerName = $ComputerName
                $Details.Account = $ADObject.name
                $details.Class = $ADObject.type
                $Details.Group = $admin.group
                $Details.domain = $ADObject.domain.name
                $Details.Type = $admin.type
                $Details.path = $ADObject.CanonicalName
                $output = $output + $Details
                # Checking if the group has members, call the function declared ealier
                # Get-DomainGroupMembership
                Get-DomainGroupMembership -GroupName $ADObject.name -ComputerName $ComputerName
            }#group (switch)
        }#switch
    }#if ($admin.domain -notlike "$ComputerName"){
}#foreach ($admin in $LocalAdministrators){
$output
}#function global:Get-LocalGroupAllMembers

```
