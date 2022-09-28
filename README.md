# Reminders Tool 
This repo contains the backend application that allows employees to create reminders, and return them to some dashboard. A user can create a reminder for a given date X, and at midnight of date X they should begin to see the reminder they created being returned to their dashboard. 

This tool should provide the following functionality to the user: 
1. Allow the employee to create a reminder with some text that will appear on a given date 


2. Return all active reminders for the current day for the employee 


3. Provide the employee the functionality to mark a reminder as `Done` effectively removing it from the list of reminders needed to be shown to them


4. Roll over any reminders from previous days that have not yet been marked as `Done` 


5. The employee should be able to provide an email address to also send the reminder to within 5 minutes of the scheduled reminder time. An email should only be sent once per reminder. Emails should only be sent if the reminder was not yet acknowledged/marked as `Done`


6. The employee can also define an occurrence rule which is the frequency and interval at which the reminder should be repeated e.g Repeat reminder X daily every 5 days 


7. If a single reminder has 10 occurrences each one is to be marked as `Done` individually and emails are to be sent individually.  

Example : 
```
>   On September 29th 2022 an employee creates reminder for October 1st 2022 
    with following properties: 
        time                    = 5pm
        message                 = "Remember to finish writing tests!"
        email                   =  some.email@gmail.com
        recurrence_frequency    = 2    <--- this means MONTH as the unit
        recurrence_interval     = 1    <--- this amount * unit * = repeat every 1 month 
        
>   If employee checks their dashboard on September 30th, they should not see the reminder 
    yet as the day of the reminder has not started. 

>   At 00:00am on October 1st, the reminder should now be visible 
    to the user on their dashboard 
    
>   at 5pm the user has not marked the reminder as Done, they should receive an email to their address with the 
    reminder 
    
>   The user has not marked the reminder as complete by EOD, it should appear in the dashboard the next day at 00:00am 
    for October 2nd 
    
>   Fast forward to Oct 31st, if the user still hasnt marked the reminder as done, 
    it should still appear in their dashboard. Given the recurrence rule the user added when intially 
    creating the reminder, at 00:00am on Nov 1st, there should now be 2 reminders displayed to the user :
    the original and the one created as an occurrence rule 
    
>   If the user hasnt marked the new reminder as Done by 5pm, an email will 
    be sent to the user address. Note that while the original reminder is still active, 
    the user will not receive a second email for that original instance of the reminder. 
    
```


## How is it designed 
### System Architecture
At a high level, the application is comprised of 3 main components: 
- a kotlin server built using the spring framework 
- a postgresql database running in docker 
- a (stubbed out) mailer service 

The flow starts by the user creating a reminder and the first occurrence for that reminder. If the
user does not create a recurrence rule and never uses the application again, this will be the only reminder
and occurrence in the application ever for this user. 

If however the user did define a occurrence rule for their reminder what happens under the hood is a scheduled task 
running every 10 seconds that is looking for any reminders with recurrences rules set, it finds the most recent occurrence 
and then calculates the date that the next occurrence for that reminder needs to be created for. 

There is also a scheduled task running every 5 minutes to find reminders that emails need to be sent for. 

### Code Design 
The code is comprised of these main components: 
- API - these are the entry points into the application for the user, so far the only endpoints are : 
  - `POST /reminder?employeeId`: this creates a new reminder 
  - `GET /reminder?employeeId`: this returns all reminders ever created by a user 
  - `GET /occurrences?employeeId`: This (should - there is a bug) returns all the active reminders for the user 
  - `PUT /occurrences/{id}`:  This marks as reminder as `Done` as per the requirements


- Use cases - the API interacts with use cases to interact with our domain modal. These use cases are the different interactions we have with our two models : `Reminders` and  `Occurrences`. These use cases are cool cause they describe essentially everything the system should do. The use cases we have so far are : 
  - `RemindersUseCase`:
    -  `create` - creates a new reminder  
    -  `find` - finds all reminders ever created for a given user 
    - 
  - `OccurrencesUseCase`: 
    -  `recur` - creates a new occurrence for a reminder
    -  `find` - finds all active occurrences for given day
    -  `email` - finds all reminders it needs to send an email (this is mocked)
    -  `complete` - this marks a reminder as `Done` or `Acknowledged` seems to be the term used on backend 


- Repositories - use cases interact with the domain model via repositories, this gives a good abstraction from the DB and allows us to test differently sections of the code independently by mocking out the repo. There are two repos :
  - `RemindersRepository` - this has the following functions: 
    - `create` - create command that is meant to create a new reminder, 
    - `findAll` - this should return a list of (all) reminders for a given user id 
    - `findBy`- this should return an individual reminder by the reminder_id
    - 
  - `OccurrencesRepository` - this has the following functions:
      - `create` - create command that is meant to create a new occurrence for a given reminder
      - `findAt(Instant)` - this should return a list of all occurrences created before some timestamp <--- bug in  here, it is not filtering out the acknowledged ones 
      - `findAt(Instant, EmployeeId)` - this should return a list of all occurrences for a given user created before some timestamp
      - `findBy` - this should return an individual occurrence by the occurrence id
      - `getInstantForNextReminderOccurrences` - this should return a map where the key is a previous occurrence id and the value is the timestamp for the next occurrence of the original reminder
      - `markAsNotified` - this should set a field on the Occurrences table indicating an email has been sent 
      - `acknowledge` - this should set a field on the Occurrences table indicating the user has marked the occurrence as `Done`

- `PostgresRemindersRepository` & `PostgresOccurrencesRepository` - in these repos we are actually connecting to the DB and querying and updating. 

### Flow for adding a new feature 
Given all the above, imagine we needed to implement the ability to delete a reminder and all its occurrences. The different components then we might need are : 
- A new endpoint in the Reminders resource , perhaps `DELETE /reminders/{id}`
- A new use case for both Reminders and Occurrences : 
  - `RemindersUseCase.deleteReminder` - this should delete the reminder 
  - `OccurrencesUseCase.deleteAllOccurrences` - if the original reminder was set up to have a recurring rule, we also want to delete any occurrences also
- In our repositories, we want to new commands to execute: 
  - `RemindersRepository.deleteReminder` - this is the abstracted function that the PostgresRemindersRepository will implement 
  - `RemindersRepository.deleteOccurrences` - this is the abstracted function that the PostgresOccurrencesRepository will implement



---

# Blockers to going to production 
## Bug fixes 
There are a couple of bugs I would want to fix before launching this application to production. 

### Reminder bug 
Inside the query for finding all active Occurrences for the current day, the query checks for any occurrences that are 
still active and their timestamp is less than now. This means if you had a reminder set for 5pm for today, and you are 
checking the dashboard at 9am, it would not be returned. This breaks the requirement of having a reminder show up on the 
dashboard at `00:00am`. 

### findAt in occurrences 
The query for returning all unacknowledged occurrences has a comment mentioning it suppose to return all unacknowledged
occurrence, however it is missing a check on the isAcknowledged field. 

### It is possible to create reminders in the past 
It is possible to create reminders with a date of anytime before the current moment. If this happens, a new reminder 
occurrence will be created for every (frequency x interval) between the date user selected and current date. The API
should enforce that the time of the reminder cannot be in the past   

### Additional APIs 
There is also some features I think could definitely improve this API and offer more functionality to the (would be)
dashboard: 
- `GET /occurrences` get all occurrences (acknowledged or not and not just for current day) - it might be useful for a user to see all the different reminders they have had in the past, and all the ones due to come in the future 
- `DELETE /reminder` delete reminder and all occurrences - this might be useful for a user if they have a reminder thats only useful for a certain amount of time e.g 3 weeks, they might set it up initally to be a recurring reminder every day and then after 3 weeks they mightnt need it anymore - they should be able to delete a reminder and all associated occurrences 

## Security
There is no authentication within the application, any user can hit the acknowledge endpoint for another users 
occurrence if they know their UUID. Also any user can see any other users occurrences if they know their UUID.
On the above point, we should have different endpoints to authenticate the user and protect other endpoints to 
ensure only logged in users can see and change their own reminders/occurrences. 


## Potential Performance Improvements
I think there is plenty of opputunities for performance improvements, some might be overkill and not necessary depending
scale, and others should be done anyway: 

- **Caching** : use caching to store reminders for employees so we dont have to keep going to DB if nothings changed e.g Redis 


- **Sync vs Async** : the API calls to create a reminder and ack an occurrence are currently synchronous - do they need to be? We could stick a message on a queue to create a reminder and then just return a 201 to the user. Also for things like the MailerService, given that (in theory) we are connecting to an external client, we could be subject to long time outs or just slow SLAs, we can put messages on a queue and then a task handlers reads from the queue and sends to the mail service. Could use something like SQS or Kafka for this.


- **Indexes** - there is currently no indexes on either the Reminders or Occurrences tables, as we scale, this will make querying the DB slower and slower in the long run 


- **Generate Occurrences Cron** - two things: 
  - Currently this cron is running every 10 seconds - does it need to ? If the minimum frequency is DAILY, then any reminders created now at a minimum wont be needed to be shown until the next day, so if we ran the cron once daily it should still make the occurrences in time to be shown correctly. If we can figure out how to get it to run just once daily, then we save 8000+ DB calls a day, 2920000+ DB calls a year and so on
  - 
  - One potential improvement would be to have a separate cron/scheduled task that looks for specific frequencies e.g a daily cron, a weekly cron, and a monthly cron etc - the query in `getInstantForNextReminderOccurrences` would be smaller now as we would be searching for specific frequencies and we could also add an index on the frequency to search by 

    
    
## Observability Improvements  
If this were to be a production application there is not much insight into how its performing. There is a couple of 
areas that could be improved: 
- **metrics** : there is no metrics or tracing in the application. A tool like datadog might give greater insight into the latency of our endpoints, the breakdown of a requests time taken, it could help identify slow/inefficient db queries

- **error tracking** - there is no error tracking in this application. Using a tool like Sentry would greatly improve our ability to monitor new errors and debug them as well as give a better insight into how our application is performing and idnetify trends e.g at what commit did a specific error start occurring 


## Testing 
There is no system tests testing all integrations - should have block box tests where we just test the exposed parts of the application e.g
- `GET /reminder?employeeId=XXXXX`    : get reminder via API - should exist
- `GET /occurrences?employeeId=XXXXX` : get create a reminder via API - none should exist


- `POST /reminder?employeeId=XXXXX`   : create reminder via API
- `GET /reminder?employeeId=XXXXX`    : get reminder via API - should exist
- `GET /occurrences?employeeId=XXXXX` : get occurrence - should exist

- `GET /occurrences?employeeId=XXXXX` : acknowledge occurrence
- `GET /occurrences?employeeId=XXXXX` : get occurrence - shouldnt exist 


## Developer Experience 
Some quality of life improvements that would make developing easier for teammates : 
- Run the API from inside a docker container - the DB is running inside a docker container but not the API, this actually led to me having a host of errors when trying to run the service for the first time due to mismatched java versions between the project and my machine 
- linting rules so that code formatting is consistent 
