# Digital Strolling Pilot Event Log Evaluation

This repository contains the plain data of our real user evaluation we conduced with a research prototype for the Städel Museum. We hereby release the data in an open matter.

The research prototype allows to discover content in two ways: Firstly, by direct, conventional search, secondly, by a new paradigm we called digital strolling. Subject of evaluation is to compare the performance of both discovery patterns. 

## Preparations that happened before publication

### Extraction of relevant user logs that took part in pilot test

	cat userevents.log | grep ";Mohawk;" >> userevents_pilot_2015-04-14.log
	cat userevents.log | grep ";afflict;" >> userevents_pilot_2015-04-14.log
	cat userevents.log | grep ";berserk;" >> userevents_pilot_2015-04-14.log
	cat userevents.log | grep ";bookshelf;" >> userevents_pilot_2015-04-14.log
	cat userevents.log | grep ";dashboard;" >> userevents_pilot_2015-04-14.log
	cat userevents.log | grep ";egghead;" >> userevents_pilot_2015-04-14.log
	cat userevents.log | grep ";orca;" >> userevents_pilot_2015-04-14.log
	cat userevents.log | grep ";seabird;" >> userevents_pilot_2015-04-14.log
	cat userevents.log | grep ";tempest;" >> userevents_pilot_2015-04-14.log
	cat userevents.log | grep ";woodlark;" >> userevents_pilot_2015-04-14.log
	cat userevents.log | grep ";sweatband;" >> userevents_pilot_2015-04-14.log
	cat userevents.log | grep ";adrift;" >> userevents_pilot_2015-04-14.log
	cat userevents.log | grep ";skullcap;" >> userevents_pilot_2015-04-14.log


### Remove confidential / restricted parts from events
	cat userevents_pilot_2015-04-14.log | grep -v ";SearchPerformed;" | grep -v ";ClickOnResource;" | awk -F ';' '{print $1";"$2";"$3";"$4";"}' >> userevents_pilot_2015_04_14_anon.log
	cat userevents_pilot_2015-04-14.log | grep ";SearchPerformed;" | sed -e "s/keyword\":\"[^\".]*\"/keyword\":\"anonymised\"/" | sed -e "s/bowTitle\":\"[^\".]*\"/bowTitle\":\"anonymised\"/"  >> userevents_pilot_2015_04_14_anon.log
	cat userevents_pilot_2015-04-14.log | grep ";ClickOnResource;" | sed -e "s/Resource_museum_exhibit_Staedel_[0-9]*/anonymised/" >> userevents_pilot_2015_04_14_anon.log


## Preparations that can be reproduced
### Import in SQLite

	sqlite3 events.db << EOF
	create table logs (timestamp text, user text, sid text, action text, payload text);
	.separator ";"
	.import userevents_pilot_2015_04_14_anon.log logs
	EOF


### Create indices and convenience view

	create index action_index on logs (action)
	create index timestamp_index on logs (timestamp)
	create index user_index on logs (user)
	create index payload_index on logs (payload)
	
	create view logsT as select *, (select action from logs b where b.timestamp > aa.nexttimestamp and b.user = aa.user and b.action <> 'Response' and b.action <> 'Request' order by timestamp limit 1) as nextnextaction from (
	select *,
	datetime(timestamp) as timestampDT,
	(select action from logs b where b.timestamp > a.timestamp and b.user = a.user and b.action <> 'Response' and b.action <> 'Request' order by timestamp limit 1) as nextaction,
	(select payload from logs b where b.timestamp > a.timestamp and b.user = a.user and b.action <> 'Response' and b.action <> 'Request' order by timestamp limit 1) as nextpayload,
	(select timestamp from logs b where b.timestamp > a.timestamp and b.user = a.user and b.action <> 'Response' and b.action <> 'Request' order by timestamp limit 1) as nexttimestamp,
	(select datetime(timestamp) from logs b where b.timestamp > a.timestamp and b.user = a.user and b.action <> 'Response' and b.action <> 'Request' order by timestamp limit 1) as nexttimestampDT
	from logs a where user in ('berserk','woodlark','sweatband','seabird','tempest','dashboard','egghead','Mohawk','orca','afflict','bookshelf','adrift','skullcap') order by user, timestamp) aa
	order by user, timestamp




## Evaluations
### Performed Searches

	select user, action, COUNT(action) as events
	from LogsT where
	action like 'SearchPerformed' and
	payload not like '%"bowTitle":%'
	group by user, action


user|action|count
---|---|---
Mohawk|SearchPerformed|24
afflict|SearchPerformed|6
berserk|SearchPerformed|3
bookshelf|SearchPerformed|39
dashboard|SearchPerformed|81
egghead|SearchPerformed|3
orca|SearchPerformed|18
seabird|SearchPerformed|3
tempest|SearchPerformed|12
woodlark|SearchPerformed|30




### Performed Strollings

	select user, count(strollingstep) from (
	select user, action, 1 as StrollingStep from logsT where
	action in ('ClickOnResource') and
	(payload like '%"type":"topic"%' OR payload like '%"type":"timeline"%' OR payload like '%"type":"link"%') and
	nextpayload not like '%"type":"freeSearch"%'
	order by user
	) group by user order by user

user|count
---|---
Mohawk|9
adrift|48
afflict|186
berserk|21
bookshelf|114
dashboard|30
egghead|21
orca|78
skullcap|51
sweatband|12
tempest|63
woodlark|57


### Discoveries per Searches
alle resourcemaximised, die direct nach clickonresource kamen, was direkt nach searchperformed kam wäre ein fund beim suchen

	select user, count(action) from logsT where action = 'SearchPerformed' and nextaction = 'ClickOnResource' and nextnextaction = 'ResourceMaximized' group by user
	

user|funde
---|---
dashboard|6
tempest|3
woodlark|9


### Discoveries per Strolling
alle resourcemaximised, die nach einem clickonresource vom type Topic wären ein fund beim schlendern

	select user, count(action) from logsT where action = 'ClickOnResource' and payload like '%"type":"topic"%' and nextaction = 'ResourceMaximized' group by user

user|funde
---|---
adrift|6
afflict|12
bookshelf|3
egghead|6
orca|3
skullcap|3
tempest|6
woodlark|15

	





## Caveat: Event Timestamp resolution
All evaluations are based on a one second timeframe resolution. Subsequent events that happen < 1 second are possibly ignored. `b.action <> 'Response' and b.action <> 'Request' ` tries to compensate that for trivial events.

