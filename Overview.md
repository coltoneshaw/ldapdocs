## LDAP Sync Phases
LDAP includes 12 sync phases where specific things happen. When debugging this it's extremely helpful to know what phase the issue is happening at. Sometimes you can know right away, sometimes you have to dig through the logs.

When looking in the logs you can search for strings like `phase10SyncGroupMembersToChannelsAndTeams` to identify these phases. They exist at the `debug` level. 

```json
{"timestamp":"2022-08-10 20:13:31.997 -05:00","level":"info","msg":"LDAP Sync Phase","caller":"ldap/ldap_sync_job.go:293","workername":"EnterpriseLdapSync","current_phase":"github.com/mattermost/enterprise/ldap.(*LdapSyncWorker).phase1GetLdapUsers-fm"}
{"timestamp":"2022-08-10 20:13:32.108 -05:00","level":"info","msg":"LDAP Sync Phase","caller":"ldap/ldap_sync_job.go:293","workername":"EnterpriseLdapSync","current_phase":"github.com/mattermost/enterprise/ldap.(*LdapSyncWorker).phase2GetSamlUsers-fm"}
{"timestamp":"2022-08-10 20:13:32.228 -05:00","level":"info","msg":"LDAP Sync Phase","caller":"ldap/ldap_sync_job.go:293","workername":"EnterpriseLdapSync","current_phase":"github.com/mattermost/enterprise/ldap.(*LdapSyncWorker).phase3GetLdapUsersFromLdap-fm"}
{"timestamp":"2022-08-10 19:13:32.094 -05:00","level":"info","msg":"LDAP Sync Phase","caller":"ldap/ldap_sync_job.go:293","workername":"EnterpriseLdapSync","current_phase":"github.com/mattermost/enterprise/ldap.(*LdapSyncWorker).phase4SyncLdapUsers-fm"}
{"timestamp":"2022-08-10 19:04:32.183 -05:00","level":"info","msg":"LDAP Sync Phase","caller":"ldap/ldap_sync_job.go:293","workername":"EnterpriseLdapSync","current_phase":"github.com/mattermost/enterprise/ldap.(*LdapSyncWorker).phase5SyncSamlUsers-fm"}
{"timestamp":"2022-08-10 19:04:32.303 -05:00","level":"info","msg":"LDAP Sync Phase","caller":"ldap/ldap_sync_job.go:293","workername":"EnterpriseLdapSync","current_phase":"github.com/mattermost/enterprise/ldap.(*LdapSyncWorker).phase6GetGroups-fm"}
{"timestamp":"2022-08-10 19:04:32.410 -05:00","level":"info","msg":"LDAP Sync Phase","caller":"ldap/ldap_sync_job.go:293","workername":"EnterpriseLdapSync","current_phase":"github.com/mattermost/enterprise/ldap.(*LdapSyncWorker).phase7GetLdapGroups-fm"}
{"timestamp":"2022-08-10 19:04:32.601 -05:00","level":"info","msg":"LDAP Sync Phase","caller":"ldap/ldap_sync_job.go:293","workername":"EnterpriseLdapSync","current_phase":"github.com/mattermost/enterprise/ldap.(*LdapSyncWorker).phase8SyncGroups-fm"}
{"timestamp":"2022-08-10 19:04:32.706 -05:00","level":"info","msg":"LDAP Sync Phase","caller":"ldap/ldap_sync_job.go:293","workername":"EnterpriseLdapSync","current_phase":"github.com/mattermost/enterprise/ldap.(*LdapSyncWorker).phase9SyncGroupMembership-fm"}
{"timestamp":"2022-08-10 19:04:32.941 -05:00","level":"info","msg":"LDAP Sync Phase","caller":"ldap/ldap_sync_job.go:293","workername":"EnterpriseLdapSync","current_phase":"github.com/mattermost/enterprise/ldap.(*LdapSyncWorker).phase10SyncGroupMembersToChannelsAndTeams-fm"}
{"timestamp":"2022-08-10 19:04:33.075 -05:00","level":"info","msg":"LDAP Sync Phase","caller":"ldap/ldap_sync_job.go:293","workername":"EnterpriseLdapSync","current_phase":"github.com/mattermost/enterprise/ldap.(*LdapSyncWorker).phase11SyncTeamRoles-fm"}
```

#### Phase 1 - Get LDAP users
This is just making a call out to the Mattermost DB and finding all users with `auth_method = ldap` set.

#### Phase 2 - Get SAML Users
Same as the above, looking for all users with `auth_method = saml` set.

#### Phase 3 - Get LDAP users from LDAP
This reaches out to the LDAP service and fetches all the related users based on the configured user filter.

#### Phase 4 - Sync LDAP Users
Here we look for the LDAP user returned from the LDAP server compared with what we have on Mattermost for LDAP from phase 1. 

We update the data for the users itself. Looking at the matches we found in phase 1 and 3. Updating the database with the associated values, and moving on. If a user was found in Phase 1 but NOT in phase 3 then they are deactivated.

#### Phase 5 - Get SAML Users
Here we look for the data from Phase 2 and match them with the data that came back from phase 3. This is just a filter job that works to identify the SAML users based on their `auth_data` or email. If `auth_data` is blank, it uses email.

We update the data for the users itself. Looking at the matches we found in phase 2 and 3. Updating the database with the associated values, and moving on. If a user was found in Phase  2 but NOT in phase 3 then they are deactivated. SAML being the exception here that they will get deactivated only if the `Sync with LDAP` flag is set. 


#### Phase 7 - Get Groups
This looks at the Mattermost database and fetches the related linked groups.

#### Phase 8 - Get LDAP Groups
Mattermost reaches out to LDAP and fetches all the groups that match the group filter.

#### Phase 9 - Sync Group Membership

**This is the phase you see the most bugs at**

This is responsible for matching the users based on their **DN** attribute to the groups only looking at `member` / `memberof` and `uniquemember` attributes on LDAP. 

#### Phase 10 - Sync Group Members to channels and Teams

This is similar to Phase [4/5](https://handbook.mattermost.com/company/about-mattermost/list-of-terms#0-5-1-5-2-5-3-5-4-5-5-5) where it goes through the list and finds the related people and syncs them where they belong. If a channel is a group sync channel and the user is no longer on the group, they are removed here.

#### Phase 11 - Sync Team roles

Simple just syncing admin / guest roles

#### Phase 12 - Sync Channel Roles
Same as above but for channels. 


## Trace Logs

All the good LDAP logs come out at the `stdout` level now (groan). Turn on `LDAP.trace = true` in the config.json and use the below env var to set this up. You'll want to restart Mattermost afterwards.

```bash
export MM_LOGSETTINGS_TRACE=true
export MM_LOGSETTINGS_ADVANCEDLOGGINGCONFIG="{\"console-log\":{\"Type\":\"console\",\"Format\":\"json\",\"Levels\":[{\"ID\":10,\"Name\":\"stdlog\",\"Stacktrace\":false},{\"ID\":5,\"Name\":\"debug\",\"Stacktrace\":false},{\"ID\":4,\"Name\":\"info\",\"Stacktrace\":false,\"color\":36},{\"ID\":3,\"Name\":\"warn\",\"Stacktrace\":false,\"color\":33},{\"ID\":2,\"Name\":\"error\",\"Stacktrace\":true,\"color\":31},{\"ID\":1,\"Name\":\"fatal\",\"Stacktrace\":true},{\"ID\":0,\"Name\":\"panic\",\"Stacktrace\":true}],\"Options\":{\"Out\":\"stdout\"},\"MaxQueueSize\":1000},\"file-log\":{\"Type\":\"file\",\"Format\":\"json\",\"Levels\":[{\"ID\":10,\"Name\":\"stdlog\",\"Stacktrace\":false},{\"ID\":5,\"Name\":\"debug\",\"Stacktrace\":false},{\"ID\":4,\"Name\":\"info\",\"Stacktrace\":false},{\"ID\":3,\"Name\":\"warn\",\"Stacktrace\":false},{\"ID\":2,\"Name\":\"error\",\"Stacktrace\":true},{\"ID\":1,\"Name\":\"fatal\",\"Stacktrace\":true},{\"ID\":0,\"Name\":\"panic\",\"Stacktrace\":true}],\"Options\":{\"Compress\":true,\"Filename\":\"logs/traceLogs.json\",\"MaxAgeDays\":15,\"MaxBackups\":3,\"MaxSizeMB\":100},\"MaxQueueSize\":1000}}"
```

## Good to Knows

- Always check that the sync is correctly identifying the users. You'll see a log message like: `"EnterpriseLdapSync","num_ldap_users":212`. This is the mattermost users with LDAP it's identified. 
- The attributes on both SAML and LDAP are **usually** case sensitive. It's important to check this. `samaccountname` does not always equal `sAMAccountName`.
- If there is no ID attribute configured it will always fall back to email. This can be found by `auth_data` being null in the database.
- **OPEN ID DOES NOT USE LDAP GROUPS AND WILL NOT USE LDAP GROUPS. IF YOU HAVE A CUSTOMER USING OPEN ID TELL THEM NOT TO.**
- If you unlink a group within System Console > Groups Mattermost will assume everyone left that group of their own choosing. So, relinking and resyning will **not** add that user back. You need to run `mmctl ldap sync --include-removed-members`.
- You can force add someone to a group by using `/invite @username` in a group synced channel / team sometimes, if they're not being added correctly. You can also make a database entry and force fix this, the table is `groupmembers`
- If you're using LDAP and SAML together the ID attributes must match. They don't have to be the same key name, but the value must match.
- `ObjectGUID` used as a SAML ID attribute is almost always a bad idea. We do not parse it properly when compared with LDAP and can cause a mismatch. 
- If they use email as their ID attribute, and it's changed, then it will need to be manually updated in the database or they will have a second person created for them.
- `email` as their `username` attribute is almost always a bad idea and can cause mismatches and other errors. You'll see we replace the `@` with a `-`. But on top of that when searching for their information we've seen countless times where the usernames are not searched properly when it's derived from an email string.
- My first troubleshooting step is to walk the trace logs with the customer. Knowing how the sync process works from above you can see where the mismatches occur at. If it's a user not being added to a group, compare the group response with the user response. Look at the `member` attribute from the group, and the DN of the user. See if Mattermost logs anything else for that user in the other phases. 
- Invalid data in LDAP can break the whole sync. Example, if one user has a strange `DN` attribute in comparison, this can cause every subsequent user to be matched incorrectly. 
- I like to tell customers to increase the time between LDAP syncs as it's usually not mission critical syncs. It's heavy on their system. Also, if there is an error during the sync where only a portion of the users are returned, then Mattermost will still action on that sync. Meaning half their users could be deactivated. To reduce the impact of this, I just increase the interval between syncs.
- Did I mention OpenID does not use LDAP groups yet? Because it doesn't.
- SAML does not use the LDAP user roles. Ex, if you have the LDAP Admin filter set to find a specific group of people, if their `auth_service` is set to SAML they will not be promoted. You must use the SAML admin filter for SAML and LDAP admin filter for LDAP users.
- SAML attributes will only update on login. If they have an old ID attribute or other value and it's not syncing with their LDAP service, the user needs to log out and back in to get that new update. 
