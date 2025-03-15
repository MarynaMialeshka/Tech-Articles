# Performance optimization an API method returning large data volume

### Backstory

My team got a complaint from a customer that the API method used to get the list of all users filtered by custom roles was failing with timeout exception. The method in question was OData-based, so it was decided to work on its SQL-based counterpart so the customer could use it instead. The reasons why the OData method couldn't be improved in terms of performance are beyond this article. Anyway, now my task was to extend the SQL-based `GET /users` method so it lives up to OData-based functionality. After doing so, we noticed its performance degraded. 

## Situation

After extending the API method with additional parameters and resposnse fields, a significant performance degradation was reported. Performance tests were run on the method without any input parameters, including the ones added in scope of the task. However, the response included additional data added in the course of implementation.

## Task

Find out the reasons for degraded performance and fix them.

## Actions

- As the main changes in scope of the task were made in a stored procedure, it was decided to start looking from there. I copied the text of the SP, assigned default values to its parameters (which, as it turned out later, was a mistake) beecause the API method was called without any parameters. Indeed, the execution took quite a long time, and the actual execution plan looked heavy. Yet, in my view, there wasn't much we could do to it. The client the SP was being tested on had about 500K users, so long execution shoudn't have surprised anyone.

2. This is when realization of my mistake dawned on me - the fact that the API method was called without parameters didn't mean it returned all client's users. I looked at the controller method's signature and discovered the UserTypeFilter parameter had a default value different from all users. I tried to run the script with the SP's text with the correct user type filter, and it finished in just about 8 seconds. While the API repsonse time was about 30 seconds. So where did this time come from?

3. Apparently, the bottleneck was somewhere in the source code. I put `System.Diagnostics.Debug.WriteLine` line with the info message and the timestamp after each step of user retrieving and processing, and it turned out the mapping of users and their properties from DB to service objects was taking the lion's share of execution time.

### What exactly slowed down the performance

A repository returned a list of users and a few lists of additional access objects. Users could be mapped with the access objects by user ids to build nested service models for further processing. Mapping that took aboout 20 seconds looked like:

    public IEnumerable<User> MapUsers(IEnumerable<dbUser> databaseUsers, IEnumerable<Access> accesses, IEnumerable<AdditionalInfo> additionalInfos)
    {
        var users = new List<User>();

        foreach(var dbUser in databaseUsers)
        {
            var user = Mapper.Map<User>(dbUser);

            //mapping user accesses by searching through the access list
            //mapping of Access type to some service object omitted for brevity
            user.Accesses = accesses.FirstOrDefault(a => a.UserId == dbUser.Id) ?? [];

            //same for additional info
            user.Info = infos.FirstOrDefault(i => i.UserId == dbUser.Id) ?? [];
        }

        return users;
    }

The problem was that for each user the search through the collection was performed, which in case of huge data volume (ours! about 16K users) would result in noticeable slowing down. 

4. Solution: rewrite access and info mapping using dictionaries.

        public IEnumerable<User> MapUsers(IEnumerable<dbUser> databaseUsers, IEnumerable<Access> accesses, IEnumerable<AdditionalInfo> additionalInfos)
        {
            var users = new List<User>();

            var accessesGrouppedByUser = accesses.GroupBy(a => a.UserId).ToDictionary(g => g.Key, g => g.ToList());

            var additionalInfosGrouppedByUser = additionalInfos.GroupBy(i => i.UserId).ToDictionary(g => g.Key, g => g.ToList());

            foreach(var dbUser in databaseUsers)
            {
                var user = Mapper.Map<User>(dbUser);

                //mapping user accesses by getting value from the dictionary
                user.Accesses = accessesGrouppedByUser.TryGetValue(user.Id, out var userAccesses) ? userAccesse : [];

                //same for additional info
                user.Info = additionalInfosGrouppedByUser.TryGetValue(user.Id, out var userInfo) ? userInfo : [];
            }

            return users;
        }

With the dictionary lookup, this code executed almost instantly even on big data volume.

*Note for my future self: evaluate executiion time before/after improvement in O notation and run benchmark test*

## Result

Despite slowing down the SP as a result of returning extended user data, the overall performance of an API method improved by ~ 10% on large data volumes. 

## Errors to avoid in the future

1. Always check the parameters the API method is invoked with event if they seem obvious and self-explanatory.
2. Source code might be the reason for degraded performance. Source code as the reason for performance issues should be cosidered as seriously as DB load or external APIs calls.