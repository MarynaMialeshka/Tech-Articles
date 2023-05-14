# Example of violating Single Responsibility Principle

In this article, I'm going to share how I dropped a test environment because this basic SOLID principle was neglected on my project. The first one of these five is paid the least attention, yet it is basically the fundament for good practices and developing easily-maintained and scalable applications.

---

Disclaimer:  What  I'm going to focus on is the importance of following SRP and SRP only. I'm aware that the architecture I was working on wasn't the best solution. However, this is the reality of enterprise.

---

## Problem description

Imagine a LocalizationService responsible for a few things:

- Retrieving a culture code from a database for a user based on the organization it belongs to
- Getting other region settings from an external service

The implementation might look like this:

    public class LocalizationService: ILocalizationService 
    {
        private readonly ILocalizationRepository _localizationRepository;

        private readonly IRegionSettingService _regionSettingService;

        //constructor

        public string GetCultureCode (User user) 
        {
            return _localizationRepository.Get(user.Id);
        }

        public RegionSettings GetRegionSetting (User user)
        {
            //some code
            return _regionSettingService.GetRegionSetting(user);
        }
    }

Now, let's look into a IRegionSettingService implementation. Apparently, the logic of calling external services must be incapsulated into a separate service consumed by RegionSettingService.

    public class RegionSettingService: IRegionSettingService
    {
        private readonly IHttpRequestService _requestService;

        //constructor

        public RegionSetting GetRegionSettings (User user)
        {
            //service url is likely to be formed from a configuration value and endpoint
            var url = "someUrl";

            return _requestService.Get(url, user.Id, currentSession);
        }
    }

Finally, here's our IHttpRequestService:

    public class HttpRequestService: IHttpRequestService
    {
        protected Dictionary<string, string> CustomHeaders { get; } = new Dictionary<string, string>();

        public HttpRequestService (ILocalizationService localizationService){
            CustomHeaders.Add({"Accept-Language", localizationService.GetCultureCode(CurrentUser)});
        }

        //some methods
    }

Here, we build an http header collection for requests. Particularly, we're passing the language code of the organization the user is an employee of with each request.

## What broke the environment?

The most attentive readers might already have noticed that if I deployed this code the environment would crash. This is what happened, and the reason is circular dependencies. LocalizationService depends on RegionSettingService which depends on HttpRequestService which depends on LocalizationService. The DI container verification fails at runtime.

## Solution

To break this cross-dependency, I decided to split LocalizationService into two separate classes - LocalizationService and RegionSettingProvider. Here is where the SRP comes to the stage. Not only can it be applied from the functionality POV, but it is also about reasons to update the class and what it depends on. In this case, the LocalizationService class incapsulated two responsibilities at once and has two reasons to be updated. Once it was split, SRP in followed and the dependency circle is broken.

## Summary

SRP isn't entirely about functional responsibilities. Consider this principle from different POVs to maintain the high code quality.