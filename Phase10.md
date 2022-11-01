# Phase 10-  Sync Group Members to Channels and Teams

This is the bulk of what you see on the frontend. This takes the existing groups and syncs the end users to their specific teams and channels associated with that group.

Errors: 
- `phase10SyncGroupMembersToChannelsAndTeams`
  - ??
  - Appears to be a general error if it cannot find the last ldap job. Seen with the error below.
- `app.job.get_newest_job_by_status_and_type.app_error`
  - Failed to fetch the last LDAP job from the store.
- `ent.ldap.syncronize.populate_syncables`
  - Failed to create the team or channel memberships
- `ent.ldap.syncronize.delete_group_constained_memberships`
  - Failed on the delete group members step.



# Process
1. Fetches the last LDAP job from the store and finds the `StartAt`
2. Identifies if you've passed `includeRemovedMembers`
3. Makes a call to `CreateDefaultMemberships` with the `since` and `includeRemovedMembers` flag. 
4. Makes a call to `DeleteGroupConstrainedMemberships` just passing the `request.Context`

## CreateDefaultMembership

This function calls two things (`createDefaultTeamMemberships` and `createDefaultChannelMemberships` ), both with `since` / `includedRemovedMembers`

[Function](https://github.com/mattermost/mattermost-server/blob/5e69c6b02f604ce1aecb4c22cc58f9412cc685a2/app/syncables.go#L118)


### Create Default Team Memberships

[createDefaultTeamMemberships](https://github.com/mattermost/mattermost-server/blob/5e69c6b02f604ce1aecb4c22cc58f9412cc685a2/app/syncables.go#L86) adds users to teams based on their group memberships and how those groups are configured to sync with teams for group members on or after the given timestamp. 

- If includeRemovedMembers is true, then team members who left or were removed from the team will be re-added; otherwise, they will not be re-added.

#### Process

##### Step 1
Fetches all the team members it needs to add. [`TeamMembersToAdd`](https://github.com/mattermost/mattermost-server/blob/5e69c6b02f604ce1aecb4c22cc58f9412cc685a2/app/group.go#L503)
  - This included the `since` / `includedRemovedMembers`. The `TeamID` from the function is blank.
  - This happens at the SQL store level. [Actual Command](https://github.com/mattermost/mattermost-server/blob/b93a0139342d271ddacc2fc947e798945c23fce1/store/sqlstore/group_store.go#L864)
  - Returns `[{UserID: string, TeamID: string}]`. Important to note that one user can be on this multiple times, one for each team.

The essence of this query is:

```sql
SELECT 
  GroupMembers.UserId as UserID, GroupTeams.TeamId as TeamID
FROM GroupMembers,
JOIN GroupTeams on GroupTeams.GroupId = GroundMembers.GroundId,
JOIN UserGroups ON UserGroups.Id = GroupMembers.GroupId,
JOIN Teams ON Team.Id = GroupTeams.TeamId,

-- Below are REMOVED if IncludeRemovedMembers is true
LEFT OUTER JOIN TeamMembers 
  ON TeamMembers.TeamId = GroupTeams.TeamId  AND TeamMembers.UserId = GroupMembers.UserId,

-- back to the standard query
WHERE
  UserGroups.DeleteAt = 0 AND
  GroupTeams.DeleteAt = 0 AND
  GroupTeams.AutoAdd = true AND
  GroupMembers.DeleteAt = 0 AND
  Teams.DeleteAt = 0

-- Below are REMOVED if IncludeRemovedMembers is true
  AND TeamMembers.UserID is null
  OR GroupMembers.CreateAt > {since} -- since here is the last LDAP sync from the jobs table.
  OR GroupTeams.UpdateAt > {since}
```

Errors:
- `team_members_to_add_tosql` 
  - if the specific query above fails
- `failed to find UserTeamIDPairs`
  - Happens when `err = s.GetMasterX().Select(&teamMembers, query, params...)` fails. Not 100% sure what this is.

##### Step 2
Takes the response from the above query and loops over it calling [`AddTeamMember`](https://github.com/mattermost/mattermost-server/blob/757c4e041a3884d79b0b4909970f5f555e19e435/app/team.go#L1081).
- Params are `userid` and `teamid` from the above step. 

1. Make a call to [`AddUserToTeam`](https://github.com/mattermost/mattermost-server/blob/757c4e041a3884d79b0b4909970f5f555e19e435/app/team.go#L581). passing it the `TeamID` and `UserID`
2. Calls [JoinUserToTeam](https://github.com/mattermost/mattermost-server/blob/757c4e041a3884d79b0b4909970f5f555e19e435/app/team.go#L785)
  - Adds a user to the team and outputs a few errors based on if they're a member of the team, domain, or team size. 
  - Calls the function [here](https://github.com/mattermost/mattermost-server/blob/757c4e041a3884d79b0b4909970f5f555e19e435/app/teams/teams.go#L134)

For every team member that gets added you'll see `added teammember` and their `user_id` and `team_id`.

Success is returning `nil`.

Errors:
- `api.team.join_user_to_team.allowed_domains.app_error`
  - User not added to team - the domain associated with the user is not in the list of allowed team domains

### Create Default Channel Memberships

[`createDefaultChannelMemberships`](https://github.com/mattermost/mattermost-server/blob/5e69c6b02f604ce1aecb4c22cc58f9412cc685a2/app/syncables.go#L21) adds users to channels based on their group memberships and how those groups are configured to sync with channels for group members on or after the given timestamp. ChannelID for this is `nil`. 

If includeRemovedMembers is true, then channel members who left or were removed from the channel will be re-added; otherwise, they will not be re-added.

#### Process

##### Step 1
1. Calls [`ChannelMembersToAdd`](https://github.com/mattermost/mattermost-server/blob/5e69c6b02f604ce1aecb4c22cc58f9412cc685a2/app/group.go#L518)
2. This calls the SQL store within the GroupStore [here](https://github.com/mattermost/mattermost-server/blob/5e69c6b02f604ce1aecb4c22cc58f9412cc685a2/store/sqlstore/group_store.go#L897)
  - This returns a `[{UserID: string, ChannelID: string}]`

```sql
SELECT 
  GroupMembers.UserID as UserID, GroupChannels.ChannelId as ChannelID
From GroupMembers,
JOIN GroupChannels ON GroupChannels.GroupId = GroupMembers.GroupId,
JOIN UserGroups ON UserGroups.Id = GroupMembers.GroupId,
JOIN Channels ON Channels.Id = GroupChannels.ChannelId,

-- Everything between the start/end block is run if the IncludeRemovedMembers flag is not present or is false. 
-- start
LEFT OUTER JOIN ChannelMemberHistory 
  ON ChannelMemberHistory.ChannelId = GroupChannels.ChannelId AND ChannelMemberHistory.UserId = GroupMembers.UserId
-- end

-- Regular Query
WHERE
  UserGroups.DeleteAt = 0
  AND GroupChannels.DeleteAt = 0
  AND GroupChannels.AutoAdd = true
  AND GroupMembers.DeleteAt = 0
  AND Channels.DeleteAt = 0

  -- Everything between the start/end block is run if the IncludeRemovedMembers flag is not present or is false. 
  -- start
  AND ChannelMemberHistory.UserId is null,
  AND ChannelMemberHistory.LeaveTime is null
  OR GroupMembers.CreateAt > since -- last ldap sync time
  OR GroupChannels.UpdateAt > since
  -- end
```

##### Step 2
Loops through each member returned and adds them to the relevant team then channel. 

1. Gets the channel
2. Gets the Team member
  - If they aren't a team member, we attempt to add them to the team.
3. Adds the user to the channel. - [`AddChannelMember`](https://github.com/mattermost/mattermost-server/blob/5e69c6b02f604ce1aecb4c22cc58f9412cc685a2/app/channel.go#L1546)