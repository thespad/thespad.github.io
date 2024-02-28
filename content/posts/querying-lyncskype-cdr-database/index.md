---
title: Querying Lync/Skype CDR Database
author: Adam
date: 2016-08-16T10:11:44+00:00
aliases: ["/angry/880"]
tags:
  - call detail recording
  - CDR
  - compliance
  - lync
  - lync 2013
  - scripts
  - skype for business
  - SQL

---
Lync and Skype for Business offer [Call Detail Recording][1], essentially just a record of all calls to, from and within your Lync/Skype infrastructure. Only problem is it's a pain to query. Now there are some SSRS custom reports available as well as the default templates but sometimes you just want to get in there and pull data quickly from SQL.

This, for example, will get all audio calls from the last 10 days:

```sql
select [SessionIdTime],[ResponseTime],[SessionEndTime],DATEDIFF(ss,[ResponseTime],[SessionEndTime]) as Duration,u1.[UserUri] as User1Uri,[User1Id],u2.[UserUri] as User2Uri,[User2Id],[TargetUserId],[SessionStartedById],[OnBehalfOfId],[ReferredById] FROM [LcsCDR].[dbo].[SessionDetails] s left outer join [LcsCDR].[dbo].[Users] u1 on s.[User1Id] = u1.[UserId] left outer join [LcsCDR].[dbo].[Users] u2 on s.[User2Id] = u2.[UserId] where [User1Id] != [User2Id] and [MediaTypes] = 16 and [ResponseTime] >= dateadd(dd,0, datediff(dd,10, getDate())) order by [ResponseTime] desc
```

Change the number value here: `datediff(dd,10, getDate()))` to display a different number of days worth of calls. Change the value of `[MediaTypes]` to alter the call type you're searching for as per the [database schema][2].

These examples will get you the user ID for a given SIP URI or external number:

```sql
select [UserId] from [LcsCDR].[dbo].[Users] where [UserUri] like '+441212001234%'
```

```sql
select [UserId] from [LcsCDR].[dbo].[Users] where [UserUri] like 'jimbob@example.com%'
```

With those UserIDs you can filter calls to a user (in this case user ID 147):

```sql
select [SessionIdTime],[ResponseTime],[SessionEndTime],DATEDIFF(ss,[ResponseTime],[SessionEndTime]) as Duration,u1.[UserUri] as User1Uri,[User1Id],u2.[UserUri] as User2Uri,[User2Id],[TargetUserId],[SessionStartedById],[OnBehalfOfId],[ReferredById] FROM [LcsCDR].[dbo].[SessionDetails] s left outer join [LcsCDR].[dbo].[Users] u1 on s.[User1Id] = u1.[UserId] left outer join [LcsCDR].[dbo].[Users] u2 on s.[User2Id] = u2.[UserId] where [User1Id] != [User2Id] and [MediaTypes] = 16 and [User2Id] = '147' order by [ResponseTime] desc
```

Or from a user:

```sql
select [SessionIdTime],[ResponseTime],[SessionEndTime],DATEDIFF(ss,[ResponseTime],[SessionEndTime]) as Duration,u1.[UserUri] as User1Uri,[User1Id],u2.[UserUri] as User2Uri,[User2Id],[TargetUserId],[SessionStartedById],[OnBehalfOfId],[ReferredById] FROM [LcsCDR].[dbo].[SessionDetails] s left outer join [LcsCDR].[dbo].[Users] u1 on s.[User1Id] = u1.[UserId] left outer join [LcsCDR].[dbo].[Users] u2 on s.[User2Id] = u2.[UserId] where [User1Id] != [User2Id] and [MediaTypes] = 16 and [User1Id] = '147' order by [ResponseTime] desc
```

Or both:

```sql
select [SessionIdTime],[ResponseTime],[SessionEndTime],DATEDIFF(ss,[ResponseTime],[SessionEndTime]) as Duration,u1.[UserUri] as User1Uri,[User1Id],u2.[UserUri] as User2Uri,[User2Id],[TargetUserId],[SessionStartedById],[OnBehalfOfId],[ReferredById] FROM [LcsCDR].[dbo].[SessionDetails] s left outer join [LcsCDR].[dbo].[Users] u1 on s.[User1Id] = u1.[UserId] left outer join [LcsCDR].[dbo].[Users] u2 on s.[User2Id] = u2.[UserId] where [User1Id] != [User2Id] and [MediaTypes] = 16 and ([User1Id] = '147' or [User2Id] = '147') order by [ResponseTime] desc
```

You can of course filter on `[ResponseTime]` or even `[InviteTime]` to limit your query by timeframe similarly to the first example. If you have a nose through the SessionDetails table in the database you'll be able to see all the additional columns that you can also query but IMO the ones I've included tend to be the important ones - who the call is to/from, when it was made, how long it was and whether it was made on behalf of or referred by another user. Some columns you may find useful are `[IsUser1Internal]` and `[IsUser2Internal]` which will tell you if one or both parties were connecting via one of your Edge servers rather than internally and `[ResponseCode]` which will tell you whether or not the call was successful (a "200" code means success).

 [1]: https://technet.microsoft.com/en-us/library/jj688079.aspx
 [2]: https://technet.microsoft.com/en-us/library/gg398589.aspx
