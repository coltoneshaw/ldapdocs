
# Not syncing group members

1. Confirm the group is linked to Mattermost
  - System Console > Groups. Search for the group and you should see "linked"
  - If the group was recently linked, you'll need to rerun an LDAP sync.
2. Confirm the member in question is a part of Mattermost already.
  - System Console > Users > Search for that user
3. Confirm the login method of a user in question
  - System Console > Users > Search for that user
  - Look at the `Sign-in Method`. This **must be** LDAP or SAML, all other signin methods do not work for LDAP Groups
  - https://docs.mattermost.com/onboard/sso-openidconnect.html#how-can-i-use-ldap-attributes-or-groups-with-openid
4. Enable LDAP trace
5. Run an LDAP sync
6. Collect the trace logs
7. Search for teh `cn` of the user in question, you can also search by email. Copy this to a seperate text file.
8. Search for the `cn` of a group in question. Search by group name in the system console if you don't know the `cn`. Also copy this to another file.
9. Confirm the **exact same `cn`** for the user appears as a `memberof`, `member`, or `uniqueuser`
  - If there are any capitization differences here, it will not work. 